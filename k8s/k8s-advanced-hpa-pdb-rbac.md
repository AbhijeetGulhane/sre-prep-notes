# K8s Advanced: HPA with Custom Metrics, PodDisruptionBudget, RBAC Impersonation

> Extends the earlier K8s batch (HPA/QoS/taints, RBAC). Includes a live PDB-blocks-drain lab and a full cluster recovery after 10 days idle.

## HPA with custom metrics

Basic HPA scales on `cpu`/`memory` relative to requests. **Custom metrics** let HPA scale on anything exposable as a metric — request rate, queue depth, active connections — because CPU is often a poor proxy for load on I/O-bound services (waiting on a DB/external API can mean real load while CPU sits low).

**Architecture — HPA never talks to Prometheus directly:**
```
HPA → Kubernetes Metrics APIs → Metrics Adapter → Prometheus (or wherever the metric lives)
```

**Three metric API types:**
1. **`metrics.k8s.io`** (Resource Metrics API) — CPU/memory only, served by metrics-server. What basic HPA uses.
2. **`custom.metrics.k8s.io`** — metrics tied to a K8s object (a Pod, a Service) — e.g. requests/sec for a Deployment's pods. Needs an adapter (usually **Prometheus Adapter**) translating PromQL into this API's shape.
3. **`external.metrics.k8s.io`** — metrics with no K8s object at all — SQS queue depth, a Pub/Sub backlog. Same adapter pattern for things K8s has no object model for.

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
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

**`Pods` vs `Object` vs `External`:**
- `Pods` — averaged across matching pods (per-pod request rate)
- `Object` — describes a single K8s object directly, not averaged — "queue depth on this Service"
- `External` — no K8s object at all — SQS queue length

**One-liner:** "HPA never talks to Prometheus directly — it queries Kubernetes' custom/external metrics APIs, implemented by an adapter like Prometheus Adapter translating PromQL underneath. CPU is often a poor proxy for real load on I/O-bound services."

## PodDisruptionBudget (PDB) — protects against voluntary disruptions only

**Involuntary disruptions** (hardware failure, kernel panic, VM preemption) — PDB has zero control, nothing to negotiate with.
**Voluntary disruptions** (`kubectl drain`, Cluster Autoscaler scale-down, rolling node upgrade) — exactly what PDB governs; it constrains how many Pods can be disrupted at once, doesn't prevent the disruption itself.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: search-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: search-api
```

`minAvailable: 2` — eviction API refuses to evict a Pod if it would drop available (Ready) matching Pods below 2. `maxUnavailable` is the inverse expression of the same constraint — use one or the other, not both. Fixed count (`minAvailable`) suits stable-size Deployments; percentage (`maxUnavailable: 25%`) suits elastically-scaled ones.

**Enforcement mechanism — the actual point:** PDB is enforced at the **Eviction API**, not by drain tooling being polite. Voluntary-disruption tools (kubectl drain, Cluster Autoscaler) call the Eviction API, and the API server checks the PDB before allowing eviction, returning **429 (Too Many Requests)** if it would violate the budget. A raw `kubectl delete pod` bypasses this entirely — which is also why involuntary disruptions aren't governed: there's no API call to intercept in a hardware crash.

**One-liner:** "PDB only governs voluntary disruptions — drain, autoscaler scale-down, rolling maintenance — because it's enforced by the Eviction API returning 429 if the budget would be violated. It has no power over involuntary disruptions since there's no API call happening to intercept."

## RBAC impersonation

`--as` doesn't switch credentials — you still authenticate as yourself, but the API server checks whether **you** have the `impersonate` verb on the target identity, and if so, evaluates the rest of the request as if it came from that identity. Impersonation is itself a permission, not a free pass.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sa-impersonator
rules:
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["impersonate"]
  resourceNames: ["prometheus-sa"]
```

**Why it exists:**
1. Testing/debugging RBAC (`kubectl auth can-i --as=...`) without extracting a real token
2. Admin support — reproducing "why can't user X do Y" without their credentials
3. Proxy/gateway architectures — authenticate via SSO/OIDC at the edge, impersonate a corresponding K8s identity for the backend call, translating external identity into RBAC without every backend understanding SSO directly

```bash
kubectl get pods --as=jane@example.com --as-group=developers -n search-api
```
Checks what jane, specifically as a `developers` group member, could do — verifies group-based bindings, not just individual ones.

**`can-i` vs real impersonation:** `kubectl auth can-i --as=X` is a dry-run (SelfSubjectAccessReview evaluated as the impersonated identity) — tells you what *would* be allowed without doing it. A real command with `--as` actually executes as that identity. Both use the same `impersonate` verb mechanism underneath.

**One-liner:** "Impersonation isn't switching credentials — you're still authenticated as yourself, but the API server evaluates the rest of the request as the impersonated identity, and doing so requires the impersonate verb on that specific identity. kubectl auth can-i is a dry-run version of this same mechanism."

## Interview one-liners, compiled

- "HPA never talks to Prometheus directly — it queries Kubernetes' custom/external metrics APIs, implemented by an adapter like Prometheus Adapter translating PromQL underneath."
- "Pods-type custom metrics average per-pod; Object-type describes a single object directly; External-type has no Kubernetes object at all."
- "PDB only governs voluntary disruptions, enforced at the Eviction API via 429 — a raw kubectl delete pod bypasses it entirely, and involuntary disruptions never go through an API call PDB could intercept."
- "minAvailable and maxUnavailable express the same constraint two ways — fixed count for stable Deployments, percentage for elastically-scaled ones."
- "Impersonation requires the impersonate verb as a real RBAC permission — you're still authenticated as yourself, the API server just evaluates the request as the impersonated identity."

## Lab: PDB blocking a real drain (live, on minikube)

**Setup:** 3-replica `pdb-demo` Deployment + PDB with `minAvailable: 3` (exactly matching replica count — zero disruptions allowed by design).

```bash
kubectl drain minikube --ignore-daemonsets --delete-emptydir-data --force --timeout=15s
```

**Result:** every `pdb-demo` pod repeatedly refused eviction across 3 retries over the full 15s timeout:
```
error when evicting pods/"pdb-demo-6d8887f669-mt8h7" -n "default" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```
Meanwhile everything else evicted normally — Deployment-managed `search-api`/`prometheus`/`grafana`/`coredns` (self-healed via their controllers), DaemonSet-excluded `calico-node`/`kube-proxy` (skipped via `--ignore-daemonsets`), and the naked `storage-provisioner` pod (required `--force`, since drain refuses to delete controller-less Pods by default — nothing would ever recreate them otherwise). Drain ultimately gave up specifically on `pdb-demo`'s 3 pods, having genuinely never been able to satisfy the budget. Confirmed the Eviction API's 429 behavior live, not just in theory.

**Real-world detour worth remembering:** hit this lab after a genuinely useful, if painful, real recovery scenario — minikube had sat stopped for ~10 days, came back with Calico completely missing (no CNI at all) plus a crash-looping metrics-server and errored storage-provisioner. Rather than debug the accumulated drift piece by piece, did a clean `minikube delete && minikube start --cni=calico --memory=2200mb` rebuild. Confirmed first that all `search-api` K8s manifests were safely committed to git (`git status` clean, `k8s/` directory tracked) before wiping — `minikube delete` was a non-event for anything actually version-controlled. One real gotcha hit during recovery: `search-api:v1`'s image itself lived only in minikube's internal Docker daemon (`imagePullPolicy: Never`), so it vanished with the cluster and needed a fresh `docker build` (via `eval $(minikube docker-env)`) before the Deployment could actually start — manifests coming back from git isn't the same as the image existing.

---
*Advanced K8s batch complete: HPA custom metrics, PDB, RBAC impersonation. Cluster fully recovered from 10-day drift and confirmed healthy.*
