# CNI Plugins: What Problem CNI Solves, Flannel vs Calico vs Cilium

> K8s core objects deep dive, continued from kube-proxy/CoreDNS/pod routing. Includes live minikube lab confirming Calico + a real stale-pod recovery.

## The problem CNI solves

Kubernetes has **no built-in networking implementation** — the K8s team deliberately punted this to a plugin interface, the **Container Network Interface (CNI)** spec (originally CoreOS, now CNCF). It's a generic, K8s-agnostic standard for "how does a container get network connectivity on start, and get cleaned up on stop."

**Mechanically:** a thin executable contract. kubelet creates a Pod → calls a CNI plugin binary (`/opt/cni/bin/`) with `ADD` (attach network to new container's namespace) / `DEL` (tear down). Plugin reads JSON config (`/etc/cni/net.d/`), does its own networking setup (veth pair, IP assignment, routes), returns standardized JSON. Deliberately minimal contract — why wildly different implementations (simple overlay vs full eBPF dataplane) can all satisfy the same interface.

**What K8s requires from whichever CNI you pick (K8s's requirement, not CNI's):**
1. Every Pod gets its own IP (no port-mapping/NAT like Docker's default bridge)
2. Pods communicate with all other Pods cluster-wide, no NAT (flat network model)
3. Nodes communicate with all Pods, no NAT
4. The IP a Pod sees itself as = the IP everyone else sees it as (no translation in between)

**Why a plugin, not baked-in:** environments vary hugely (bare metal, AWS, GCP, on-prem VMware) with different network capabilities (BGP support, VXLAN, cloud routing APIs) — one hardcoded implementation can't serve all well. CNI swaps the networking layer per-environment while the rest of K8s stays identical.

**One-liner:** "CNI is a minimal plugin contract, not a networking implementation — kubelet calls a CNI binary with ADD/DEL for each Pod's lifecycle, and what actually happens (overlay tunnel, BGP routing, eBPF dataplane) is entirely up to whichever plugin you installed. Kubernetes just requires the outcome: every Pod gets a real, NAT-free, cluster-routable IP."

## Flannel — simple, get-it-working

**Architecture:** overlay, VXLAN by default (also host-gw, other backends). Each node gets a Pod CIDR subnet slice; `flanneld` per node maintains node→subnet→real-IP mapping (etcd historically, often K8s API now). Cross-node traffic wrapped in VXLAN header, tunneled, unwrapped at destination, delivered via normal bridge.

**Does NOT do:** NetworkPolicy enforcement at all — the single most important fact about Flannel (already covered in NetworkPolicy topic).

**Performance:** VXLAN overhead (extra header, reduced effective MTU, encap/decap CPU cost) — negligible for most workloads, real concern only at high packet-rate/latency-sensitive scale.

**When to pick:** simplicity is the whole value — minimal config, dev/learning clusters, or production with no NetworkPolicy requirement and no extreme perf sensitivity. minikube's historical default for exactly this reason.

## Calico — routing-focused, policy-capable

**Architecture:** native L3 routing via **BGP** by default (not overlay) — each node runs `bird` (BGP daemon), announces its Pod subnet directly, packets routed via real IP routing table entries, no encapsulation. Overlay (IP-in-IP/VXLAN) mode available as fallback where direct BGP peering isn't feasible (some cloud VPC setups).

**NetworkPolicy:** full support — Calico predates and heavily influenced the K8s NetworkPolicy API design. Own extended CRDs (`GlobalNetworkPolicy`, etc.) beyond what standard NetworkPolicy expresses (cluster-wide policies without per-namespace duplication, non-Pod-based rules).

**Performance:** best of the three for raw throughput/latency in native BGP mode — no encapsulation overhead. Tradeoff: BGP needs underlying network support (clean on bare metal/networks you control; some cloud VPCs need the overlay fallback, reintroducing encap cost).

**When to pick:** most common "production-grade, need real NetworkPolicy" default — likely choice at a bank/fintech specifically for compliance-driven segmentation. Also strong when raw performance matters and infra supports native BGP.

## Cilium — eBPF-based, most feature-rich

**Architecture:** fundamentally different mechanism. Uses **eBPF** — programs running directly in the Linux kernel at various hook points (socket layer, XDP/driver layer, traffic control layer) to implement forwarding, load balancing, policy enforcement without traversing iptables chains or userspace proxies.

**Practical implication:** can **replace kube-proxy entirely** (`kube-proxy replacement` mode) — eBPF does ClusterIP translation directly in the kernel packet path instead of iptables DNAT chains, avoiding linear iptables chain-walking entirely — structurally different (more scalable) than even IPVS mode.

**NetworkPolicy:** full K8s support + `CiliumNetworkPolicy` CRD with **L7-aware policies** — genuinely distinguishing: rules like "allow HTTP GET to `/api/public/*`, deny POST to `/api/admin/*`" at the application protocol layer, not just IP/port. Standard K8s NetworkPolicy and Calico's default model can't express this.

**Extra capabilities:**
- **Hubble** — built-in flow-level observability (which Pod talked to which, protocol, allowed/denied by policy) without separate tooling
- **Service mesh features without sidecars** — eBPF operates kernel-level, so mTLS/L7 policy/observability possible without injecting Envoy into every Pod (the architecture Istio/Linkerd traditionally require) — meaningfully lower per-Pod resource overhead

**Performance:** generally best of the three at scale — avoids both iptables linear-chain problem and (in kube-proxy replacement mode) DNAT/conntrack overhead for many operations.

**When to pick:** L7-aware policy needs, built-in observability without a separate stack, or scale where iptables performance becomes a real bottleneck. Tradeoff: genuine operational complexity — newer/more sophisticated than iptables or BGP, needs a sufficiently modern kernel, steeper troubleshooting learning curve (debugging eBPF vs reading iptables rules/BGP routes are very different skills).

## Side-by-side

| | Flannel | Calico | Cilium |
|---|---|---|---|
| Core mechanism | VXLAN overlay | Native BGP routing (overlay fallback available) | eBPF (kernel-level programs) |
| NetworkPolicy support | **None** | Full, plus extended CRDs | Full, plus L7-aware CRDs |
| kube-proxy replacement | No | No | Yes (optional) |
| Performance | Good enough, encap overhead | Best in native mode, no encap overhead | Best at scale, avoids iptables/conntrack overhead |
| Observability | None built-in | Basic | Hubble (flow-level, built-in) |
| Complexity | Lowest | Moderate | Highest |
| Typical fit | Dev/learning clusters, no policy needs | Production needing real NetworkPolicy (compliance-driven) | Large-scale production, L7 policy needs, eBPF-comfortable teams |

## Interview one-liners, compiled

- "CNI is a minimal plugin contract — kubelet calls ADD/DEL, and the actual networking mechanism is entirely up to the plugin. Kubernetes only requires the outcome: every Pod gets a real, NAT-free, cluster-routable IP."
- "Flannel is overlay-based simplicity with zero NetworkPolicy enforcement — fine for dev, insufficient wherever segmentation matters."
- "Calico does native BGP routing with no encapsulation overhead and full NetworkPolicy support — the standard production choice when you need real policy enforcement and your network supports BGP."
- "Cilium uses eBPF instead of iptables or plain routing, can replace kube-proxy entirely, and is the only one of the three offering L7-aware NetworkPolicy — the tradeoff is real operational complexity from a newer, kernel-level technology."

## Lab: confirmed which CNI was running + real recovery from stale pods (minikube)

```bash
kubectl get pods -n kube-system
```
Confirmed **Calico was the running CNI** (`calico-node`, `calico-kube-controllers` present) — carried over from the earlier NetworkPolicy lab session, not Flannel/minikube's default.

**Unplanned but genuinely useful detour:** `calico-kube-controllers` and `coredns` both showed `Error` state, 18h old — leftover from the WSL2 shutdown/Docker Desktop restart cycle earlier in the session, not a CNI misconfiguration. Real debugging flow applied (from the kubectl notes):
```bash
kubectl describe pod -n kube-system -l k8s-app=calico-kube-controllers
kubectl logs -n kube-system -l k8s-app=calico-kube-controllers --previous
kubectl describe pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --previous
```
Confirmed the errors traced to the interrupted restart, not a real config problem — fixed by deleting the stale Pods and letting their controllers recreate fresh ones:
```bash
kubectl delete pod -n kube-system -l k8s-app=calico-kube-controllers
kubectl delete pod -n kube-system -l k8s-app=kube-dns
kubectl get pods -n kube-system   # confirmed all Running afterward
```

**Takeaway worth remembering:** this is a live example of "check `describe`/`logs --previous` before assuming the worst" — the instinct to actually read the error first (rather than immediately nuking and restarting minikube) confirmed it was a transient restart artifact, not something wrong with Calico or the cluster's networking setup.

---
*This closes out the full 7-item K8s batch: kubectl commands, Ingress, NetworkPolicy, RBAC, kube-proxy/CoreDNS/pod routing, and CNI. Remaining from the original batch: K8s HPA/QoS classes/taints-and-tolerations together (scheduler interaction under memory pressure).*
