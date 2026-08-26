## Day 12 — NetworkPolicy + RBAC Validation (`k8s/networkpolicy.yaml`)

### Why NetworkPolicy?

Without a NetworkPolicy, Kubernetes allows all pod-to-pod traffic by default —
any pod in any namespace can reach any other pod on any port. This is the
"flat network" model: convenient but a security liability.

A NetworkPolicy restricts which traffic is allowed to reach search-api pods
and which traffic they can send. It implements the principle of least privilege
at the network layer, complementing the RBAC least privilege at the API layer.

**Threat model without NetworkPolicy:**
- A compromised pod in any namespace can reach search-api on port 8000
- A compromised search-api pod can reach any other service in the cluster
- Lateral movement is unrestricted

**Threat model with NetworkPolicy:**
- Only pods in search-sre and monitoring namespaces can reach search-api
- search-api pods can only reach DNS, HTTPS external endpoints, and each other
- Lateral movement from a compromised search-api pod is significantly restricted

### Why policyTypes: [Ingress, Egress]?

Specifying both types means the NetworkPolicy controls both directions:
- Ingress: who can send traffic TO search-api pods
- Egress: where search-api pods can send traffic

If only Ingress is specified, egress remains unrestricted — a compromised pod
could still exfiltrate data or reach internal services. Specifying both gives
complete control over the pod's network posture.

### Ingress Rules — Why These Sources?

**Rule 1: Allow from search-sre namespace (podSelector: {})**
```yaml
- from:
  - podSelector: {}   # empty selector = all pods in this namespace
```
`podSelector: {}` with no `namespaceSelector` means "any pod in the same
namespace as this policy" (search-sre). This allows:
- kubectl port-forward connections (which appear to come from within the namespace)
- Future intra-namespace services that might call search-api

**Rule 2: Allow from monitoring namespace**
```yaml
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
```
Without this rule, Prometheus (in the monitoring namespace) can't reach
search-api pods' /metrics endpoint. The NetworkPolicy would silently drop
Prometheus's scrape requests, causing targets to show as DOWN.

`kubernetes.io/metadata.name` is an automatic label Kubernetes adds to every
namespace — its value equals the namespace name. This is the standard way to
select a namespace by name in a NetworkPolicy.

### Egress Rules — Why These Destinations?

**Rule 1: DNS on port 53**
```yaml
- ports:
  - port: 53
    protocol: UDP
  - port: 53
    protocol: TCP
```
DNS resolution is required for almost everything. Without this:
- The model can't resolve HuggingFace CDN hostnames on first download
- Any external HTTP call fails at DNS resolution
- Even internal K8s service names fail (they resolve via kube-dns)

Both UDP and TCP are needed — DNS uses UDP by default, but falls back to TCP
for responses larger than 512 bytes (common with DNSSEC or large record sets).

**Rule 2: HTTPS on port 443**
```yaml
- ports:
  - port: 443
    protocol: TCP
```
The sentence-transformers model downloads from HuggingFace's CDN over HTTPS
on first pod start. Without this egress rule, the model download would be
blocked and the pod would fail to start.

In a fully air-gapped production environment, the model would be baked into
the Docker image (eliminating the download) and this rule could be removed.

**Rule 3: Intra-namespace egress**
```yaml
- to:
  - podSelector: {}
```
Allows search-api pods to reach other pods in search-sre. Currently no other
services exist here, but this allows future services (a caching layer, a
feature flag service) without requiring NetworkPolicy updates.

### Why Verify with kubectl auth can-i After NetworkPolicy?

NetworkPolicy controls network-level traffic; RBAC controls API-level access.
They're independent — NetworkPolicy doesn't affect what the Prometheus
ServiceAccount can do via the K8s API, and RBAC doesn't affect TCP connections.

Verifying both after each change confirms:
1. Network: Prometheus can still scrape /metrics (targets show UP in /targets)
2. API: Prometheus SA still has list/watch on pods (auth can-i list = yes)
3. API: Prometheus SA still cannot delete pods (auth can-i delete = no)

This dual verification ensures neither the network layer nor the API layer
was accidentally misconfigured by the change.

### Verification Results

```bash
# NetworkPolicy applied
kubectl get networkpolicy -n search-sre
# NAME                  POD-SELECTOR       AGE
# search-api-netpol     app=search-api     Xs

# Prometheus targets still UP after NetworkPolicy (Rule 2 working)
# http://localhost:9090/targets → 3/3 search-api targets UP

# RBAC unchanged by NetworkPolicy (separate control plane)
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:prometheus -n search-sre
# yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:monitoring:prometheus -n search-sre
# no
```
