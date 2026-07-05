# Kubernetes Networking — Complete Study Notes
## CNI, Services, CoreDNS, and Ingress

> ⚠️ Note: Networking fundamentals (NAT, routing tables, L2/L3, subnets) are a known gap.
> Before the hands-on session, complete:
> 1. Cloudflare Learning Center (cloudflare.com/learning) — NAT, routing, DNS, subnets
> 2. freeCodeCamp Brian Ferrill — first 3 hours (OSI, IP, routing)
> 3. iximiuz.com "Container Networking Is Simple!" — alongside WSL2 setup
> 4. "Life of a Packet" KubeCon talk (~30 min) — before revisiting this material

---

## 1. The Four Problems K8s Networking Solves

| Problem | Solved By |
|---------|-----------|
| Pod-to-Pod: any Pod reaches any other Pod by IP, no NAT | CNI plugin |
| Pod-to-Service: stable virtual IP load-balancing across backing Pods | kube-proxy |
| External-to-Service: traffic from outside reaches Services | NodePort / LoadBalancer / Ingress |
| Pod-to-external: Pods reach the internet (egress) | NAT masquerade (usually iptables MASQUERADE) |

**The fundamental K8s networking requirement:** Any Pod can reach any other Pod directly by IP, across nodes, **without NAT**. The receiving Pod sees the sender's real Pod IP as the source — no address rewriting. Every CNI plugin must satisfy this.

---

## PART A: CNI — Container Network Interface

## 2. What CNI Is

CNI is a **specification + plugin interface**, not a specific technology. When kubelet creates a Pod, runc sets up a network namespace (`CLONE_NEWNET`) — at this point, the namespace has only a loopback interface. Nothing else. CNI's job is to wire that namespace into the broader network.

kubelet calls a CNI plugin binary (in `/opt/cni/bin/`) passing it a JSON config and the Pod's network namespace path. The plugin does whatever it needs to do and returns the assigned Pod IP. What happens inside the plugin is up to the implementation.

## 3. Pod-to-Pod on the Same Node

**The veth pair:** CNI creates a virtual ethernet pair:
- One end inside the Pod's namespace as `eth0`
- One end in the node's root namespace as `vethXXXXX` (random name)
- A veth pair is a virtual wire — packet in one end comes out the other, regardless of namespace boundaries

**The Linux bridge:** All node-side veth ends connect to a Linux bridge (`cni0` in Flannel, `cbr0` in others). Think: virtual L2 switch in the node's root namespace.

**Same-node flow:**
```
Pod A → eth0 → veth pair → root namespace → bridge → veth pair → Pod B's eth0
```
Pure L2 forwarding — both Pods are on the same bridge subnet, no routing needed.

## 4. Pod-to-Pod Across Nodes — The Core Challenge

Pod A (10.244.1.5, Node 1) wants to reach Pod B (10.244.2.7, Node 2). Pod B must see the packet with Pod A's real IP — no NAT.

Problem: Node 1's kernel doesn't natively know how to route to 10.244.2.0/24 (Node 2's Pod CIDR). Two approaches to solve this:

### Overlay Networks (Encapsulation)

The CNI plugin wraps the original Pod-to-Pod packet in another packet:
- Inner packet: original Pod-to-Pod (src 10.244.1.5, dst 10.244.2.7)
- Outer packet: node-to-node (src Node 1 IP, dst Node 2 IP)

On arrival at Node 2: outer packet stripped, original delivered to Pod B. Pod B sees Pod A's real IP — no NAT.

**VXLAN** (Flannel default, Calico option): encapsulates in UDP port 4789. The encapsulating device on each node is a VTEP (VXLAN Tunnel Endpoint) — `flannel.1` in Flannel. Each VTEP knows how to tunnel to every other VTEP (mapping distributed via apiserver/etcd).

**Cross-node VXLAN flow — step by step:**
1. Pod A sends packet to 10.244.2.7 → leaves `eth0` → crosses veth → root namespace
2. Node 1 routing table: `10.244.2.0/24 via flannel.1` (written by CNI plugin at startup)
3. Packet hits `flannel.1` (VTEP) → VXLAN kernel driver encapsulates: original packet becomes UDP payload, outer src = Node 1 IP, outer dst = Node 2 IP, port 4789
4. Outer UDP packet leaves via Node 1's `eth0` → physical network
5. Arrives at Node 2's `eth0` → kernel sees UDP/4789 → VXLAN driver strips encapsulation → recovers original packet (src 10.244.1.5, dst 10.244.2.7)
6. Node 2 routing: `10.244.2.0/24 via cni0` → bridge → Pod B's veth → Pod B's namespace
7. Pod B sees src = 10.244.1.5 ✅ No NAT happened

**VXLAN overhead:** ~50-byte header per packet. Noticeable for throughput-sensitive workloads.

**IP-in-IP** (Calico option): lighter encapsulation, same concept.

### Native Routing (No Encapsulation)

Instead of wrapping packets, program actual routes so the physical network forwards Pod CIDR packets directly.

**Calico in BGP mode:**
- Each node runs a BGP daemon (BIRD) peering with the network fabric
- Advertises its Pod CIDR via BGP
- Physical routers learn all Pod CIDRs → route Pod traffic directly
- Zero encapsulation overhead
- Requires BGP-capable physical network

**AWS VPC CNI:**
- Assigns actual VPC ENI secondary IPs directly to Pods
- Pod IPs are real VPC IPs, natively routable within the VPC
- Zero encapsulation, native performance

Tradeoff: native routing requires a cooperative physical network. Overlay works anywhere.

## 5. Cilium — eBPF-Based CNI

Increasingly the modern choice. Uses **eBPF programs** loaded into the kernel's network stack instead of iptables or VXLAN:
- No iptables rules at all (eBPF uses hash-table lookups — O(1) vs O(n) scan)
- Network policy enforcement at kernel level with near-zero overhead
- Optional transparent encryption via WireGuard
- Rich observability via Hubble (per-flow network traffic with identity context)
- Can replace both CNI networking AND kube-proxy's Service routing

## 6. IP Address Management (IPAM)

CNI also handles IP allocation:
- Cluster-level Pod CIDR (`--cluster-cidr`) divided into per-node Pod CIDRs
- Per-node CIDR assigned via `spec.podCIDR` on Node objects
- `host-local` IPAM plugin: allocates from the per-node CIDR range stored on disk
- Calico-ipam, Whereabouts: more complex allocation strategies

---

## PART B: Services — Stable Virtual IPs

## 7. Why Services Exist

Pod IPs are ephemeral — a Pod gets an IP when it starts and loses it when it dies. Any client hardcoding Pod IPs breaks constantly. Services provide:
- A stable virtual IP (ClusterIP) that persists regardless of Pod churn
- Load balancing across all healthy backing Pods
- Service discovery via DNS (covered in Part C)

A Service selects backing Pods via a **label selector**. The Endpoints controller maintains an Endpoints/EndpointSlice object listing current healthy Pod IPs. kube-proxy watches this and programs routing rules.

## 8. ClusterIP — The Default

The apiserver allocates a virtual IP from the Service CIDR range (e.g. 10.96.0.0/12). This IP **exists on no interface anywhere** — purely a rule in iptables/IPVS.

**How a packet gets to a Pod:**
1. Pod sends packet to ClusterIP (e.g. 10.96.0.100:80)
2. kube-proxy's iptables DNAT rule intercepts (before routing): rewrites dst to a real Pod IP
3. Connection tracking table (`conntrack`) remembers the DNAT mapping
4. Return traffic gets reverse-DNAT applied automatically via conntrack
5. Packet routes to selected Pod via CNI

**iptables rule chain for a 3-endpoint Service:**
```
PREROUTING → KUBE-SERVICES (match ClusterIP) → KUBE-SVC-XXXX:
  Rule 1: 33.3% probability → DNAT to 10.244.1.5:port
  Rule 2: 50.0% probability → DNAT to 10.244.2.7:port   (= 1/2 of remaining 66.7%)
  Rule 3: 100% probability  → DNAT to 10.244.3.2:port   (remainder)
Each endpoint gets ~1/3 of traffic
```

## 9. NodePort

Extends ClusterIP by also opening a port on **every node's external IP** (range 30000–32767).

Traffic to `<any-node-IP>:<nodePort>` → DNAT → random healthy Pod (possibly on a different node).
- Reachable from outside the cluster without a cloud load balancer
- Needs you to know a node's IP and the assigned port

**`externalTrafficPolicy: Local` — critical production concept:**

Default (`Cluster`): NodePort packet arriving at Node 1 may be forwarded to a Pod on Node 2 → extra network hop + **SNAT** (Node 1 masquerades packet as its own IP so return traffic flows back through it → Pod sees Node 1's IP, not the real client IP).

`Local`: kube-proxy only forwards to Pods on the **same node**. No extra hop, no SNAT → Pod sees real client IP. But: if no local Pod exists, connection is dropped. Uneven load distribution if nodes have different Pod counts.

## 10. LoadBalancer

Extends NodePort by provisioning a cloud load balancer automatically (via the cloud controller manager). LB's external IP written back to `status.loadBalancer.ingress`.

Traffic: external client → cloud LB → node NodePort → DNAT to Pod.

## 11. ExternalName

No ClusterIP, no iptables rules. CoreDNS returns a CNAME record pointing at an external hostname. Gives an external dependency a stable in-cluster DNS name.

## 12. Headless Service (`clusterIP: None`)

No ClusterIP assigned. DNS queries return individual Pod IPs directly instead of a virtual IP.

Used by StatefulSets — each Pod gets a stable DNS name:
`pod-0.service.namespace.svc.cluster.local` → Pod-0's actual IP

Critical because StatefulSet replicas are not interchangeable (database primary vs replica, etc.) — clients need to address specific Pods, not get a random one from a VIP.

## 13. SessionAffinity

`spec.sessionAffinity: ClientIP` — all connections from the same client IP always go to the same Pod. Implemented via iptables `recent` module or IPVS source-hash mode. Useful when the application has server-side session state.

---

## PART C: CoreDNS — Service Discovery

## 14. The DNS Contract

Every Pod's `/etc/resolv.conf` is written by kubelet at Pod creation:

```
nameserver 10.96.0.10          ← CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Short names get the search domains appended in order:
- `mysql` → tries `mysql.default.svc.cluster.local` → if not found, tries `mysql.svc.cluster.local` → etc.

## 15. What CoreDNS Does

CoreDNS runs as a **Deployment** (typically 2 replicas) behind a Service with a well-known ClusterIP (10.96.0.10, the `--cluster-dns` kubelet flag). It watches apiserver for Service and Endpoint changes via the `kubernetes` plugin and maintains an in-memory DNS record set.

## 16. DNS Records CoreDNS Creates

**For a normal Service (`mysql` in `default` namespace):**
```
mysql.default.svc.cluster.local.    A    10.96.0.100   ← ClusterIP
_mysql._tcp.mysql.default.svc.cluster.local.  SRV  0 100 3306 mysql.default.svc.cluster.local.
```

**For a headless Service:**
```
mysql.default.svc.cluster.local.    A    10.244.1.5    ← Pod 0 IP
mysql.default.svc.cluster.local.    A    10.244.2.7    ← Pod 1 IP
mysql.default.svc.cluster.local.    A    10.244.3.2    ← Pod 2 IP
(multiple A records, client picks)
```

**For a StatefulSet Pod:**
```
mysql-0.mysql.default.svc.cluster.local.   A   10.244.1.5   ← specific Pod IP
mysql-1.mysql.default.svc.cluster.local.   A   10.244.2.7
```

## 17. DNS Resolution Flow

Pod queries `mysql`:
1. libc appends first search domain → `mysql.default.svc.cluster.local`
2. UDP DNS query to 10.96.0.10 (CoreDNS ClusterIP)
3. kube-proxy iptables DNAT: 10.96.0.10 → one of the CoreDNS Pod IPs
4. CoreDNS looks up in-memory record → returns ClusterIP for `mysql`
5. Pod now has the ClusterIP, sends traffic to it
6. kube-proxy DNAT: ClusterIP → a backing mysql Pod IP

## 18. The ndots:5 Gotcha — Real Production Latency Source

**`ndots:5` rule:** If a query name has fewer than 5 dots, the resolver tries appending each search domain first before treating it as an absolute name.

**External hostname query:** `api.example.com` (3 dots, fewer than 5):
1. Try `api.example.com.default.svc.cluster.local` → NXDOMAIN (fail)
2. Try `api.example.com.svc.cluster.local` → NXDOMAIN (fail)
3. Try `api.example.com.cluster.local` → NXDOMAIN (fail)
4. Try `api.example.com.` (absolute) → success

**3 extra DNS round-trips** per external hostname, per new connection. In latency-sensitive services making external HTTP calls, this adds measurable overhead (10–60ms per call depending on DNS latency).

**Fixes:**
- Append trailing dot: `api.example.com.` (treated as absolute, skips search)
- Reduce `ndots` in Pod's `dnsConfig`:
  ```yaml
  spec:
    dnsConfig:
      options:
      - name: ndots
        value: "1"
  ```
- `dnsPolicy: None` with fully custom `dnsConfig`

## 19. CoreDNS Caching

CoreDNS caches responses (default 30s TTL for positive, shorter for negative). **Negative caching** (NXDOMAIN for non-existent names) means a failed lookup stays cached for a few seconds — there's a brief window after creating a new Service where DNS still returns NXDOMAIN. Not a bug, expected behavior — wait a few seconds.

---

## PART D: Ingress — External HTTP/HTTPS Routing

## 20. Why Ingress Exists

A LoadBalancer Service provisions one cloud LB per Service — expensive and unwieldy at scale (20 Services = 20 LBs, 20 external IPs). Ingress provides a **single entry point** routing to multiple Services based on HTTP host/path rules, with a single external LB in front.

## 21. The Two-Part Model

**Ingress resource** (`kind: Ingress`) — a Kubernetes API object declaring routing rules. Does nothing by itself — just configuration in etcd.

**Ingress controller** — a running component (Pod, usually a Deployment) that watches Ingress objects and actually implements the routing. Common controllers:
- nginx Ingress controller (configures nginx)
- Traefik
- AWS Load Balancer Controller (provisions ALB)
- GKE Ingress controller (provisions Google Cloud LB)
- HAProxy Ingress

**Critical:** An Ingress resource without a running Ingress controller is inert. "Why is my Ingress returning 404/not working" often means no controller is deployed, or the wrong `ingressClassName` is set.

## 22. How nginx Ingress Controller Works

1. Watches apiserver for Ingress objects
2. On create/update: generates `nginx.conf` from Ingress rules
3. Signals nginx to reload (`nginx -s reload`) — graceful: new connections use new config, in-flight requests drain on old workers
4. nginx handles actual HTTP proxying, TLS termination, upstream (Service backend) load balancing

The controller itself is exposed via a Service of type LoadBalancer — one cloud LB, one external IP for the whole cluster.

**Traffic flow:**
```
Client → cloud LB → Ingress controller Pod (nginx) → Service ClusterIP → backing Pod
```

## 23. Ingress Resource Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx            # which controller handles this
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret       # Secret of type kubernetes.io/tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix             # Prefix, Exact, or ImplementationSpecific
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

**pathType values:**
- `Exact` — exact path only (`/api` matches only `/api`, not `/api/v2`)
- `Prefix` — path segment prefix (`/api` matches `/api`, `/api/users`, `/api/v2/things`)
- `ImplementationSpecific` — controller defines semantics

**Annotations:** Controller-specific behavior goes in annotations (each controller has its own namespace prefix, e.g., `nginx.ingress.kubernetes.io/`). Common nginx annotations:
- `rewrite-target` — rewrite the path before forwarding
- `proxy-connect-timeout` / `proxy-read-timeout`
- `rate-limit` — request rate limiting
- `auth-url` — external authentication service
- `ssl-redirect` — force HTTPS

## 24. TLS Termination

Ingress controller handles TLS — client's HTTPS terminates at nginx (or the cloud LB), and the controller forwards plain HTTP to the backend Service (unless backend HTTPS is explicitly configured).

TLS cert stored as a Secret of type `kubernetes.io/tls`:
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: app-tls-secret
data:
  tls.crt: <base64-encoded certificate>
  tls.key: <base64-encoded private key>
```

**cert-manager** automates cert provisioning and renewal (from Let's Encrypt or internal CA) by watching Ingress objects and creating/updating Secrets automatically.

## 25. Gateway API — The Successor

**Gateway API** (`gateway.networking.k8s.io`) is the newer (GA in K8s 1.31), more expressive replacement for Ingress:
- `GatewayClass` — the controller type
- `Gateway` — the listener/LB
- `HTTPRoute` / `TCPRoute` — routing rules

Advantages over Ingress:
- Supports TCP/UDP routing natively (Ingress is HTTP-only)
- Traffic splitting (canary deployments) in spec, not annotations
- Header-based routing in spec
- Cleaner separation of concerns (infrastructure team owns Gateway, app team owns Routes)

Most major controllers now support it. Worth knowing exists; Ingress still dominates production today.

---

## 26. Full External Request Trace

Browser requests `https://app.example.com/api/users`:

1. **DNS:** Browser resolves `app.example.com` → public DNS → returns cloud LB external IP
2. **TLS:** Browser opens TLS connection to LB IP → TLS terminates at cloud LB or Ingress controller Pod (depending on config)
3. **LB → Controller:** Cloud LB forwards HTTP request to one of the Ingress controller Pods (via NodePort or direct Pod targeting)
4. **nginx routing:** Controller matches `host: app.example.com`, `path: /api` → selects backend `api-svc`
5. **nginx → Service:** nginx opens connection to `api-svc`'s ClusterIP
6. **DNAT:** kube-proxy iptables/IPVS rewrites destination: ClusterIP → one of the api Pod IPs
7. **CNI:** If selected Pod is on a different node → VXLAN encapsulation carries packet across
8. **Pod:** Packet arrives at api Pod's `eth0`, processed by the application
9. **Response:** Returns via the same path in reverse

---

## Retention Questions and Answers

**Q1: What is the one hard requirement every CNI plugin must satisfy, and why does it matter for application code?**

Any Pod can reach any other Pod directly by its IP address, **without NAT**. The receiving Pod sees the sender's actual Pod IP as the source. This matters for application code because: logging and tracing by source IP works correctly (no IP rewriting to debug around), services can use the client IP for auth/rate-limiting/routing decisions, and distributed tracing across service boundaries works naturally. If NAT were involved, every service would see a node IP instead of the originating service's Pod IP, breaking IP-based access control and making distributed debugging significantly harder.

**Q2: Walk through what happens at the kernel level when Pod A on Node 1 sends a packet to Pod B on Node 2 in VXLAN mode — every hop.**

1. Pod A sends to 10.244.2.7 → leaves `eth0` → crosses veth pair into Node 1 root namespace
2. Node 1 routing table: `10.244.2.0/24 via flannel.1` → packet routed to VTEP device
3. VXLAN kernel driver on `flannel.1`: encapsulates packet. Original packet becomes UDP payload. Outer header: src = Node 1 IP, dst = Node 2 IP, UDP port 4789
4. Outer packet leaves Node 1's `eth0` → physical network forwards it (sees normal IP packet)
5. Arrives at Node 2's `eth0` → kernel sees UDP/4789 → VXLAN driver strips encapsulation → recovers original packet (src 10.244.1.5, dst 10.244.2.7)
6. Node 2 routing: `10.244.2.0/24 via cni0` → packet goes to Linux bridge
7. Bridge does L2 lookup → forwards to Pod B's veth end → delivered into Pod B's network namespace
8. Pod B receives packet with source IP = 10.244.1.5. No NAT anywhere. Requirement satisfied.

**Q3: A ClusterIP is not assigned to any interface on any node. How does a packet destined for it ever reach a real Pod?**

The ClusterIP is a virtual IP that exists only as iptables/IPVS rules programmed by kube-proxy. When a packet is sent to the ClusterIP, the kernel's netfilter subsystem intercepts it in the PREROUTING chain (before routing) and applies a DNAT rule, rewriting the destination to a real backing Pod IP. The DNAT rule is programmed by kube-proxy when it processes the Service's Endpoints object. The connection tracking table (`conntrack`) records this rewrite so return traffic gets the reverse-DNAT applied automatically. The packet is then routed normally to the selected Pod IP via CNI routing. The ClusterIP never needs to exist on an interface — it just needs to match a DNAT rule.

**Q4: What is externalTrafficPolicy: Local, what problem does it solve, and what does it break?**

Default (`Cluster`): a NodePort packet arriving at Node 1 may be DNAT'd to a Pod on Node 2 — adding an extra network hop. Because return traffic must flow back through Node 1, Node 1 **SNATs** the packet (rewrites source IP to its own IP). The backend Pod sees Node 1's IP as the client, not the real client's IP. This breaks IP-based access control, real client IP logging, and geo-routing.

`Local`: kube-proxy only DNAT's to Pods on the same node. No extra hop, no SNAT — backend Pod sees the real client IP. Solves the source IP preservation problem.

What it breaks: if a node has no local Pod matching the Service, connections to that node's NodePort are dropped. Load distribution becomes uneven (nodes with more Pods handle more traffic). Health checking must account for nodes with zero local Pods. Also adds complexity to external LB health-check configuration.

**Q5: A service makes external HTTP calls and intermittently adds ~60ms of latency to the first request to any new external hostname. No network issues detected. What's your hypothesis?**

The **ndots:5 DNS problem.** The Pod's `/etc/resolv.conf` has `ndots:5`. External hostnames like `api.example.com` have 3 dots — fewer than 5 — so the resolver appends all search domains first before trying the name as-is. For each new hostname, the Pod makes 3 failing DNS queries (NXDOMAIN for `api.example.com.default.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local`) before finally resolving `api.example.com.` directly. Each round-trip to CoreDNS adds latency. 3 extra DNS queries × ~20ms each = ~60ms of overhead before the first real request can be sent. Fix: set `dnsConfig.options.ndots: 1` on the Pod spec, or append trailing dots to external FQDNs.

**Q6: An Ingress resource was created 10 minutes ago but traffic is still returning 404. The Service and Pods behind it are healthy. What do you check first?**

Whether an **Ingress controller is actually running** in the cluster. An Ingress resource without a running controller does nothing — it's just configuration in etcd. Check: `kubectl get pods -n ingress-nginx` (or whichever controller namespace). Second, check the `ingressClassName` field — it must match the controller's `IngressClass` name. Third, check the Ingress controller's own logs for errors processing this specific Ingress object. Fourth, verify the `host` field matches exactly what's being requested (case-sensitive). Fifth, if using annotations, verify they're spelled correctly (a typo in an annotation is silently ignored, not rejected).

**Q7: What's the difference between a headless Service and a normal ClusterIP Service at the DNS level, and what is that difference used for?**

**Normal ClusterIP Service:** DNS returns a single A record pointing at the virtual ClusterIP. Clients connect to the VIP; kube-proxy DNAT's to a random backend Pod. Client doesn't know which Pod it's talking to.

**Headless Service** (`clusterIP: None`): No ClusterIP allocated. DNS returns multiple A records — one per backing Pod IP. The client receives all Pod IPs and connects directly (client-side load balancing, or client picks one deliberately). Used for:
- **StatefulSets:** Each pod gets a stable DNS name (`pod-0.service.namespace.svc.cluster.local`) mapping to that specific Pod's IP. Clients can address specific replicas (critical for databases where primary and replica are not interchangeable).
- **Service discovery with client-side LB:** gRPC services often prefer this (gRPC multiplexes over one HTTP/2 connection, so a ClusterIP VIP with single-connection DNAT doesn't distribute load properly).

**Q8: Explain why a StatefulSet uses headless Services while a Deployment typically uses ClusterIP Services.**

A Deployment's Pods are interchangeable — any replica can handle any request, so random load balancing via a ClusterIP VIP works perfectly. You never need to address a specific replica.

A StatefulSet's Pods are **not interchangeable** — a Kafka topic has specific partition leaders on specific brokers; a PostgreSQL cluster has one primary and N replicas that serve different roles; a Cassandra ring uses consistent hashing to route specific keys to specific nodes. Clients need to be able to address `kafka-broker-2` specifically, not just "any kafka broker." The headless Service + stable DNS name pattern (`kafka-broker-2.kafka.default.svc.cluster.local`) provides this: the DNS name is stable (persists across Pod restarts) and maps directly to that specific Pod's IP.
