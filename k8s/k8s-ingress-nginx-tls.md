# Kubernetes Ingress: NGINX IngressController, Path-Based Routing, TLS Termination

> K8s core objects, first real deep-dive on Ingress (earlier exposure was shallow/not fully absorbed).

## The problem Ingress solves

Without it: `NodePort` (high port 30000-32767 on every node, no hostnames/paths/TLS) or `LoadBalancer` Services (one cloud LB *per Service* — expensive/wasteful with many microservices). Ingress gives one entry point, L7-aware routing rules, fan-out to many backend Services by hostname/path.

## Ingress resource vs Ingress Controller (the most commonly confused pair)

- **Ingress (resource)** — just YAML. Declares rules ("route `api.example.com/users` to `users-svc`"). Does nothing by itself — inert config in etcd.
- **Ingress Controller** — actual running software (a Pod, watches the API server) that implements those rules. **K8s ships no controller by default** — an Ingress object with no controller running does nothing. Must explicitly deploy one (NGINX Ingress Controller, Traefik, HAProxy, cloud-managed GKE/ALB controllers).

**One-liner:** "The Ingress object is a declaration, not an implementation — it's the Ingress Controller, a separate piece of software you deploy, that actually watches those objects and configures a real proxy to match."

## NGINX Ingress Controller — what it does mechanically

NGINX reverse proxy running as a Pod (Deployment/DaemonSet), fronted by a `LoadBalancer`/`NodePort` Service — the one real external entry point for potentially hundreds of Ingress-routed services.

1. Watch loop against API server for Ingress, Service, Endpoints objects
2. On change, regenerates `nginx.conf` from a template (current routing rules + healthy Pod IPs)
3. Reloads NGINX (`nginx -s reload`) — graceful, doesn't drop in-flight connections

Worth stating: it's a control-plane reconciler that programs a real, ordinary NGINX config — automation on top of a proxy you could hand-configure, not magic.

## Path-based and host-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: search-api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /search
        pathType: Prefix
        backend:
          service:
            name: search-api-svc
            port:
              number: 80
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-svc
            port:
              number: 80
```

**`pathType` (commonly skipped detail):**
- `Exact` — matches URL path exactly, trailing-slash sensitive
- `Prefix` — matches by path *segment* prefix (`/search` matches `/search`, `/search/`, `/search/foo`, NOT `/searchbar` — segment-aware, not raw string prefix)
- `ImplementationSpecific` — behavior depends on the controller (legacy/wildcard matching in NGINX)

**Host-based routing:** multiple `- host:` blocks (one Ingress or several) let one IP/LB serve `api.example.com`, `admin.example.com`, `blog.example.com` differently, using the HTTP `Host` header (or SNI for TLS) to pick the rule set.

## TLS termination

```yaml
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-example-com-tls
```

Referenced `Secret` (type `kubernetes.io/tls`) holds cert + private key. **TLS termination** = client's HTTPS is decrypted AT the Ingress Controller — it holds the private key, completes the handshake, and by default forwards plain HTTP to the backend Pod over the cluster network.

**Tradeoff:**
- **Pro:** backends don't manage their own certs; simpler rotation (often automated via cert-manager watching Ingress, provisioning/renewing Let's Encrypt certs)
- **Con:** Ingress-to-Pod traffic is unencrypted by default — fine if the cluster network is trusted/isolated, NOT fine for compliance needs (PCI, HIPAA-adjacent, banking regulatory requirements). Fix: **TLS passthrough** (Ingress forwards encrypted bytes as-is, Pod does its own termination) or a **service mesh with mTLS** (Istio/Linkerd) for pod-to-pod encryption regardless of Ingress.

**SNI's role:** with multiple `host` entries each with different certs, NGINX uses SNI (hostname announced during the TLS handshake, before encryption starts) to pick the right cert — before it even knows the HTTP path or has decrypted anything. This is why host-based TLS routing works without one cert covering everything.

## Full traffic flow: internet → pod (the "draw from memory" answer)

1. **DNS resolution** — client resolves `api.example.com` → external IP of the Ingress Controller's `LoadBalancer` Service
2. **TCP + TLS handshake** — client connects to IP:443; ClientHello includes SNI; NGINX picks the matching cert, completes handshake — encryption ends here (unless passthrough)
3. **HTTP request arrives at NGINX**, plaintext; reads `Host` header + URL path, matches against generated config
4. **NGINX forwards to a Pod IP directly** — controller watches Endpoints/EndpointSlices itself and load-balances across real Pod IPs, **often bypassing kube-proxy's iptables/IPVS Service abstraction entirely** for this hop (does its own upstream load balancing for lower latency, better Pod-readiness awareness)
5. **Pod receives plain HTTP** on its container port, processes, responds back through the same path

**One-liner:** "DNS gets you to the Ingress Controller's external IP; TLS terminates there using SNI to pick the right cert before any HTTP is visible; NGINX then matches Host+path against its generated config and forwards directly to a Pod IP it's tracking via Endpoints — often bypassing kube-proxy's Service abstraction entirely for that hop."

---
*Next up: K8s NetworkPolicy (default allow-all, ingress vs egress rules, label selectors, namespace-restriction YAML example).*
