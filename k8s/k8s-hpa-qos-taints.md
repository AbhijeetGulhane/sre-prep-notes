# K8s HPA, QoS Classes, Taints and Tolerations — How the Scheduler Uses All Three

> K8s core objects deep dive, final item in the 7-topic batch. Includes live minikube lab: QoS class verification, taint/toleration scheduling, and a real container-level OOMKill.

## The three mechanisms, framed by the question each answers

- **HPA** — "how many replicas do I need right now?"
- **QoS classes** — "if this node runs out of resources, which Pods get killed first?"
- **Taints and tolerations** — "which Pods are even allowed to be scheduled onto this node?"

Different points in a Pod's lifecycle: taints/tolerations at **scheduling time**, HPA continuously adjusting **replica count**, QoS classes only at **eviction/OOM time**, after scheduling already happened. They compose, but aren't the same job.

## HPA — Horizontal Pod Autoscaler

**Mechanism:** HPA controller polls metrics (via metrics-server, or custom/external metrics adapters) on a fixed interval (default 15s), compares against target, adjusts `replicas`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: search-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: search-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Critical prerequisite:** requires `resources.requests` set on the container — utilization % is calculated relative to the request, not node capacity. No request = no denominator = HPA can't function for that metric.

**Scaling behavior:**
- Scale-up: fast/aggressive by default, no built-in delay
- Scale-down: stabilization window (default 5 min) to prevent flapping (`behavior.scaleDown.stabilizationWindowSeconds`)
- Desired replicas computed from metric ratio (e.g. 140% of 70% target → ~double), not fixed step — can jump multiple replicas at once for a spike

**What HPA does NOT do:** doesn't create nodes. No room for new replicas → they sit `Pending` until existing Pods free resources or **Cluster Autoscaler** (separate, node-level mechanism) adds capacity. HPA = replica count, Cluster Autoscaler = node count — independently configured, commonly confused as the same thing.

**One-liner:** "HPA scales replica count based on observed metrics relative to requests — not absolute values — and only handles the Pod layer; if there's no node capacity for the new replicas, that's the Cluster Autoscaler's job, a completely separate mechanism."

## QoS classes — derived automatically from requests/limits

No `qosClass` field to set directly — K8s **derives** it. Three classes:

**Guaranteed** — every container has BOTH requests AND limits set, EQUAL, for BOTH CPU and memory.
```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```

**Burstable** — at least one container has a request set, but requests/limits aren't equal (or limits missing on some resource). Most common real-world case.
```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }
```

**BestEffort** — no requests or limits on any container, any resource. Least protected — often an accident, not intentional.

**Gotcha proven in lab:** setting requests=limits for memory ONLY (no CPU set at all) does NOT yield Guaranteed — it falls back to **Burstable**, since equal requests/limits must cover every resource type across every container. Partial coverage silently downgrades the class.

**Why the classification exists:** drives kubelet's OOM killer priority via kernel `oom_score_adj` — under memory pressure, there's a deterministic order for who dies first.

## Eviction order under memory pressure

- **BestEffort dies first, always** — highest oom_score_adj, made zero commitment about needs, owed least protection.
- **Burstable dies second** — sub-ordered by how far over its `request` it's grown at the moment of pressure; a Pod using memory near its request is relatively protected, one that's bursted way past it is targeted first among Burstable Pods.
- **Guaranteed dies last, effectively never** (barring extreme circumstances) — requests=limits means kubelet already accounted for exactly this Pod's footprint when scheduling; shouldn't cause node-level pressure alone if the node wasn't overcommitted to begin with.

**Direct answer — BestEffort vs Guaranteed under memory pressure:** the BestEffort Pod gets OOMKilled, full stop. Not proportional, not "whoever's using more in that instant" — QoS class is the deciding factor, checked before raw usage is even the tiebreaker. Guaranteed is protected because it made an explicit, bounded promise the system honors at the expense of Pods that made none.

**One-liner:** "QoS class directly maps to OOM kill priority — BestEffort dies first because it made no resource commitment, Burstable dies next ordered by how far over its request it's grown, and Guaranteed is essentially protected because requests equal limits, so the kubelet already accounted for its exact footprint when scheduling."

## Taints and tolerations

**Taint** (on a node) — repels Pods unless explicitly tolerated.
```bash
kubectl taint nodes node1 dedicated=gpu-workloads:NoSchedule
```

**Toleration** (on a Pod) — allows but doesn't force scheduling onto a matching-tainted node.
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu-workloads"
  effect: "NoSchedule"
```

**Critical distinction:** toleration ≠ attraction. It only *permits* scheduling there if the scheduler independently picks that node. To *force* placement, need **node affinity** too — taint = "keep most things out," affinity = "pull this specific thing in." Typically used together: taint GPU nodes so nothing lands by accident, add affinity on GPU workloads so they specifically target those nodes (toleration alone doesn't guarantee that placement).

**Three effects:**
- `NoSchedule` — blocks new Pods without toleration; existing Pods unaffected
- `PreferNoSchedule` — soft, scheduler avoids but will use if nothing else fits
- `NoExecute` — strict: blocks new Pods AND **evicts already-running Pods** without toleration

**Automatic taints (connects to DaemonSet topic):** `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable`, `node.kubernetes.io/memory-pressure`, `node.kubernetes.io/disk-pressure`. Normal Pods evicted via `NoExecute` after grace period (`tolerationSeconds`, default 300s). **This is exactly why DaemonSets (log collectors, monitoring agents) need explicit tolerations for these taints** — they need to keep running on unhealthy/pressured nodes specifically to observe and report on that state.

## How all three compose

1. **Scheduling time:** taints/tolerations + affinity determine eligible nodes
2. **Ongoing, background:** HPA adjusts replica count independent of taints/QoS — scheduler (subject to taints/tolerations/affinity) places the new replicas
3. **Under pressure, post-scheduling:** QoS class determines eviction order, only matters for Pods that already passed step 1's filter

**Concrete scenario:** taint GPU nodes `NoSchedule`; ML training Job has matching toleration + node affinity (so it lands there, not just can); set requests=limits (Guaranteed) since an OOM-killed multi-hour training run is expensive to restart. Meanwhile HPA scales an unrelated web-tier Deployment independently — those Pods are Burstable (reasonable baseline + generous burst limit), since a web request dying under pressure is cheaply recoverable.

## Interview one-liners, compiled

- "HPA changes replica count based on metrics relative to requests; Cluster Autoscaler changes node count — commonly confused, but genuinely separate systems that happen to work together."
- "QoS class isn't set directly — it's derived from whether requests equal limits on every resource (Guaranteed), requests exist but don't equal limits (Burstable), or neither is set (BestEffort) — and it directly maps to OOM kill priority."
- "Under memory pressure, BestEffort dies first, Burstable dies next ordered by how far over its request it's grown, and Guaranteed is effectively protected because the kubelet already accounted for its exact footprint."
- "A toleration permits scheduling onto a tainted node, it doesn't force it — you need node affinity too if you actually want a Pod to land on specific tainted nodes, not just be allowed to."
- "NoExecute is the taint effect that evicts already-running Pods, not just blocks new ones — the mechanism behind automatic node-condition taints like not-ready and memory-pressure, and why DaemonSets need explicit tolerations to keep observing unhealthy nodes."

## Lab: verified on minikube

**Part 1 — QoS class derivation:** confirmed `Guaranteed`/`Burstable`/`BestEffort` pods correctly show their derived class via `.status.qosClass`.

**Part 2 — Taints/tolerations:** tainted the node `dedicated=special:NoSchedule`; a Pod without matching toleration stayed `Pending`; the same Pod spec with the toleration scheduled and ran `Running` — proved the block/permit mechanism directly.

**Part 3 — Node-level QoS eviction attempt (partially successful, documented honestly):** attempted to force real node-wide `MemoryPressure` by running BestEffort vs Guaranteed stress pods and constraining minikube's VM memory. Ran into two practical obstacles worth remembering as real-world gotchas:
- `minikube start --memory=1800mb` did not reliably constrain the actual VM (node kept reporting `Allocatable: ~16GB` regardless) — minikube's docker driver can ignore the memory flag when the underlying Docker Desktop allocation is already large.
- `kubectl top` requires the `metrics-server` addon, not enabled by default (`minikube addons enable metrics-server` fixes it).

Given the node's real capacity was much larger than intended, cluster-wide `MemoryPressure` never triggered within a reasonable time. **Node-level QoS eviction ordering (BestEffort → Burstable → Guaranteed) is documented, reliable Kubernetes behavior** even though today's lab didn't force-trigger it end-to-end — worth revisiting later with a properly memory-constrained VM (or a cloud-based small node) if a from-scratch reproduction is wanted.

**Part 3b — Container-level OOMKill (what we actually captured live):** deployed a single container with `requests=limits=50Mi` memory trying to allocate 150M via `stress --vm-bytes 150M`. Result:
```
State: Terminated
  Reason: OOMKilled
  Exit Code: 1 (kubectl's summary field; the kernel's actual OOM signal is SIGKILL, conventionally surfaced as exit code 137 = 128+9)
Restart Count: 2
```
Confirmed the kernel's cgroup memory limit enforcement kills a container instantly on exceeding its own limit, independent of node-wide memory state — the mechanism underlying OOM kills generally, demonstrated reliably regardless of node size (unlike the node-eviction attempt above).

**Bonus gotcha caught live:** the same pod's `QoS Class` showed `Burstable`, not `Guaranteed`, despite memory requests=limits — because no CPU requests/limits were set at all. Confirms: **Guaranteed requires requests=limits on every resource type, not just memory** — partial coverage silently downgrades to Burstable.

---
*This closes out the full 7-item K8s batch: kubectl commands, Ingress, NetworkPolicy, RBAC, kube-proxy/CoreDNS/pod routing, CNI, and HPA/QoS/taints. Still pending from the wider networking remediation block: iximiuz.com container networking series + "Life of a Packet" KubeCon talk.*
