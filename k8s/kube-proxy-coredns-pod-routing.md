# kube-proxy (iptables/ClusterIP), CoreDNS, Pod-to-Pod Routing

> K8s core objects deep dive, continued from RBAC. Includes live minikube lab: iptables chains, conntrack DNAT, ss verification.

## The problem this stack solves

Pods are ephemeral — IPs change constantly. Nothing should hardcode a Pod IP. **Service** is K8s's stable abstraction over a shifting set of Pods; **ClusterIP** (default type) is a stable virtual IP load-balancing to whatever Pods currently match the selector. Core question: ClusterIP isn't a real interface anywhere — nothing "listens" on it — so how does traffic sent there reach a real Pod?

## kube-proxy: makes ClusterIP real

Runs as an agent on every node. Watches API server for Service/Endpoints/EndpointSlice objects, programs the **node's own packet-handling rules** so ClusterIP-destined traffic gets rewritten to a real Pod IP, usually before leaving the node.

**Three modes:**
- **userspace** (K8s 1.0-era) — kube-proxy itself in the data path as a real proxy process; slow (kernel/userspace crossing twice); deprecated, essentially unused now
- **iptables** (long-time default) — kube-proxy programs kernel netfilter/iptables rules directly; no proxy process in the data path, kernel does DNAT itself
- **IPVS** (newer, opt-in) — kernel IP Virtual Server module, O(1) hash-table lookup vs iptables' linear chain evaluation — real perf win at scale (thousands of Services = thousands of sequential iptables rules to walk)

## How ClusterIP via iptables works, mechanically

ClusterIP (e.g. `10.96.45.12`) is allocated from a reserved CIDR that's **never actually routed anywhere** — it only means something because every node's iptables rules know to intercept and rewrite traffic to it.

**Chain structure:**
1. **`KUBE-SERVICES`** — entry chain, hooked into `PREROUTING` + `OUTPUT`; every packet's destination checked against it
2. Match → jump to **`KUBE-SVC-<hash>`**, specific to that Service
3. That chain has one **`KUBE-SEP-<hash>`** rule per backing Pod, each with a `--probability` value implementing weighted random selection (e.g. 3 Pods → ~0.333, 0.5, 1.0 cascading probabilities → ~1/3 each)
4. Matched `KUBE-SEP` rule does **DNAT** — rewrites destination from ClusterIP to a specific Pod IP:port
5. Return traffic un-DNAT'd automatically via **conntrack** — client never sees the real Pod IP

**Key point:** load-balancing decision happens **once per connection**, at DNAT time — not per-packet. An established connection keeps going to the same Pod even if Endpoints change mid-connection (via conntrack), until it naturally closes.

**Why rules regenerate constantly:** every Endpoints change (Pod added/removed/unready) → kube-proxy regenerates `KUBE-SVC`/`KUBE-SEP` chains; older versions recalculated/reinstalled the entire ruleset per change — slow at scale, the concrete motivation behind IPVS mode and newer proxy implementations.

## CoreDNS: name → ClusterIP

Runs as Pods in `kube-system`, fronted by its own ClusterIP (a fixed, well-known IP baked into every Pod's `/etc/resolv.conf` — solves the "how do you resolve DNS before you can resolve DNS" bootstrap problem via a static, non-DNS-dependent address).

**Watches the API server directly** for Service/Endpoints objects — same event-driven model as kube-proxy, different consumer, goes through the API server not etcd directly. New Service → new DNS record automatically, no manual management.

**Two record patterns:**
- Normal ClusterIP Service: `<service>.<namespace>.svc.cluster.local` → A record → the Service's ClusterIP
- Headless Service (`clusterIP: None`, used by StatefulSets): **one A record per backing Pod**, each pointing directly at a Pod IP — the actual DNS mechanism behind StatefulSet's per-replica addressability (`mysql-0.mysql.default.svc.cluster.local`)

**Resolution flow:** app calls `web` → glibc resolver reads `/etc/resolv.conf` (kubelet-injected, points at CoreDNS ClusterIP + search domains like `default.svc.cluster.local`) → CoreDNS returns the Service's ClusterIP → app connects → kube-proxy's iptables DNAT takes over from there.

**One-liner:** "CoreDNS answers 'what's the ClusterIP for this name,' kube-proxy answers 'what real Pod does this ClusterIP actually mean right now' — DNS resolution happens once at connection time, then iptables DNAT + conntrack handle every packet in that connection without another DNS lookup."

## Pod-to-pod routing across nodes (CNI's job, layer underneath Services)

Every Pod gets a real, cluster-routable IP from the CNI's Pod CIDR (e.g. `10.244.0.0/16` cluster-wide, node subnet slices like `10.244.1.0/24`). **Pod IPs must be directly reachable, no NAT, from any Pod/node in the cluster** — K8s's flat-network requirement. This is what lets Pod-to-Pod communication skip Service/ClusterIP entirely if the Pod IP is already known.

**Packet path, Node A Pod → Node B Pod (shape, varies by CNI):**
1. Leaves Pod via veth pair into node's netns
2. Node's routing table/bridge determines destination isn't local
3. Encapsulated (VXLAN overlay — Flannel default) or natively routed via BGP (Calico default) to destination node
4. Destination node de-encapsulates if needed, delivers to correct local Pod's netns via its own veth setup

**Overlay (VXLAN) vs native (BGP):**
- Overlay (Flannel VXLAN) — wraps original packet in a new outer packet, tunneled between nodes; works over any underlying network without router config changes, but adds encapsulation overhead (extra header, reduced MTU, encap/decap CPU cost)
- Native/BGP (Calico default) — each node announces its Pod subnet via BGP, packets routed directly with real IP routing; no encapsulation overhead, better performance, but requires underlying network support (easy on a flat L2/L3 network you control, harder in restrictive cloud setups)

## Connecting to `ss -tnp`

- **Local/remote address shows a real Pod IP**, never a ClusterIP — by the time a connection is established and visible in `ss`, DNAT has already happened; the kernel's socket table reflects the real endpoint. **You will never see a ClusterIP as an actual connected endpoint in `ss` output** — it only exists as a DNAT target.
- Cross-node connections show real Pod IPs from different node subnets — direct evidence of the flat, NAT-free Pod network.
- Debugging use: `ss -tnp` showing `SYN-SENT` stuck (never completes) to a Pod IP → check the CNI layer (routing/encapsulation broken between nodes), not kube-proxy/CoreDNS — those two only matter *before* the connection exists.

## Lab: traced a real request through CoreDNS → iptables DNAT → conntrack (minikube)

**Setup:** `web` Deployment (nginx, 3 replicas) + ClusterIP Service, resolved via `nslookup web.default.svc.cluster.local` from a debug pod — confirmed CoreDNS returns the exact ClusterIP shown by `kubectl get svc web`.

**conntrack result after a real curl request** (`sudo conntrack -L | grep <ClusterIP>` on the minikube node):
```
tcp 6 59 TIME_WAIT src=10.244.120.73 dst=10.105.218.40 sport=59610 dport=80 ...
         src=10.244.120.72 dst=10.244.120.73 sport=80 dport=59610 ...
```
Read as two directions of one flow:
- **Original direction:** debug pod (`10.244.120.73`) → ClusterIP `10.105.218.40:80` — what the app actually sent
- **Reply direction (post-DNAT):** real Pod `10.244.120.72:80` replying to debug pod — conntrack transparently substituted the ClusterIP for the real Pod on the way back

Confirms: app addressed the ClusterIP, but the real conversation was with a specific nginx Pod — conntrack made the substitution invisible to both sides. Same TCP TIME_WAIT state from the earlier TCP session, seen again here in a live K8s context.

**Tooling note:** `curlimages/curl` (Alpine, minimal, non-root) has no `ss`/`iproute2` and can't `apk add` due to permission restrictions on its package cache — don't fight it. Use `nicolaka/netshoot` instead for any lab needing `ss`, `dig`, `tcpdump`, `ip`, etc. — worth keeping as the default debug image going forward instead of bare curl images.

## Interview one-liners, compiled

- "ClusterIP is a virtual address that's never actually routed anywhere — it only has meaning because every node's iptables rules know to DNAT traffic addressed to it toward a real Pod IP."
- "The load-balancing decision happens once, at connection/DNAT time, via conntrack — not per-packet — which is why an established connection keeps going to the same Pod even if Endpoints change mid-connection."
- "CoreDNS and kube-proxy both watch the same API server events but solve different problems: CoreDNS turns a name into a ClusterIP, kube-proxy turns that ClusterIP into a real Pod IP via DNAT."
- "Headless Services skip the ClusterIP layer entirely — CoreDNS returns one A record per Pod instead of one record for a virtual IP, which is the actual DNS mechanism behind StatefulSet's per-replica addressability."
- "Pod-to-Pod networking is a flat, NAT-free model by K8s design — every Pod IP must be directly reachable from anywhere in the cluster, which is the CNI's job via either overlay encapsulation (Flannel/VXLAN) or native BGP routing (Calico)."
- "In `ss` output you'll only ever see real Pod IPs, never ClusterIPs — by the time a connection exists, DNAT has already resolved it to something real."

---
*Next up: CNI plugins deep dive — what problem CNI solves, Flannel vs Calico vs Cilium (architecture, NetworkPolicy support, performance, when to pick each).*
