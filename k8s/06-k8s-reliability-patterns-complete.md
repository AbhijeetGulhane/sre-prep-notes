# K8s Reliability Patterns — Complete Study Notes
## Probes, Pod Disruption Budgets, HPA, Requests vs Limits

**Date:** Week 4, Day 6

---

## PART A: Probes — Liveness, Readiness, Startup

## 1. Why Three Separate Probes

A single health check can't express enough nuance. Three distinct questions need three distinct answers:
- "Is this container still alive?" → **Liveness**
- "Is it ready to serve traffic right now?" → **Readiness**
- "Is it still initializing?" → **Startup**

Conflating them causes either unnecessary restarts or unnecessary traffic-routing failures. All three probe types share the same four mechanisms and the same configuration parameters.

## 2. Probe Mechanisms — Four Types

**`exec`** — run a command inside the container. Exit 0 = success, anything else = failure.
```yaml
exec:
  command: ["pg_isready", "-U", "postgres"]
```

**`httpGet`** — HTTP GET to a path/port. HTTP 200–399 = success.
```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: Custom-Header
    value: Awesome
```

**`tcpSocket`** — TCP connection attempt. Connection established = success. For non-HTTP protocols (Redis, Postgres).
```yaml
tcpSocket:
  port: 6379
```

**`grpc`** — gRPC health check protocol. Standard `grpc.health.v1.Health` service. Available since K8s 1.24.
```yaml
grpc:
  port: 9090
  service: "myservice"
```

## 3. Probe Configuration Parameters

Every probe type shares these tuning knobs:

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `initialDelaySeconds` | 0 | Wait after container starts before first check |
| `periodSeconds` | 10 | How often to run the probe |
| `timeoutSeconds` | 1 | Probe response deadline before counting as failure |
| `failureThreshold` | 3 | Consecutive failures before acting |
| `successThreshold` | 1 | Consecutive successes to recover (liveness/startup must be 1) |

## 4. Startup Probe

**Purpose:** Give slow-starting containers time to initialize without interfering with liveness/readiness.

**Key behavior:** While startup probe is still failing, liveness and readiness probes are **completely suppressed** — kubelet doesn't run them at all. Startup probe gates the other two.

**On failure past `failureThreshold`:** kubelet kills and restarts the container (same as liveness failure).

**Why it exists:** Before startup probes (K8s 1.18), slow-starting containers had no good option. Large `initialDelaySeconds` on liveness = broken app stays up too long. Small `initialDelaySeconds` = liveness kills the app mid-initialization. Startup probe solves this cleanly.

**Pattern for slow-starting JVM or database:**
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30      # 30 × 10s = 5 minutes to start
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 3       # tighter once startup confirmed
  periodSeconds: 10
```

## 5. Liveness Probe

**Purpose:** Detect irrecoverably broken containers — deadlocked, stuck, not making progress despite being "Running."

**On failure past `failureThreshold`:** kubelet sends SIGTERM to container's PID 1 → waits `terminationGracePeriodSeconds` → SIGKILL if still running → creates a new container (same Pod, increments `restartCount`) → exponential backoff (10s, 20s, 40s... capped at 300s) = `CrashLoopBackOff`.

**Use case:** Web server that has exhausted its thread pool — accepting connections but never responding. No crash, no exit, but no useful work. Liveness timeout catches it.

**Critical antipatterns:**

1. **Probing an external dependency** — if liveness checks the database and the database goes down, every pod fails liveness and restarts in a stampede. Restarting the app doesn't fix the database. Liveness should only test the application itself.

2. **Too aggressive timing** — `failureThreshold: 1` + `timeoutSeconds: 1` means one slow probe (e.g. GC pause) kills the container. Use `failureThreshold: 3`, `periodSeconds: 10`, `timeoutSeconds: 5`.

3. **Same endpoint as readiness** — defeats the purpose. An overloaded server should fail readiness (drain traffic) but pass liveness (don't restart — overload may resolve on its own).

## 6. Readiness Probe

**Purpose:** Control whether traffic flows to this pod right now. Temporary unavailability, not permanent brokenness.

**On failure:** kubelet sets Pod's `Ready` condition to `False` → Endpoints controller removes Pod IP from Endpoints/EndpointSlice → kube-proxy removes from iptables/IPVS rules → traffic stops. **Container is NOT restarted.**

**On recovery:** `Ready` returns to `True` → IP re-added → traffic resumes.

**Use cases:**
- Slow startup (app running but caches not loaded, connections not established)
- Dependency unavailable (downstream service down — stop taking traffic, not restart)
- Graceful shutdown drain (fail readiness manually, let in-flight requests finish)
- Circuit breaker open (protecting from cascade failure)

## 7. What Kubelet Actually Does — Mechanically

**Liveness/startup failure:**
1. Sends SIGTERM to container's PID 1
2. Waits `terminationGracePeriodSeconds` (default 30s)
3. SIGKILL if still running
4. Creates replacement container in same Pod
5. Increments `containerStatuses[].restartCount`
6. Applies exponential backoff before restart

**Readiness failure:**
1. Sets Pod `Ready` condition to `False` via apiserver
2. Endpoints controller detects condition change
3. Removes Pod IP from Endpoints/EndpointSlice
4. kube-proxy on every node picks up change → removes IP from iptables/IPVS backend pool
5. No restart — container keeps running
6. In-flight connections may continue draining

## 8. Probe Decision Matrix

```
Still initializing?              → Startup probe controls this window
Irrecoverably broken → restart?  → Liveness probe → kills + restarts container
Temporarily unable to serve?     → Readiness probe → drains traffic, no restart
```

---

## PART B: Pod Disruption Budgets (PDB)

## 9. The Problem PDB Solves

Multiple Kubernetes mechanisms can remove pods voluntarily:
- Node drain (`kubectl drain`) for maintenance
- Cluster autoscaler scale-down
- Rolling updates evicting old pods

Without PDB: all 3 replicas could be evicted simultaneously → service completely offline during drain. PDB gives application owners a way to say "never take us below X available replicas at a time."

## 10. What PDB Is

A policy object constraining **voluntary disruptions** — anything going through the Eviction API. Does **NOT** protect against node failures, OOM kills, or hardware issues (involuntary disruptions).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  selector:
    matchLabels:
      app: api
  minAvailable: 2        # always keep at least 2 pods available
  # OR:
  # maxUnavailable: 1    # allow at most 1 pod unavailable at a time
```

Specify either `minAvailable` or `maxUnavailable`, not both. Both accept integers or percentages.

## 11. Enforcement Mechanism

When any component tries to evict a pod, it calls the **Eviction API** (`POST /api/v1/namespaces/{namespace}/pods/{pod}/eviction`) — not a direct Delete. The apiserver checks whether eviction would violate any covering PDB:

- Budget satisfied → eviction proceeds, pod deleted, replacement scheduled
- Budget violated → apiserver returns `429 Too Many Requests` → requester must retry

This is why `kubectl drain` blocks — it retries on 429, waiting for pods to be rescheduled and become Ready before attempting the next eviction.

## 12. PDB and Rolling Updates

Two separate mechanisms at different layers:
- **Deployment's `maxUnavailable`:** how many pods the Deployment controller allows unavailable during rollout (controls its own pace)
- **PDB `minAvailable`:** hard constraint the Eviction API enforces

**Common misconfiguration:** PDB says `minAvailable: 3` and Deployment has 3 replicas → rolling update completely blocked — can't take any pod offline without violating PDB. Correct sizing: `minAvailable: n-1` or `maxUnavailable: 1` for n replicas.

## 13. PDB and Node Drains

`kubectl drain <node> --ignore-daemonsets` process:
1. Marks node unschedulable (cordon)
2. For each pod: calls Eviction API
3. If blocked by PDB → waits and retries
4. If eviction succeeds → pod deleted → Deployment controller creates replacement → replacement becomes Ready → next pod can be evicted

**`--force` flag:** bypasses Eviction API entirely, directly deletes pods — PDB completely ignored. Never use on production.

## 14. PDB Gotchas

- **Already-disrupted pods:** if pods are already unavailable and PDB is already violated, further evictions are still blocked
- **No PDB = no eviction protection:** cluster autoscaler and drains evict freely
- **Stuck drains:** pods in `CrashLoopBackOff` can block drains indefinitely if PDB counts them as "available"

---

## PART C: HorizontalPodAutoscaler (HPA)

## 15. What HPA Does

Automatically adjusts `replicas` count of a Deployment (or ReplicaSet, StatefulSet) based on observed metrics, within configured min/max bounds. A control loop — observe, compare, act.

## 16. The Control Loop

Runs on a configurable interval (default 15s, `--horizontal-pod-autoscaler-sync-period`):

```
Every 15 seconds:
  1. Fetch current metric values
  2. Calculate: desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
  3. Apply min/max bounds
  4. If desiredReplicas != currentReplicas: update Deployment spec.replicas
```

## 17. Metric Sources

**Resource metrics** (built-in, served by metrics-server):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70      # % of requests across all pods
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
```

**Custom metrics** (via Prometheus Adapter):
```yaml
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

**External metrics** (queue depth, external service):
```yaml
- type: External
  external:
    metric:
      name: queue_messages_ready
    target:
      type: AverageValue
      averageValue: "30"
```

## 18. The Scaling Calculation

CPU target 70%, 3 replicas at 150% average:
```
desiredReplicas = ceil(3 × (150 / 70)) = ceil(6.43) = 7
```

**Multiple metrics:** HPA calculates desired replicas for each metric independently and takes the **maximum**. Ensures all metrics satisfied simultaneously.

## 19. Stabilization Windows — Preventing Thrashing

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0    # scale up immediately (default)
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15              # can double replicas every 15s
  scaleDown:
    stabilizationWindowSeconds: 300  # wait 5 min before scaling down (default)
    policies:
    - type: Pods
      value: 4
      periodSeconds: 60              # remove at most 4 pods per minute
```

- **Scale-up aggressive** (window = 0): traffic spikes need fast response
- **Scale-down conservative** (300s window): prevents scaling down during temporary dip then immediately needing to scale back up

## 20. What HPA Uses for CPU

HPA measures CPU against **requests**, not limits:
```
CPU utilization % = (actual CPU usage) / (CPU request) × 100
```

Pod with no CPU request set → HPA cannot compute utilization → pod excluded from calculation. **This is why accurate resource requests are critical for HPA to work.**

## 21. HPA Limitations and Gotchas

- **Scale to zero not supported** — `minReplicas` cannot be 0 (nothing to measure). Use KEDA for scale-to-zero.
- **metrics-server must be running** — HPA does nothing without it. Not installed by default everywhere.
- **HPA overrides manual scaling** — `kubectl scale deployment api --replicas=10` gets overridden at next HPA sync.
- **VPA conflict** — don't run HPA on CPU/memory AND VPA on same workload simultaneously.

---

## PART D: Resource Requests vs Limits — The Full Picture

## 22. The Two Numbers

**Request** — what the container is *guaranteed* to get. Used by:
- Scheduler for node placement (only looks at requests, never limits)
- HPA for utilization calculations
- cgroup weight/share for relative CPU priority
- QoS class assignment (affects OOM kill order)

**Limit** — the maximum the container is *allowed to use*. Enforced by:
- CPU limit → cgroup `cpu.max` → CFS bandwidth throttling (pulled off run queue when quota exhausted, not killed)
- Memory limit → cgroup `memory.max` → OOM kill if exceeded

```yaml
resources:
  requests:
    cpu: "250m"      # 250 millicores = 0.25 CPU
    memory: "256Mi"
  limits:
    cpu: "1000m"     # 1 CPU
    memory: "512Mi"
```

## 23. QoS Classes

Determined entirely by the relationship between requests and limits:

| Class | Condition | OOM Priority | Eviction Priority |
|-------|-----------|--------------|-------------------|
| **Guaranteed** | Every container: requests == limits for both CPU and memory | Last killed | Last evicted |
| **Burstable** | At least one container has a request or limit, but not Guaranteed | Middle | Middle |
| **BestEffort** | No container has any requests or limits | First killed | First evicted |

Guaranteed gets `oom_score_adj` of ~-997. BestEffort gets 1000 (killed first under pressure).

## 24. The Scheduler Uses Only Requests

Scheduler looks at `requests` to determine if a node has enough **allocatable capacity**:
```
Node allocatable = total capacity - system-reserved - kubelet-reserved
```

A node can be at 100% actual CPU utilization but still have schedulable headroom if pods are bursting above requests. Conversely, a node can have free actual CPU but be "full" from the scheduler's perspective if all allocatable request capacity is consumed by idle pods.

## 25. CPU: What Actually Happens

| Scenario | Behavior |
|----------|---------|
| Under limit | Container uses as much CPU as it needs, up to limit |
| At limit | CFS throttler pulls tasks off run queue until next period. Request stalls. No kill. |
| No limit set | Container can burst to use all available node CPU (noisy neighbor risk) |

**CFS throttling signature:** `container_cpu_cfs_throttled_seconds_total` rising while average CPU% looks fine. Container hitting quota within periods even though average utilization is low.

## 26. Memory: What Actually Happens

| Scenario | Behavior |
|----------|---------|
| Under limit | Container uses memory freely |
| At limit | Kernel attempts reclaim. If that fails → OOM killer fires within cgroup → SIGKILL → exit 137 |
| No limit set | Container grows until node runs out → global OOM killer fires → may kill processes from other pods |

**Always set memory limits.** No limit = noisy-neighbor OOM risk on the entire node.

## 27. Why Accurate Requests Are Critical

- **Too low:** over-scheduling — scheduler places more pods than node can sustain. OOM kills, CPU throttling, pod interference.
- **Too high:** under-utilization — scheduler sees nodes as "full" when they're actually idle. Wasted cost, pods stuck Pending.
- **Zero requests:** HPA can't compute utilization (division by zero). Scheduler can't make informed placement decisions.

**LimitRange admission controller:** sets default requests/limits at namespace level — if a pod spec doesn't set them, LimitRange fills them in automatically.

## 28. Memory Limit Sizing Heuristic

- Set **memory limit** = observed peak usage + 20–30% headroom
- Set **memory request** = observed steady-state usage
- Watch `container_memory_working_set_bytes` — if consistently within 10% of limit, OOM kills are imminent

## 29. The Key Asymmetry — Exam-Critical

| Resource | Exceeding limit | Mechanism | Recoverable? |
|----------|----------------|-----------|-------------|
| CPU | Throttled | CFS bandwidth control (`cpu.max`) | Yes — resumes next period |
| Memory | OOMKilled | cgroup OOM killer (`memory.max`) | No — process killed |

CPU is a **rate** (time passes, quota refills). Memory is a **stock** (you hold it continuously — no way to pause usage).

---

## PART E: Tying It Together — Production Scenario

A web API Deployment with 3 replicas:
```yaml
resources:
  requests: {cpu: 250m, memory: 256Mi}
  limits:   {cpu: 1000m, memory: 512Mi}
livenessProbe:  {httpGet: /healthz, failureThreshold: 3, periodSeconds: 10}
readinessProbe: {httpGet: /ready, failureThreshold: 2, periodSeconds: 5}
```
With HPA targeting 70% CPU and PDB `minAvailable: 2`.

**Traffic spike:** CPU climbs → HPA scales to 7 → new pods fail readiness briefly → excluded from Endpoints → pass readiness → traffic distributes.

**One pod deadlocks:** Liveness fails 3× → killed and restarted → during restart, readiness fails → removed from Endpoints → other pods handle traffic → pod restarts and becomes ready.

**Node drain:** PDB `minAvailable: 2` → drain evicts one pod at a time → Deployment creates replacement → replacement becomes Ready → drain continues → service never below 2 ready pods.

**Memory spike:** Pod hits 512Mi `memory.max` → cgroup OOM killer fires → SIGKILL → exit 137 → `kubectl describe pod` shows `OOMKilled` → kubelet restarts → `CrashLoopBackOff` if it keeps happening → fix: raise limit or cap request size.

---

## Retention Questions and Answers

**Q1: A pod has been running for 2 hours and suddenly starts getting OOMKilled every 10 minutes. What's your diagnostic sequence?**

First, confirm it's actually an OOM kill: `kubectl describe pod <name>` — look for `OOMKilled` in `lastState.terminated.reason` and `exitCode: 137`. This confirms the cgroup-scoped OOM killer fired, not a crash.

Second, get a memory usage graph, not a snapshot. RSS at restart time always looks modest — you need to see the spike that triggered the kill. Check `container_memory_working_set_bytes` over the past hour.

Third, check `memory.events` on the container's cgroup — the `oom_kill` counter confirms when kills happened and how frequently.

Fourth, look for what changed 2 hours ago — was there a traffic increase, a batch job that started, a new deployment, a larger payload type? "Suddenly starts" usually means something changed in the workload, not the infrastructure.

Fifth, compare `memory.current` (or Prometheus metrics) against `memory.max` (the limit). If working_set is consistently at 80%+ of the limit, the limit is too tight for the actual workload.

Fix options: raise the memory limit, add application-level request size cap, reduce concurrent request count, or investigate memory leak if usage grows unboundedly.

---

**Q2: Explain why setting your liveness probe to check an external database is an antipattern — what specifically goes wrong?**

When the database goes down, every pod in the fleet simultaneously fails its liveness probe. After `failureThreshold` consecutive failures, kubelet kills and restarts all pods. The restarted pods immediately try to reconnect to the still-down database, fail liveness again, and get killed again. You now have a restart storm on top of a database outage.

The restart doesn't fix anything — your app can't cure a database failure by restarting itself. What you've done is converted a database outage (which your app could potentially handle gracefully with retries and queuing) into a complete service outage with continuous pod churn, making the situation worse.

Liveness probes should only test the application process itself — is it deadlocked, has it exhausted resources, is it capable of making progress. External dependencies belong in readiness probes: "I can't serve traffic right now because my database is unavailable" → fail readiness, remove from load balancer, wait for recovery, re-add when ready.

---

**Q3: A `kubectl drain node-3` command has been hanging for 20 minutes. What are the likely causes and how do you diagnose each?**

There are three likely causes, each with a distinct diagnosis:

**PDB violation (most common):** A PDB is blocking eviction because evicting the next pod would drop available replicas below `minAvailable`. Check: `kubectl get pdb --all-namespaces` — look for any PDB where `ALLOWED DISRUPTIONS` is 0. The drain is waiting for the replacement pod to become Ready before it can evict the next one. If the replacement pod is stuck Pending or in CrashLoopBackOff, the drain will wait forever. Fix: find why the replacement isn't becoming Ready.

**Pods not managed by a controller:** Standalone pods (not owned by a Deployment, ReplicaSet, etc.) can't be rescheduled — drain refuses to evict them by default. Check: `kubectl get pods --all-namespaces --field-selector spec.nodeName=node-3` — look for pods with no owner reference. Fix: `--force` to delete them (data loss risk) or move them manually first.

**DaemonSet pods:** By default drain skips DaemonSet pods — if you forgot `--ignore-daemonsets`, drain blocks on them. Fix: add `--ignore-daemonsets` flag.

**Stuck eviction with PDB already violated:** If pods are already unhealthy and the PDB is already violated (e.g. a pod is in CrashLoopBackOff), drain will never succeed because evicting any pod would violate the PDB further. Check `kubectl describe pdb` for current disruption count vs allowed.

---

**Q4: HPA is configured for CPU utilization target of 70%, but it never scales up even when pods are visibly overloaded. What are the possible causes?**

**No CPU request set on the pods (most common):** HPA computes utilization as `actual usage / request`. If request is not set, the calculation is undefined and those pods are excluded from the HPA calculation entirely. Check: `kubectl describe hpa` — if it shows `<unknown>` for current metrics, this is the cause.

**metrics-server not running:** HPA depends on the metrics-server to get CPU data from the `metrics.k8s.io` API. If metrics-server is down or not installed, HPA has no data to act on. Check: `kubectl top pods` — if this fails, metrics-server is the issue.

**Already at maxReplicas:** HPA won't scale beyond its configured maximum. Check: `kubectl get hpa` — if `REPLICAS` equals `MAXREPLICAS`, the ceiling is hit. Raise `maxReplicas`.

**Stabilization window:** HPA has a default 300-second scale-down stabilization window. For scale-up the default is 0, but if someone customized `behavior.scaleUp.stabilizationWindowSeconds` to a high value, scale-up can be artificially delayed.

**Target reference wrong:** If `scaleTargetRef` points at the wrong Deployment name or kind, HPA silently does nothing. Check `kubectl describe hpa` for any reference errors.

---

**Q5: A Deployment has 5 replicas, all Running. A rolling update starts and immediately gets stuck — no pods are being replaced. What's your first hypothesis?**

A PDB with `minAvailable` set too tight for the replica count. If PDB says `minAvailable: 5` and the Deployment has exactly 5 replicas, taking any pod offline for replacement would drop available replicas to 4 — violating the PDB. The Deployment controller tries to evict via the Eviction API, gets 429, and waits. No pods are replaced.

Confirm: `kubectl get pdb --all-namespaces` — check `ALLOWED DISRUPTIONS`. If it shows 0, that's the cause. Also check `kubectl describe pdb <name>` for the current disrupted/expected counts.

Fix: either increase replicas (e.g. to 6, so minAvailable: 5 allows 1 unavailable) or lower `minAvailable` to `n-1` which is the correct setting for n replicas.

Secondary hypothesis: `maxUnavailable: 0` in the Deployment's rolling update strategy combined with `maxSurge: 0` — this is an impossible combination that prevents any update from proceeding. But PDB is the more common real-world cause.

---

**Q6: Explain the exact kernel mechanism that fires when a container exceeds its memory limit, and why it gets SIGKILL rather than SIGTERM.**

The container's memory limit is enforced as `memory.max` on its cgroup (cgroup v2). When a process inside the cgroup tries to allocate memory that would push the cgroup's usage over `memory.max`, the kernel first attempts **direct reclaim** within that cgroup — trying to free memory by evicting page cache and reclaimable pages. If reclaim fails to free enough memory to satisfy the allocation, the **cgroup-scoped OOM killer** fires.

The OOM killer selects a victim process within the cgroup (the one with the highest `oom_score`, computed from memory usage relative to available memory and adjusted by `oom_score_adj` — which kubelet sets based on QoS class). It sends that process **SIGKILL**, not SIGTERM.

SIGKILL rather than SIGTERM because: the OOM killer fires because memory is critically exhausted right now. Sending SIGTERM would give the process time to gracefully shut down — but graceful shutdown typically allocates more memory (flushing buffers, logging, closing connections). There's no memory to spare for graceful anything. SIGKILL immediately terminates the process, releasing its memory to the kernel without any further allocation. The immediacy is the point.

The container exits with code 137 (128 + 9, SIGKILL). Kubelet detects the exit, sees the cgroup's `memory.events` `oom_kill` counter incremented, and records `OOMKilled` as the termination reason in the pod's container status.

---

**Q7: A pod's CPU request is 200m and limit is 2000m. Node has 4 CPUs. What happens under three conditions?**

**Node is idle:** The pod can use up to 2000m (2 CPUs) freely — its limit is the only constraint. No other workloads competing, so the CFS scheduler gives it whatever it asks for up to the limit. `cpu.max` in the cgroup sets the absolute ceiling at 2000m per 100ms period.

**Node is saturated (all 4 CPUs heavily utilized):** The pod's CPU request of 200m becomes its guaranteed share — it gets at least 200m relative to other pods in proportion to their requests. This is the cgroup `cpu.weight` mechanism — CPU shares are distributed proportionally to requests when there's contention. The pod won't starve, but it won't get its full 2000m burst capacity — other pods with higher requests will get proportionally more. The scheduler initially placed this pod here based on 200m fitting in allocatable headroom; the actual contention is managed by the kernel.

**Pod using 1500m continuously:** If the node has spare capacity, the pod runs at 1500m freely (under its 2000m limit). If other pods are also competing, this pod's 1500m usage represents 7.5× its 200m request — it's using significantly more than its "fair share." During saturation, it may get throttled back toward its proportional share. But note: CFS throttling only kicks in if usage hits the hard `cpu.max` (2000m per 100ms). The proportional sharing at 1500m during contention is a softer effect — the pod still runs, but may not consistently get 1500m if others are competing hard.

---

**Q8: 3 replicas, one pod's readiness probe failing. What does the end user experience, and what changed in cluster state?**

**End user experience:** No visible impact. The 2 remaining healthy pods handle all traffic. From the user's perspective, the service is fully operational — requests are served normally, latency may be slightly higher if the 2 pods are now each handling ~50% more load than before. If they're within capacity, users see nothing.

**What changed in cluster state — step by step:**

1. kubelet on the pod's node detects consecutive readiness probe failures past `failureThreshold`
2. kubelet updates the Pod's `Ready` condition to `False` via kube-apiserver → written to etcd
3. The Endpoints controller (in kube-controller-manager) watches Pod conditions — detects the `Ready: False` change
4. Endpoints controller removes this Pod's IP from the Service's Endpoints/EndpointSlice object → updated in etcd
5. kube-proxy on every node in the cluster watches Endpoints — detects the removal
6. kube-proxy removes this Pod IP from the Service's iptables/IPVS backend pool on every node
7. New connections to the Service ClusterIP are now DNAT'd only to the 2 remaining healthy pod IPs

The failing pod itself is untouched — still running, still accumulating `restartCount: 0`, not in CrashLoopBackOff. Only traffic is affected, not the container lifecycle. When the readiness probe starts passing again, the entire chain reverses: `Ready: True` → Endpoints updated → kube-proxy rules updated → traffic resumes.

This is the correct behavior for: slow startup, temporary dependency unavailability, deliberate traffic draining during graceful shutdown.
