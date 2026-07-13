# Kubernetes NetworkPolicy: Default Allow-All, Ingress vs Egress, Label Selectors

> K8s core objects deep dive, continued from Ingress. Includes a live minikube lab result.

## Default behavior — open by default, isolation is opt-in

**By default, K8s networking is flat and fully open.** Any Pod can reach any other Pod, across any namespace, on any port — no NetworkPolicy objects exist by default, and with none present, zero restriction applies. Opposite of traditional network segmentation thinking (isolation default, openness opt-in). K8s: **openness default, isolation opt-in.**

**Why:** K8s assumes the CNI plugin, not core K8s, handles enforcement; early design prioritized "just works" over "locked down by default." Operators must explicitly add restriction.

**Prerequisite everyone forgets:** NetworkPolicy only does anything if the CNI plugin implements enforcement. **Flannel does NOT support NetworkPolicy** — YAML applies with zero effect, silently. Calico, Cilium, Weave do support it. Classic interview gotcha: "policy written, traffic still not blocked" is usually "CNI doesn't enforce NetworkPolicy," not a YAML bug.

## The additive/allowlist model — the core mechanical fact

Once ANY NetworkPolicy selects a Pod (for a given direction), that Pod's traffic in that direction switches from allow-all to **deny-everything-except-explicitly-allowed**.

- No policy selects the Pod → all traffic allowed (default)
- ≥1 policy selects the Pod for `ingress` → all inbound denied except what that policy (or any other policy also selecting this Pod for ingress) explicitly allows
- Same logic independently for `egress`

**Multiple policies on the same Pod are additive (OR'd), never subtractive.** No "deny" rule can override another policy's "allow." To exclude specific traffic you must never allow it in the first place via selector design — can't patch it out after.

**One-liner:** "NetworkPolicy is allowlist-only and additive — the moment any policy selects a Pod for a direction, that direction goes from fully-open to default-deny-except-explicitly-allowed, and stacking more policies can only add permitted traffic, never subtract it."

## Ingress vs Egress

Direction is from the *selected Pod's* perspective:
- **Ingress** — what's allowed IN to the selected Pod(s) — "who can talk to me"
- **Egress** — what the selected Pod(s) can send OUT — "who can I talk to"

Independent of each other — a Pod can have an ingress policy with no egress policy (still fully open outbound), or vice versa, or both, or neither.

## Label selectors — three mechanisms

1. **`podSelector`** — matches Pods by label, within the same namespace as the NetworkPolicy object itself
2. **`namespaceSelector`** — matches by namespace labels ("any pod in any namespace labeled `team: platform`")
3. **`ipBlock`** — raw CIDR range, for traffic to/from outside the cluster — only mechanism independent of K8s labels

Combining within one `from`/`to` entry = logical AND ("this label, but only in that namespace"). Multiple separate entries in the same list = OR'd together.

## YAML: restrict a pod to same-namespace-only ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-to-namespace
  namespace: search-api
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: search-api   # built-in label every namespace gets since K8s 1.21+
```

**Subtlety:** an empty `podSelector: {}` inside a `from` block, with NO `namespaceSelector`, means "any Pod in the same namespace as this NetworkPolicy" — namespace-scoped by default. People assume `{}` means "any pod anywhere" — it doesn't; it inherits the policy object's own namespace. The explicit `namespaceSelector` version above is safer/self-documenting against that misunderstanding.

## Default-deny-all baseline pattern (production standard)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: search-api
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  # no rules specified = nothing allowed in either direction
```

Layer specific allows on top (namespace-restriction, "allow from Ingress Controller namespace on port 8080," "allow egress to DNS :53 and external payment API"). This is the real production pattern for security-sensitive namespaces (banking/payments): **explicit default-deny, then allowlist exactly what's needed.**

## Lab: watched the allow-all → deny flip live (minikube + Calico)

Flannel doesn't enforce NetworkPolicy, so switched CNI first: `minikube delete && minikube start --cni=calico`.

**Setup:** `search-api` namespace with `test-pod` (nginx) + `curler` (same namespace); `other-team` namespace with `curler-outside`.

**Before any policy applied:**
```
same-namespace: 200
other-namespace: 200
```
Confirms default allow-all.

**After applying `restrict-to-namespace` (namespaceSelector matching `search-api`):**
```
same-namespace: 200
other-namespace: 000   (curl exit code 28 — timeout)
```
`curler` in the same namespace needed no explicit allow rule — the `namespaceSelector` covered it. `curler-outside` in `other-team` got blocked immediately upon the policy being applied, no other changes made. This is the additive/allowlist model proven live: one policy flips the Pod from open to restricted, and same-namespace traffic passes because it matches the one allow rule that exists.

**Extra verification worth doing:** `kubectl delete networkpolicy restrict-to-namespace -n search-api` then re-curl — should flip back to `200`, confirming the policy (not something else) was the actual enforcement mechanism.

## Interview one-liners, compiled

- "Kubernetes networking is flat and open by default — NetworkPolicy is opt-in isolation, not opt-out."
- "NetworkPolicy only works if your CNI implements it — Flannel doesn't, Calico and Cilium do."
- "The moment a policy selects a Pod for a direction, that direction flips from allow-all to deny-except-explicitly-allowed, and multiple policies only add to the allowlist, never subtract."
- "An empty podSelector in a `from` block means 'same namespace as this policy,' not 'anywhere' — you need an explicit namespaceSelector to broaden that."

---
*Next up: K8s RBAC (Role vs ClusterRole, RoleBinding vs ClusterRoleBinding, ServiceAccount) — minimal RBAC for Prometheus to scrape pods across namespaces.*
