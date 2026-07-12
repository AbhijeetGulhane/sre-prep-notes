# Kubernetes Workload Controllers: Deployment, DaemonSet, StatefulSet, Job, CronJob

> Week 4/5 boundary — K8s core objects deep dive.

**Core framing:** every workload controller answers "how many replicas, where, and with what identity/lifecycle guarantees?" Deployment is the default answer; the other four exist because Deployment's answer is wrong for specific workload shapes.

**Ties the whole set together for an interview:**
> "Every controller besides Deployment exists because one of Deployment's three core assumptions — Pods are interchangeable, Pods run forever, and replica count is a number you choose — is wrong for that workload shape." DaemonSet breaks "replica count you choose," StatefulSet breaks "interchangeable," Job/CronJob break "runs forever."

## Deployment (baseline)

Manages a ReplicaSet → N identical, interchangeable, stateless Pods. Random hash-suffixed names, no stable identity, no stable storage, no start/stop ordering guarantees, scheduler places anywhere that fits. Rolling updates replace Pods without caring which goes first. Works when any replica can serve any request and none needs to remember anything.

## DaemonSet

**Guarantee:** exactly one Pod per node (or matching subset via nodeSelector/affinity), automatically — add a node, a Pod schedules there with no action needed; remove a node, its Pod goes too.

**Why Deployment can't do this:** Deployment's replica count is a fixed number decoupled from node count; nothing enforces one-per-node placement. DaemonSet doesn't do bin-packing — it targets "one Pod per eligible node" directly (via injected per-node affinity in modern K8s).

**Use cases:**
- Log collection agents (Fluentd, Filebeat, Fluent Bit)
- Node monitoring agents (Prometheus Node Exporter, Datadog Agent)
- CNI network plugins (Calico, Cilium, Weave) — must run on every node by definition
- Storage daemons needing direct local disk access (Ceph, GlusterFS node plugins)

**Key mechanism:** DaemonSet Pods commonly tolerate `node.kubernetes.io/not-ready` and control-plane taints by default — deliberate, since log/monitoring agents need to run even on cordoned/tainted/control-plane nodes that regular workloads are excluded from.

**One-liner:** "DaemonSet's replica count isn't a number you set — it's implicitly 'one per node,' recalculated automatically as nodes join or leave. Use it for anything that must have local presence on every node: log shippers, monitoring agents, CNI plugins."

## StatefulSet

**Guarantee:** stable, unique network identity + stable, dedicated storage per replica, persisting across restarts/rescheduling.

Three things Deployment doesn't give:
1. **Stable, predictable Pod names** — `mysql-0`, `mysql-1`, `mysql-2` (not random hashes); a rescheduled replica keeps its ordinal name.
2. **Stable network identity** — each Pod gets a stable DNS hostname via a required headless Service (`mysql-0.mysql.default.svc.cluster.local`), addressable individually, not just via a Service VIP.
3. **Stable, per-replica storage** — each replica gets its own PVC from a `volumeClaimTemplate`; rescheduled Pod reattaches to the *same* PVC, data survives.

**Ordering:** by default, strict sequential create/terminate — `mysql-0` Running+Ready before `mysql-1` created; highest-ordinal terminates first on scale-down. Matters for e.g. a replica joining after the primary exists. `podManagementPolicy: Parallel` opts out.

**Use cases:** anything where replicas are NOT interchangeable —
- Databases with primary/replica topology (MySQL, PostgreSQL, MongoDB replica sets)
- Distributed coordination systems needing stable peer identity for quorum/leader election (ZooKeeper, etcd, Kafka)
- Systems bootstrapping via seed node hostnames (Cassandra, Elasticsearch)

**One-liner:** "StatefulSet exists for workloads where 'which specific replica' matters — stable names, stable per-replica storage that follows the replica across rescheduling, and ordered startup/teardown. Deployment Pods are cattle; StatefulSet Pods are pets with numbers on them."

## Job

**Guarantee:** a Pod (or set of Pods) runs to completion, exactly once (or N times), then stops — not a long-running service.

**Why ReplicaSet can't do this:** ReplicaSet enforces "keep N Pods running forever" — a Pod exiting even successfully (code 0) triggers a replacement, causing an infinite restart loop for a task that already succeeded. Job tracks **completions**, not running-replica-count: success stops (no restart); failure retries up to `backoffLimit`.

**Key fields:**
- `completions` — total successful completions needed (default 1)
- `parallelism` — concurrent Pods while working toward that count
- `backoffLimit` — retries allowed before Job marked Failed
- `activeDeadlineSeconds` — hard wall-clock timeout for the whole Job

**Three work-distribution patterns:**
1. Non-parallel (`completions:1, parallelism:1`) — one-off script, simplest case
2. Fixed completion count + parallelism (e.g. `completions:5, parallelism:2`) — process a fixed batch, N at a time
3. Work queue pattern (`completions` unset, `parallelism:N`) — Pods pull from external queue (Redis/RabbitMQ/SQS) until empty, each exits successfully; Job doesn't know total work count upfront

**Use cases:** DB migrations, batch/ETL processing, one-off admin tasks (bulk password reset, cache warming), ML training runs.

**One-liner:** "Job tracks completions, not desired replica count — success terminates the Pod and doesn't restart it, the opposite of Deployment/ReplicaSet behavior on exit code 0. Use it for anything with a defined 'done' state: migrations, batch processing, one-off scripts."

## CronJob

**What it is:** a Job factory, scheduled. Doesn't run Pods directly — creates a new Job object on a cron schedule; that Job then creates Pods exactly like a standalone Job. Thin scheduling wrapper, not a separate execution model.

**Schedule:** standard cron format (`minute hour day-of-month month day-of-week`), e.g. `0 2 * * *` = daily 2 AM. Timezone gotcha: older K8s used the controller-manager's local time (usually UTC) by default; `timeZone` field settable explicitly since ~v1.24/1.27 — worth mentioning as a real production gotcha.

**CronJob-specific fields:**
- `concurrencyPolicy` — behavior if a run is still going when the next scheduled time arrives:
  - `Allow` (default) — overlap, run concurrently
  - `Forbid` — skip new run if previous hasn't finished
  - `Replace` — kill running old Job, start the new one
- `startingDeadlineSeconds` — how far past the scheduled time a "late" run (controller was down) is still allowed to start before that occurrence is abandoned
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` — how many completed Job objects/Pods to retain for debugging before GC (small defaults: 3 and 1) since Jobs/Pods otherwise accumulate in etcd forever

**Use cases:** nightly backups, scheduled report generation, periodic cleanup/GC (expired sessions, log rotation), cache refresh, periodic reconciliation sweeps.

**Concurrency gotcha:** non-idempotent workload (e.g. "charge customer's card") + `concurrencyPolicy: Allow` + a slow run that overruns its interval → double execution. Real production bug pattern. Fix: `Forbid`/`Replace`, or make the task idempotent (better — defense-in-depth against restarts/retries generally, not just CronJob overlap).

**One-liner:** "CronJob doesn't execute anything itself — it's a scheduler that creates Job objects on a cron schedule, and each Job then behaves exactly like a standalone Job. The interesting design decisions are all about overlap handling via concurrencyPolicy, since 'what happens if the previous run is still going' is where production incidents actually come from."

## Summary table

| | Deployment | DaemonSet | StatefulSet | Job | CronJob |
|---|---|---|---|---|---|
| Replica count | Fixed number you set | Implicit: one per node | Fixed number you set | Completions target | N/A (creates Jobs) |
| Pod identity | Random, interchangeable | Random, one per node | Stable, ordinal (`-0`, `-1`) | Random, ephemeral | N/A |
| Storage | Optional, not sticky to a replica | Usually none / hostPath | Stable PVC per replica, follows it | Usually none | N/A |
| Lifecycle | Runs forever, self-heals | Runs forever, one per node | Runs forever, ordered start/stop | Runs to completion, then stops | Schedules Jobs on a cron |
| Restart on exit 0 | Yes (unwanted for batch work) | Yes | Yes | No (this is the point) | N/A |
| Core use case | Stateless web services/APIs | Per-node agents (logging, monitoring, CNI) | Databases, coordination systems (peers matter) | Migrations, batch/ETL, one-off scripts | Scheduled/recurring batch work |

---
*Next: K8s QoS classes / HPA / taints theory (still pending from Sat Jul 11 plan), then iximiuz.com container networking series + "Life of a Packet" KubeCon talk to close out the networking remediation block.*
