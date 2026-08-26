## Day 13 — HPA + Load Test (`k8s/hpa.yaml`, `load_test/loadgen.py`)

### Why HPA?

Single-replica and fixed 3-replica deployments can't adapt to traffic changes.
Under load, fixed replicas either:
- Over-provision: 6 replicas at idle wastes ~3GB RAM continuously
- Under-provision: 2 replicas under heavy load causes latency spikes and errors

HPA (HorizontalPodAutoscaler) automatically adjusts replica count based on
observed metrics — adding capacity when needed, releasing it when not. This
is the Kubernetes implementation of horizontal scaling: more pods, not bigger pods.

### Why CPU utilization as the scaling metric?

search-api is CPU-bound under load — sentence-transformers runs neural network
inference on the CPU for every search query. Under load, CPU utilization is a
direct proxy for request pressure.

Alternative metrics (not implemented here):
- Request rate via Prometheus custom metrics (KEDA or prometheus-adapter required)
- Memory utilization — not appropriate for search-api because memory is dominated
  by the model weights (~500MB constant), not per-request usage
- Queue depth — not applicable (stateless HTTP service, no queue)

CPU is the right signal for this workload and requires no additional components
(metrics-server is sufficient).

### Why 50% target utilization?

```
CPU request:   250m per pod
Target:        50% of request = 125m per pod
Scale-up at:   actual CPU > 125m per pod (averaged across all pods)
```

50% leaves headroom for traffic spikes between HPA control loop cycles (15s).
If the target were 90%, a sudden traffic spike could saturate all pods before
the next scale-up fires. At 50%, there's a 2x buffer — CPU can double before
the next scaling decision without causing failures.

HPA scaling calculation:
```
desiredReplicas = ceil(currentReplicas × (currentCPU / targetCPU))
```

From the load test: `ceil(2 × (174 / 50)) = ceil(6.96) = 7`, capped at maxReplicas=6.

### Why minReplicas: 2, not 3?

After Day 9 (scaling to 3 replicas for HA), HPA takes over replica management.
HPA will scale down to minReplicas when traffic is low. Setting minReplicas: 2
instead of 3 means HPA can reclaim one pod's worth of resources (~500MB RAM)
during sustained low-traffic periods, while still maintaining redundancy.

minReplicas: 1 would mean a single pod handles all traffic between scale-up
events — the same single-replica risk from Postmortem #1. minReplicas: 2 is
the minimum for meaningful redundancy.

### Why maxReplicas: 6?

6 × 250m CPU request = 1500m = 1.5 CPU cores from the scheduler's perspective.
The Minikube node has 4 CPUs allocated. Staying at 6 max leaves headroom for:
- K8s system pods (etcd, apiserver, scheduler, coredns): ~500m
- Prometheus + Grafana: ~300m
- Buffer for burst: ~700m

In production, maxReplicas would be set based on cost budget and the service's
maximum expected load, not node capacity.

### Why scaleUp window 0s, scaleDown window 60s?

**scaleUp stabilizationWindowSeconds: 0**
Scale up immediately when CPU exceeds threshold. Traffic spikes are real —
users are waiting for responses right now. Delaying scale-up to wait for
"stable" high CPU means users experience latency or errors during the delay.
Fast scale-up is almost always correct for web services.

**scaleDown stabilizationWindowSeconds: 60**
Wait 60 seconds of sustained low CPU before scaling down. Prevents thrashing:
a brief traffic lull followed by another spike would cause immediate scale-down
then immediate scale-up. The 60-second window ensures the load is actually gone
before releasing capacity. The cost of keeping extra pods for 60 seconds is
negligible compared to the cost of a scale-down/scale-up cycle.

**scaleDown policy: 1 pod per 30s**
Gradual scale-down rather than immediate drop to minReplicas. Removes one pod
at a time with 30-second gaps, giving the service time to rebalance traffic
after each removal. Prevents a sudden capacity cliff if traffic resumes
during the scale-down period.

### Load Test Design — Why 10 Workers × 10 req/s?

```python
NUM_WORKERS = 10    # concurrent async workers
# Each worker sends 1 request every 0.1s = 10 req/s per worker
# Total: 10 workers × 10 req/s = ~100 req/s total
```

100 req/s against 2 pods = 50 req/s per pod. Each search query takes ~300–500ms
of CPU time. 50 req/s × 400ms CPU per request = 2000ms CPU per second = 200% CPU
per pod — well above the 50% × 250m = 125m threshold. Sufficient to trigger scaling.

Why async (asyncio + httpx)?
Synchronous requests would bottleneck on the 300–500ms latency per search.
A synchronous single-threaded loop could only send ~2–3 req/s. Async allows
10 concurrent in-flight requests per worker, achieving the target 100 req/s
total despite the high per-request latency.

### Load Test Results

| Time | CPU | Replicas | Event |
|------|-----|----------|-------|
| 7h6m | 1% | 2 | Idle baseline |
| 7h6m | 174% | 2 | Load test starts |
| 7h7m | 174% | 4 | First scale-up (+2 pods) |
| 7h7m | 200% | 6 | Second scale-up (maxReplicas) |
| 7h8m | 15% | 6 | Load test ends |
| 7h9m | 1% | 5 | Scale-down begins (-1 pod) |
| 7h10m | 1% | 4 | Continuing scale-down (-1 pod) |

**Why CPU hit 200% at 6 replicas?**
The two new pods (5 and 6) were still loading the model during scale-up —
they passed the readiness probe but the model was still initializing in memory.
For ~30 seconds, only 4 pods were actually handling search traffic while all
6 were counted in the average. Once all 6 were fully ready, CPU dropped to 15%.

**Error budget impact of the load test:**
During the 200% CPU peak, some requests may have experienced higher latency
(above the 300ms SLO). The load test was intentional — errors during deliberate
capacity testing don't consume production error budget by convention. Documented
in Postmortem #2 (Day 14).
