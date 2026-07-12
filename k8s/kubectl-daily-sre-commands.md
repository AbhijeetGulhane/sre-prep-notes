# kubectl for Daily SRE Use: get, describe, logs, exec, rollout status, rollout undo

> Week 4/5 boundary — K8s core objects, kubectl fluency for debugging.

## kubectl get — "what's the current state"

```bash
kubectl get pods                          # pods in current namespace
kubectl get pods -A                       # -A = --all-namespaces, cluster-wide view
kubectl get pods -o wide                  # adds node, IP, nominated node — first check for scheduling issues
kubectl get pods -w                       # -w = --watch, streams live changes
kubectl get pods -l app=search-api        # -l = label selector
kubectl get pods --field-selector=status.phase=Running   # filter by field, not label
kubectl get deploy,svc,ingress            # multiple resource types in one call
kubectl get pod mypod -o yaml             # full LIVE manifest (includes status/defaults, not just what you applied)
kubectl get pod mypod -o jsonpath='{.status.podIP}'   # extract exactly one field, useful for scripting
```

**Debugging habit:** `kubectl get pods -o wide` is usually step 1 of any incident — tells you if it's one Pod (isolated) vs all Pods of a Deployment (systemic), which node(s), phase, restart count at a glance.

## kubectl describe — "why"

```bash
kubectl describe pod mypod
```

Not the manifest — manifest + **Events** at the bottom, where most debugging value lives (scheduler reasoning, image pull attempts, probe failures, OOMKills — a separate object type the API server correlates in here).

**What to check, top to bottom:**
- `Status` — Running/Pending/CrashLoopBackOff/etc.
- `Containers → State` (+ `Last State` if restarted: exit code + reason — `OOMKilled`, `Error`, `Completed`)
- `Conditions` — `PodScheduled`, `Ready`, `ContainersReady` — `False` here pinpoints which lifecycle stage is stuck
- `Events` (bottom) — chronological: `FailedScheduling` (scheduler rejection reason per node — resources, taint mismatch, affinity conflict), `Pulling`/`Pulled`/`Failed` (image pull issues), `Unhealthy` (probe failures with actual probe output)

**Debugging habit:** if a Pod is `Pending`, `describe` is the ONLY useful command — `logs` gives nothing since the container never started; Events states the scheduler's rejection reason directly.

## kubectl logs — container stdout/stderr

```bash
kubectl logs mypod                        # current container (single-container pod assumed)
kubectl logs mypod -c sidecar             # -c = specify container, required for multi-container pods
kubectl logs mypod --previous             # PREVIOUS instantiation's logs — critical after crash/restart
kubectl logs mypod -f                     # -f = follow, stream live
kubectl logs mypod --since=10m            # last 10 minutes only
kubectl logs mypod --tail=50              # last 50 lines only
kubectl logs -l app=search-api --all-containers=true --prefix=true   # logs across ALL pods matching a label, prefixed by pod name
```

**Debugging habit:** `--previous` is the flag people forget. After an OOMKill/restart, plain `logs` shows the NEW healthy instance — actual crash evidence lives in `--previous`.

## kubectl exec — shell inside a running container

```bash
kubectl exec -it mypod -- bash            # -i interactive, -t TTY → shell
kubectl exec -it mypod -c sidecar -- sh   # -c for multi-container; sh if no bash (alpine/distroless)
kubectl exec mypod -- cat /etc/resolv.conf   # one-off command, no shell — scriptable
kubectl exec -it mypod -- curl -v http://other-service:8080/health   # test connectivity FROM inside the pod's netns
```

**Debugging habit:** distinguishes "the app is broken" from "the network/config around the app is broken" — curl from inside the pod isolates whether the pod's own network path works, independent of the app's own error message.

**Limitation:** doesn't work on a Pod stuck `Pending`/`CrashLoopBackOff` before the container starts (nothing to attach to), or on `distroless` images (no shell binary — deliberate security hardening). Workaround: `kubectl debug` (ephemeral containers).

## kubectl rollout status — watch a deployment progress

```bash
kubectl rollout status deployment/search-api
kubectl rollout status deployment/search-api --timeout=60s
```

Blocks until rollout finishes or timeout hits — the real CI/CD gate for "did the deploy succeed" rather than eyeballing `get pods`.

```bash
kubectl rollout history deployment/search-api                 # list of revisions
kubectl rollout history deployment/search-api --revision=3    # what changed in that revision
```

## kubectl rollout undo — the incident-response command

```bash
kubectl rollout undo deployment/search-api                  # roll back to immediately previous revision
kubectl rollout undo deployment/search-api --to-revision=3  # roll back to a specific numbered revision
```

**Debugging habit:** a bad deploy causing errors is one of the most common incident causes; `rollout undo` is usually faster/safer than hotfixing forward under pressure. Works because Deployment keeps old ReplicaSets around (`revisionHistoryLimit`, default 10) — undo just scales the old ReplicaSet back up and current one down, no rebuild, hence fast.

```bash
kubectl rollout pause deployment/search-api    # freeze a rollout mid-way (e.g. bad canary)
kubectl rollout resume deployment/search-api   # continue it
kubectl rollout restart deployment/search-api  # force new Pods without changing spec — picks up a changed ConfigMap/Secret that Pods don't auto-reload
```

## The realistic debugging flow these compose into

1. `kubectl get pods -o wide` — scope blast radius (one Pod? all? one node?)
2. `kubectl describe pod X` — if Pending/restarting, read Events for the why
3. `kubectl logs X --previous` (if crashed) or `kubectl logs X -f` (if live but misbehaving)
4. `kubectl exec -it X -- <probe command>` — isolate app vs network/config
5. If traced to a recent deploy: `kubectl rollout undo` — stop the bleeding first, root-cause after

**Interview one-liner:**
> "get scopes the blast radius, describe tells you why via Events, logs (with --previous if it crashed) shows what the app actually did, exec lets you probe the pod's own network/filesystem reality directly, and rollout undo is the fastest incident-response lever when a bad deploy is the root cause — because it just re-points traffic at the still-existing old ReplicaSet instead of rebuilding anything."

---
*Next up in this batch: K8s Ingress (NGINX IngressController, path-based routing, TLS termination) — first genuine deep-dive on this topic after earlier shallow exposure.*
