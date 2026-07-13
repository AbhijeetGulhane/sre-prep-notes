# Kubernetes RBAC: Role vs ClusterRole, RoleBinding vs ClusterRoleBinding, ServiceAccount

> K8s core objects deep dive, continued from NetworkPolicy. Includes live minikube lab result.

## The three-part model

1. **Subject** — who — `ServiceAccount` (Pods/apps), `User`, or `Group` (Users/Groups aren't K8s objects — come from the auth provider: cloud IAM, OIDC, client cert CN)
2. **Role / ClusterRole** — what — permissions: verbs (`get`, `list`, `watch`, `create`, `delete`) on resources (`pods`, `configmaps`, `secrets`)
3. **RoleBinding / ClusterRoleBinding** — the connection — grants a Role/ClusterRole's permissions to a subject

Nothing works alone — a Role with no binding grants nothing; a subject with no binding has zero permissions. **RBAC is deny-by-default**, same philosophy as NetworkPolicy but for the API server instead of the network.

## ServiceAccount — identity for things inside the cluster

How a **Pod** authenticates to the API server, distinct from `User` (humans/external systems). Every namespace gets a `default` ServiceAccount automatically; every Pod uses it unless `serviceAccountName` is set explicitly.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
```
```yaml
spec:
  serviceAccountName: prometheus-sa   # on the Pod
```

Mechanically: kubelet mounts a token into the Pod's filesystem (`/var/run/secrets/kubernetes.io/serviceaccount/token`). Modern K8s (~1.22+) uses short-lived, auto-rotating **projected tokens** via the TokenRequest API rather than old long-lived Secret-backed tokens — fixes the leak/exfiltration risk of static SA tokens if a Pod is compromised.

**One-liner:** "ServiceAccount is identity for workloads, not humans — every Pod authenticates to the API server as some ServiceAccount, defaulting to `default` if unspecified, and what that identity can actually do is entirely determined by whatever RBAC bindings reference it."

## Role vs ClusterRole — scope is the entire distinction

**Role** — scoped to a single namespace; lives in that namespace; permissions only apply there.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: search-api
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**ClusterRole** — can apply cluster-wide, OR grants access to cluster-scoped resources with no namespace at all (Nodes, PersistentVolumes, Namespaces, ClusterRoles/ClusterRoleBindings themselves).
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-cluster
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**Key subtlety:** a ClusterRole's definition isn't inherently "grants everywhere" — depends entirely on HOW it's bound:
- ClusterRoleBinding → applies everywhere
- RoleBinding (yes, RoleBinding can reference a ClusterRole) → applies only in that one namespace

**Why ClusterRoles exist even for namespaced resources like Pods: reusability.** Define the permission set once, bind per-namespace via RoleBindings wherever needed, instead of copy-pasting an identical Role into every namespace. Must use ClusterRole (not Role) for genuinely cluster-scoped resources — a namespaced Role literally can't reference something with no namespace.

## RoleBinding vs ClusterRoleBinding

**RoleBinding** — grants the referenced Role or ClusterRole, scoped to the RoleBinding's OWN namespace.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-search-api
  namespace: search-api
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring        # subject can live in a different namespace than the binding
roleRef:
  kind: ClusterRole             # referencing a ClusterRole here still scopes it to search-api only
  name: pod-reader-cluster
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding** — grants a ClusterRole cluster-wide, no scoping.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods-everywhere
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-reader-cluster
  apiGroup: rbac.authorization.k8s.io
```

**Valid combinations:**

| Binding type | → Role | → ClusterRole |
|---|---|---|
| RoleBinding | Grants Role's perms, scoped to binding's namespace | Grants ClusterRole's perms, but scoped ONLY to binding's namespace |
| ClusterRoleBinding | Invalid — can't reference a namespaced Role | Grants ClusterRole's perms, cluster-wide |

The RoleBinding→ClusterRole cell is the non-obvious one — it's the exact mechanism proven in the lab below.

**One-liner:** "A RoleBinding always scopes permissions to its own namespace, no matter whether it points at a Role or a ClusterRole — it's the binding type, not the role type, that determines whether access is namespace-scoped or cluster-wide."

## Minimal RBAC for Prometheus to scrape pods across namespaces

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-scraper
rules:
- apiGroups: [""]
  resources: ["nodes", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes/metrics"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-scraper-binding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-scraper
  apiGroup: rbac.authorization.k8s.io
```

**Why each piece is forced, not stylistic:**
- **ServiceAccount** — Prometheus is a Pod, needs workload identity
- **ClusterRole not Role** — `nodes` is cluster-scoped (namespaced Role can't reference it); Pods/Services need reading across ALL namespaces
- **ClusterRoleBinding not RoleBinding** — a RoleBinding, even referencing this exact ClusterRole, would only grant access in its own namespace — would need one RoleBinding per namespace, defeating cluster-wide scraping

**Verbs chosen matter:** `get`, `list`, `watch` only — never `create`/`update`/`delete`/`patch`. Principle of least privilege — a compromised Prometheus Pod with write access becomes a cluster-wide attack surface for no operational reason, since scraping never mutates anything.

## Lab: verified with `kubectl auth can-i` (minikube)

**Setup:** `monitoring` namespace + `prometheus-sa` ServiceAccount, `prometheus-scraper` ClusterRole (get/list/watch on nodes/services/endpoints/pods), bound via ClusterRoleBinding.

**Read access confirmed (all `yes`):**
```
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:prometheus-sa -A          → yes
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:prometheus-sa -n search-api → yes  (cross-namespace, proves ClusterRoleBinding scope)
kubectl auth can-i get nodes --as=system:serviceaccount:monitoring:prometheus-sa              → yes
```

**Write access denied (least privilege enforced, both `no`):**
```
kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:prometheus-sa -A          → no
kubectl auth can-i create pods --as=system:serviceaccount:monitoring:prometheus-sa -n search-api → no
```

**Bonus — RoleBinding collapses a ClusterRole's scope to one namespace:**
Deleted the ClusterRoleBinding, replaced with a RoleBinding (same ClusterRole `prometheus-scraper`) in namespace `search-api` only:
```
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:prometheus-sa -n search-api  → yes  (bound here)
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:prometheus-sa -n other-team  → no   (not bound here)
```
Same ClusterRole, same permission definition — only the binding type changed, and that alone flipped cross-namespace access off. This is the RoleBinding→ClusterRole table cell proven live.

## Interview one-liners, compiled

- "RBAC is deny-by-default, same philosophy as NetworkPolicy but for the API server instead of the network — no binding means no access, regardless of what Roles exist."
- "ServiceAccount is identity for workloads, not humans."
- "A RoleBinding always scopes to its own namespace regardless of whether it references a Role or a ClusterRole — the binding type determines scope, not the role type."
- "ClusterRoles exist for reusability and for cluster-scoped resources — define once, bind per-namespace via RoleBinding wherever needed, or cluster-wide via ClusterRoleBinding when you genuinely need everywhere."
- "Prometheus needs a ClusterRole (Nodes are cluster-scoped, and Pods must be readable across all namespaces) bound via a ClusterRoleBinding (a RoleBinding would only cover one namespace) — with read-only verbs, since scraping never needs to mutate anything."

---
*Next up: kube-proxy iptables/ClusterIP implementation, CoreDNS service-name resolution, and pod-to-pod routing across nodes — connected to `ss -tnp` output for Guide Day 6.*
