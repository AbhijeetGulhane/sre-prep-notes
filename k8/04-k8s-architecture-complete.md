# Kubernetes Architecture — Complete Study Notes
## Control Plane, Data Plane, and Pod Scheduling Trace

---

## 1. The Foundational Mental Model

Kubernetes is a **control loop system** built on one architectural pattern repeated everywhere:

```
Observe current state → Compare against desired state → Act to reconcile the difference
```

Every controller does exactly this, nothing more. The entire system — Deployment controller, ReplicaSet controller, scheduler, kubelet — is a collection of these loops operating concurrently on different slices of cluster state.

**Key principle: level-triggered, not edge-triggered.** Controllers don't act only on change events — they continuously reconcile against the full desired state on every iteration. This is what makes Kubernetes self-healing: a controller doesn't care *why* something is wrong, only that it is.

---

## 2. The Two Planes

**Control plane (master nodes):** Maintains and acts on desired state. Components: etcd, kube-apiserver, kube-scheduler, kube-controller-manager.

**Data plane (worker nodes, and masters in small clusters):** Makes desired state real on each node. Components: kubelet, kube-proxy, container runtime (containerd).

**Critical architectural constraint:** Every component talks exclusively to **kube-apiserver**. Nothing talks to etcd directly. kube-apiserver is the single gateway to cluster state.

---

## PART A: Control Plane Components

## 3. etcd

### What It Is

etcd is the **single source of truth** for the entire cluster. Every piece of desired and observed state — Pod specs, ConfigMaps, Secrets, Service definitions, node registrations, bindings — lives in etcd and nowhere else. The rest of the control plane is effectively a cache and a set of processes that act on what's in etcd.

Technically: a distributed key-value store built on the **Raft consensus algorithm**.

### Raft and Quorum

Raft guarantees writes are acknowledged only once a **majority quorum** of etcd members has durably committed the entry:
- 3-node cluster: needs 2 nodes to confirm → tolerates 1 failure
- 5-node cluster: needs 3 nodes to confirm → tolerates 2 failures
- Formula: can tolerate `(n-1)/2` failures for n-node cluster

Below quorum: etcd goes **read-only** to prevent split-brain (two partitioned halves each believing they're authoritative and diverging state).

### Data Model

```
/registry/pods/<namespace>/<name>       ← Pod objects
/registry/services/specs/<namespace>/<name>  ← Service objects
/registry/deployments/<namespace>/<name>
/registry/nodes/<name>
/registry/secrets/<namespace>/<name>
```

Values are serialized **protobuf** (not JSON — switched for performance). etcd also supports **watches** — clients register interest in a key prefix and get notified on any change. This is the foundational mechanism behind all reactivity in Kubernetes.

### Impact If etcd Dies

| Scenario | Impact |
|----------|--------|
| One etcd node down (3-node cluster) | Cluster still operational, quorum maintained |
| Two etcd nodes down (3-node cluster) | **etcd goes read-only**, control plane writes stop |
| All etcd nodes down | Control plane completely frozen; running Pods continue running |

Running Pods are **not immediately affected** — kubelet doesn't depend on etcd directly, only on kube-apiserver. But the control plane can't act on any new desired state.

**Recovery:** Restore quorum by bringing members back, or from a snapshot (`etcdctl snapshot restore`). etcd backup is non-negotiable SRE practice for self-managed clusters.

**Performance note:** etcd is not designed for high write throughput — it's optimized for consistency of configuration data. A misconfigured controller in a tight update loop can saturate etcd and cause cluster-wide slowdowns. etcd latency directly translates to apiserver latency.

---

## 4. kube-apiserver

### What It Is

The API server is the **only component that talks directly to etcd** and the entry point for every operation on the cluster. It is stateless (all state is in etcd) — multiple apiserver replicas can run behind a load balancer for HA.

### Request Lifecycle — Memorize This Sequence

1. **Transport security** — TLS on every request, always

2. **Authentication** — who is making this request?
   - x509 client certificates (kubectl default)
   - Bearer tokens (Service Account tokens, OIDC tokens)
   - Webhook authn (delegating to an external service)
   - On failure: 401 Unauthorized

3. **Authorization** — is this identity allowed to do this action on this resource?
   - Almost universally RBAC in modern clusters
   - Checks: verb (get/list/create/update/delete/watch), resource type, namespace, name
   - On failure: 403 Forbidden

4. **Admission control** — two-phase pipeline running after authz, before the write:
   - **Mutating admission webhooks** (run first): can modify the incoming object
     - LimitRanger: injects default resource requests if not set
     - Service mesh sidecar injectors: add sidecar containers to Pod specs
     - DefaultStorageClass: assigns default storage class to PVCs
   - **Validating admission webhooks** (run second): can only approve or reject
     - PodSecurity: enforce pod security standards
     - OPA/Gatekeeper: custom policy enforcement
     - Other policy engines
   - Built-in admission plugins woven throughout (NamespaceLifecycle, ResourceQuota, etc.)

5. **Schema validation** — verify the object conforms to its API type definition

6. **Write to etcd** — object persisted

7. **Return to client** — 200/201/202 with the stored object

### Watch Mechanism and Informers

After a write lands in etcd, etcd notifies apiserver (which watches its own writes), which fans out notifications to all registered watchers. In the Go client library (client-go), this is the **informer** pattern:

- Component registers a watch for a resource type
- Gets a stream of add/update/delete events
- Maintains a local in-memory **cache (store)** — most reads hit the cache, not etcd
- The reconciliation loop: **watch event → reconcile against cache → act → write result**

### Impact If apiserver Dies

- kubectl stops working entirely
- No new Pods can be scheduled (scheduler can't read desired state or write bindings)
- Controllers stop reconciling (can't read or write)
- Running Pods **continue running** — kubelet has a local cache and continues operating
- A brief apiserver outage is less catastrophic than it sounds for running workloads

---

## 5. kube-scheduler

### What It Does

Exactly one thing: **watch for Pods with no `spec.nodeName` assigned, and assign one.**

Output: writes a `Binding` object (or patches `spec.nodeName`) to kube-apiserver. That's its entire observable effect.

### Two-Phase Scheduling Cycle

**Phase 1 — Filtering (eliminate ineligible nodes):**

Every node is run through a series of hard filters. Any node that fails is eliminated from the candidate set:

| Filter | What it checks |
|--------|---------------|
| `NodeUnschedulable` | Node is cordoned (`spec.unschedulable: true`) |
| `PodFitsResources` | Node has enough allocatable CPU + memory for Pod's *requests* (not limits) |
| `NodeAffinity` | `requiredDuringSchedulingIgnoredDuringExecution` rules |
| `TaintToleration` | Pod must tolerate all taints on the node |
| `VolumeBinding` | PVCs can be bound to volumes available on this node |
| `PodTopologySpread` | `required` topology spread constraints |
| Pod affinity/anti-affinity | `required` rules |

Result: a candidate node list (could be all nodes, could be zero).

**Phase 2 — Scoring (rank eligible nodes):**

Remaining candidates are scored 0–100 on each scoring function, combined with weights:

| Scoring function | Prefers |
|-----------------|---------|
| `LeastAllocated` | Nodes with more free resources (spreading) |
| `MostAllocated` | Nodes with less free resources (bin-packing) |
| `NodeAffinityPriority` | Soft (preferred) affinity rules |
| `InterPodAffinity` | Preferred affinity/anti-affinity |
| `TaintToleration` | Nodes with fewer relevant taints |
| `ImageLocality` | Nodes that already have the container image |

Node with highest combined score wins. Ties broken arbitrarily.

**Binding:**
Scheduler writes a `Binding` object to apiserver → apiserver writes `spec.nodeName` to etcd → kubelet on target node's watch fires.

### Preemption

If no node passes filtering for a high-priority Pod, the scheduler can:
1. Find a node where evicting lower-priority Pods would create enough room
2. Evict those pods (SIGTERM → grace period)
3. Reschedule the high-priority Pod there

### Impact If Scheduler Dies

- New Pods pile up in `Pending` indefinitely (they exist in etcd with no `nodeName`)
- Running Pods completely unaffected
- Graceful degradation — cluster operates but can't accommodate new workload

---

## 6. kube-controller-manager

### What It Is

A single binary running **many independent control loops** concurrently. Each controller manages one aspect of cluster state.

Major controllers:
- **Deployment controller** — manages ReplicaSets based on Deployment specs
- **ReplicaSet controller** — creates/deletes Pods to match desired replica count
- **StatefulSet controller** — manages ordered, persistent Pod sets
- **DaemonSet controller** — ensures one Pod per node (or per matching node)
- **Job/CronJob controller** — manages batch workloads
- **Namespace controller** — handles namespace lifecycle and finalizers
- **ServiceAccount controller** — auto-creates default SAs and tokens
- **Node controller** — monitors node heartbeats, marks NotReady, triggers eviction
- **Endpoints controller** — populates Endpoints objects from Services and Pods
- **EndpointSlice controller** — same, but the scalable modern replacement

### The Reconciliation Loop Pattern

```go
for {
    desired  := informer.GetDesiredState()  // from local cache
    current  := informer.GetCurrentState()  // from local cache
    
    if current != desired {
        err := actToMakeCurrentMatchDesired()
        if err != nil {
            requeueAfter(backoff)
        }
    }
    waitForNextWatchEventOrRequeue()
}
```

### Deployment Controller in Detail

Deployment → manages ReplicaSets → ReplicaSet controller manages Pods. Two layers:

**On creation:** Deployment controller creates a ReplicaSet with the Pod template. ReplicaSet controller creates Pods up to `replicas` count.

**On rolling update (new image, env var change, etc.):**
1. Create new ReplicaSet with new Pod template
2. Scale up new RS by `maxSurge` (default 1 extra Pod)
3. Wait for new Pods to become ready (pass readiness probe)
4. Scale down old RS by `maxUnavailable` (default 1)
5. Repeat until new RS is at full count, old RS is at zero
6. Old RS is kept (scaled to 0) for rollback purposes

**`kubectl rollout undo`** scales the old RS back up and scales the new RS down.

### Node Controller

- Watches node heartbeats (kubelet updates `NodeStatus` every `nodeStatusUpdateFrequency`, default 10s)
- If heartbeat missing for `node-monitor-grace-period` (default 40s) → marks node `NotReady`
- Applies taint `node.kubernetes.io/not-ready:NoExecute`
- Pods without a toleration for this taint are **evicted** (not immediately killed — they get a grace period per their `tolerationSeconds`, default 300s = 5 minutes in older versions; behavior evolved in newer K8s)

### Impact If kube-controller-manager Dies

- Control loops stop
- A Deployment at 3/5 replicas (because 2 nodes failed) stays at 3 — nothing brings Pods back
- Jobs don't progress
- DaemonSets don't place Pods on new nodes
- Running Pods completely unaffected — this is purely a "desired vs actual, act to reconcile" failure

---

## PART B: Data Plane Components

## 7. kubelet

### What It Is

The **node agent** — runs on every node, bridges control plane desired state and container runtime actual state.

### How It Knows What to Run

kubelet has an informer watching kube-apiserver for Pods where `spec.nodeName` matches its own node. When the scheduler writes a binding, kubelet's watch fires and it takes ownership of that Pod.

### Pod Lifecycle on kubelet

1. **Admission check** — verifies Pod fits against node's *actual* current allocatable resources (scheduler used cached data; kubelet verifies live)

2. **Cgroup setup** — creates pod-level parent cgroup, per-container child cgroups, sets `oom_score_adj` per container based on QoS class

3. **Volume preparation** — mounts PVs, projects ConfigMaps/Secrets as files, sets up `emptyDir`

4. **Network setup** — calls the **CNI plugin** to wire the Pod's network namespace: create veth pair, assign Pod IP, set up routes

5. **Image pull** — calls containerd's CRI `ImageService.PullImage()` for each image not already on node

6. **Container creation** — calls containerd's `RuntimeService.RunPodSandbox()` (creates pause container and shared network namespace), then `CreateContainer()` + `StartContainer()` per app container

7. **Health management** — runs liveness/readiness/startup probes continuously

8. **Status reporting** — updates Pod status via apiserver: container statuses, Pod IP, phase

### The Three Probe Types

| Probe | Purpose | On Failure |
|-------|---------|-----------|
| **Startup** | Give slow-starting apps time to initialize | Suppresses liveness/readiness until it succeeds; kills+restarts container if it never succeeds past `failureThreshold` |
| **Liveness** | Is this container healthy enough to keep running? | Kills and restarts the container (per `restartPolicy`) |
| **Readiness** | Is this container ready to serve traffic? | Removes Pod IP from Endpoints/EndpointSlice — traffic stops. **Does NOT restart the container.** |

**Probe mechanisms:**
- `exec` — run a command inside the container; exit 0 = success
- `httpGet` — HTTP GET to a path/port; 2xx-3xx = success
- `tcpSocket` — TCP connection to port; connection established = success
- `grpc` — gRPC health check protocol

**Key misconfiguration antipatterns:**
- Liveness probe that checks an external dependency: if the database is down, all pods restart in a loop — doesn't fix anything, makes things worse
- Startup probe timeout too short: pods restart continuously during legitimate slow initialization
- No readiness probe: traffic sent to pods that aren't ready yet (during startup, after a restart)
- Readiness and liveness pointing at the same endpoint: defeats the purpose of having both

### kubelet and the CRI

kubelet doesn't call containerd directly — it uses the **CRI (Container Runtime Interface)**, a gRPC API:
- `RuntimeService` — run/stop/list containers, exec into containers, get logs
- `ImageService` — pull/list/remove images

This makes the runtime pluggable — kubelet works the same with containerd, CRI-O, or any other CRI-compliant runtime.

### Impact If kubelet Dies

- Running containers **keep running** (container runtime doesn't stop)
- No new containers can be started
- No health probes run → crashed containers stay crashed
- Node heartbeat stops → node goes `NotReady` after 40s
- Node controller taints the node → pods evicted after grace period

---

## 8. kube-proxy

### What It Is

Watches kube-apiserver for Services and Endpoints/EndpointSlices changes and translates them into network routing rules on the node so that traffic to a ClusterIP (a virtual IP on no interface) reaches a real Pod IP.

kube-proxy handles **no actual traffic itself** — it only programs the kernel's networking subsystem.

### iptables Mode (Traditional)

For a Service with ClusterIP 10.96.0.100 and 3 backends (10.244.1.5, 10.244.2.7, 10.244.3.2):

```
PREROUTING → KUBE-SERVICES → match dst 10.96.0.100:port → KUBE-SVC-XXXX chain:
  Rule 1: probability 0.333 → DNAT to 10.244.1.5:targetPort
  Rule 2: probability 0.500 → DNAT to 10.244.2.7:targetPort  (= 1/2 of remaining 2/3)
  Rule 3: probability 1.000 → DNAT to 10.244.3.2:targetPort  (remainder)
```

Result: each endpoint gets ~1/3 of traffic. DNAT happens before routing — `conntrack` tracks the mapping for return traffic (reverse DNAT applied automatically).

**Limitation:** O(n) rule scan per new connection. Measurable performance degradation at several thousand Services.

### IPVS Mode (Preferred for Large Clusters)

Programs IPVS (IP Virtual Server) rules instead. IPVS uses a **hash table** — O(1) lookup regardless of number of Services. Supports richer load balancing: round-robin (default), least-connections, source-hash. Significantly better at scale.

### Impact If kube-proxy Dies

- Existing iptables/IPVS rules **stay in place** — existing Service routing continues working
- Changes don't propagate: new Services don't get rules, scaled-out Pods don't get added, dead Pod IPs don't get removed
- Over time, traffic increasingly hits stale backend entries → connection errors

---

## 9. containerd and runc

### containerd

The container runtime — speaks CRI to kubelet, manages images, creates containers.

**What it does per container:**
1. Pulls and unpacks OCI images (manages the image layer store)
2. Creates the overlayfs mount stack (base image layers + writable top layer)
3. Calls runc to do the actual process creation

### runc

The **low-level OCI runtime** — takes an OCI runtime spec (JSON file) and makes the corresponding kernel calls.

**What runc does:**
1. Calls `clone()` with namespace flags: `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC` (+ `CLONE_NEWUSER` for rootless)
2. Writes the process into the cgroup path kubelet created
3. Calls `pivot_root()` to change the root filesystem to the overlayfs rootfs
4. Mounts `/proc`, `/dev`, `/sys` inside the new mount namespace
5. Calls `execve()` to run the container's entrypoint

This is exactly where virtual memory, cgroups, namespaces, and VFS all converge — everything from the Linux internals sessions made real.

### The Pause Container

Every pod has a hidden "infra" container running the minimal `pause` binary:
- Created first, before any app containers
- Holds the shared network and IPC namespaces open
- Does nothing in an infinite loop
- When an app container crashes and kubelet restarts it, `pause` keeps the network namespace (and Pod IP) alive
- Without pause: a container restart would destroy the network namespace, changing the Pod's IP
- Visible on nodes via `crictl ps` or `docker ps` — one per pod, always

---

## 10. Component Failure Reference Table

| Component | Running pods affected? | New scheduling? | Control loops? | How to recover |
|-----------|------------------------|-----------------|----------------|----------------|
| etcd (quorum lost) | ❌ No | ❌ No | ❌ No | Restore quorum or snapshot restore |
| kube-apiserver | ❌ No | ❌ No | ❌ No | Restart; stateless, horizontally scalable |
| kube-scheduler | ❌ No | ⚠️ No (Pods pile up Pending) | N/A | Restart |
| kube-controller-manager | ❌ No | N/A | ⚠️ No (loops stop) | Restart |
| kubelet | ⚠️ Keep running, no new starts | N/A | N/A | Restart; node goes NotReady in ~40s |
| kube-proxy | ⚠️ Stale rules stay | N/A | N/A | Restart; new Service changes don't propagate |
| containerd | ⚠️ Running containers survive | N/A | N/A | Restart; new container creates fail |

---

## 11. Full Pod Scheduling Trace — Narrate Without Notes

`kubectl apply -f pod.yaml` → Pod is Running. Every step:

**Step 1 — kubectl to kube-apiserver:**
kubectl serializes Pod spec, sends authenticated HTTPS POST. apiserver pipeline: authenticate → authorize → mutating webhooks → validating webhooks → schema validation → write to etcd. Pod now exists: `nodeName` empty, `phase: Pending`.

**Step 2 — Scheduler watch fires:**
Scheduler has informer watching Pods with no `nodeName`. etcd notifies apiserver of new write → apiserver fans out to scheduler's informer → Pod lands in scheduler's unscheduled queue.

**Step 3 — Scheduling cycle:**
Scheduler filters nodes (resources, taints, affinity, volumes). Scores remaining candidates (spreading, affinity, image locality). Picks highest-scoring node.

**Step 4 — Binding:**
Scheduler writes `Binding` to apiserver → apiserver writes `spec.nodeName` to etcd. Pod now has a node.

**Step 5 — kubelet watch fires:**
kubelet on target node has informer watching Pods with `spec.nodeName` = its own name. Updated Pod triggers the watch. kubelet takes ownership.

**Step 6 — kubelet setup:**
Admission check (live resource fit). Create pod-level + per-container cgroups. Set `oom_score_adj`. Call CNI plugin: create veth pair, set up network namespace, assign Pod IP. Prepare volumes.

**Step 7 — Image pull:**
kubelet calls containerd's `ImageService.PullImage()` for each image not on node. containerd fetches layers from registry.

**Step 8 — Container creation:**
kubelet calls `RunPodSandbox()` → containerd creates pause container (establishes shared network/IPC namespace). Then `CreateContainer()` + `StartContainer()` per app container: containerd invokes runc → runc calls `clone()` with namespace flags → writes process into cgroup → `pivot_root()` into overlayfs rootfs → `execve()` entrypoint.

**Step 9 — Status reported back:**
kubelet updates Pod status via apiserver: `containerStatuses[].state.running`, Pod IP, `phase: Running`. Written to etcd. Anyone watching Pods now sees it Running.

---

## Retention Questions and Answers

**Q1: What is the only component allowed to talk to etcd directly, and why does that constraint exist architecturally?**

Only **kube-apiserver** talks directly to etcd. The constraint exists for several reasons: it centralizes the authentication, authorization, and admission control pipeline — every write to cluster state must pass through these gates. It makes etcd's security posture simpler (only one client to authenticate). It allows apiserver to maintain an authoritative watch/notification fan-out layer so other components only need to watch apiserver, not etcd. And it keeps etcd's load manageable — if every controller directly watched and wrote etcd, the load would be much higher and harder to reason about.

**Q2: Walk through the kube-apiserver request pipeline in order — what are the stages before a write hits etcd?**

1. **Transport security** (TLS)
2. **Authentication** — verify identity (x509 cert, bearer token, webhook) → 401 if fail
3. **Authorization** — verify identity is permitted this action → 403 if fail
4. **Admission control** — mutating webhooks (can modify object) then validating webhooks (can only approve/reject)
5. **Schema validation** — verify object conforms to its API type
6. **Write to etcd**

**Q3: A Deployment is stuck at 3/5 replicas after two nodes failed. Pods are not coming back up. Control plane looks healthy. What's your first hypothesis?**

kube-controller-manager is down or its reconciliation loop for Deployments/ReplicaSets has stalled. The Deployment controller (inside kube-controller-manager) is responsible for detecting that actual replica count (3) doesn't match desired (5) and creating the 2 missing Pods. If the controller loop isn't running, that comparison never happens and the deficit persists indefinitely. Check kube-controller-manager logs and status, look for errors or crashes. Secondary hypothesis: the scheduler is down — Pods may have been created but remain `Pending` indefinitely without a node assignment.

**Q4: Explain what the pause container is, why it exists, and what would break if it didn't.**

pause is a minimal binary (does an infinite `pause()` syscall loop) that runs as PID 1 in every pod's shared namespaces before any app containers start. Its job is to hold the shared network and IPC namespaces open.

Why: namespaces exist as long as at least one process holds them. If your app container crashes and kubelet restarts it, there's a window where no process is in the network namespace — without pause, the namespace (and the Pod's IP address) would be destroyed. The replacement container would get a new IP, breaking Service routing and any client connections.

With pause: it's always running, always holding the namespace. The Pod's IP survives container restarts. This is a fundamental enabler of the "pod has a stable IP during its lifetime" guarantee.

**Q5: A Service has a healthy Pod backing it, but traffic is still hitting an old dead Pod IP. kubelet and the container runtime are healthy. What component do you look at first, and why?**

**kube-proxy.** Traffic routing for Services is implemented as iptables/IPVS rules programmed by kube-proxy. If kube-proxy is down or has crashed, existing rules stay in place (the kernel rules don't disappear when kube-proxy stops) but changes don't propagate — a dead Pod's IP remains in the rule set even after its Endpoints entry is removed. The Endpoints controller (in kube-controller-manager) may have already removed the dead Pod IP from the Endpoints object in etcd, but kube-proxy hasn't picked up that change and updated the iptables/IPVS rules on the node. Check kube-proxy status and logs on the node receiving the traffic.

**Q6: What's the difference between a liveness probe failure and a readiness probe failure — mechanically, what does kubelet do differently in each case?**

**Liveness probe failure:** kubelet determines the container is not healthy enough to continue running. It kills the container (sends SIGTERM, waits grace period, then SIGKILL if still running) and restarts it according to the Pod's `restartPolicy`. The container gets a fresh start.

**Readiness probe failure:** kubelet determines the container is temporarily not ready to serve traffic. It removes the Pod's IP from the Endpoints/EndpointSlice object via apiserver. kube-proxy picks up this change and removes that Pod IP from the Service's iptables/IPVS rules. Traffic stops flowing to this pod. The container **is not restarted** — it stays running. When the probe succeeds again, the IP is re-added and traffic resumes.

The intent: liveness = "this container is broken, restart it." Readiness = "this container is temporarily unable to serve, drain it without restarting it."

**Q7: Trace the exact path of a kubectl apply -f pod.yaml call through every component until the container process is running.**

kubectl → kube-apiserver (authn/authz/admission/write to etcd, Pod created with no nodeName) → scheduler watch fires (Pod goes into unscheduled queue) → scheduling cycle (filter nodes, score candidates, pick winner) → scheduler writes Binding to apiserver (nodeName set in etcd) → kubelet on target node watch fires (kubelet takes ownership) → kubelet admission + setup (cgroups, oom_score_adj, CNI plugin for network, volumes) → kubelet calls containerd ImageService.PullImage() → kubelet calls containerd RunPodSandbox() (pause container created, shared network namespace established) → containerd calls runc for each app container (clone() with namespace flags, write into cgroup, pivot_root() into overlayfs, execve() entrypoint) → kubelet updates Pod status via apiserver → etcd updated → Pod shows Running.
