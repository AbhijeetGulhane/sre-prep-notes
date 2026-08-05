# DDIA Ch 6 — Replication

> 2nd edition numbering. Read for **tradeoffs and failure modes**; skim replication-log formats and exact conflict-resolution algorithms.
> Interview level: *"Why async replication here, and what breaks when the leader dies?"*

---

## Why replicate at all

Three reasons, and they pull in different directions: keep data **geographically close** to users (latency), keep the system **working when a node fails** (availability), and **scale out reads** (throughput). Every replication decision is a tradeoff between these and consistency.

The hard part is never the happy path — it's what happens to in-flight and lagging data **when something fails**. That's what interviewers probe.

---

## The three approaches

### Single-leader (master-slave)
One node takes all writes, streams a replication log to followers, reads can go to any replica.

- **Verdict:** the default. Simple, no write conflicts, used by Postgres, MySQL, most RDBMS, MongoDB.
- **Cost:** the leader is a write bottleneck and a failover risk.

### Multi-leader
Multiple nodes accept writes (e.g. one leader per datacenter), replicate to each other.

- **Verdict:** use only when you genuinely need multi-region writes or offline-capable clients.
- **Cost:** write conflicts are now possible and you must resolve them. This is the expensive part.

### Leaderless (Dynamo-style)
Client writes to several replicas directly; reads query several and reconcile. Quorum-based.

- **Verdict:** high availability, no failover step. Cassandra, DynamoDB, Riak.
- **Cost:** you manage consistency yourself via quorums; reads can return stale or conflicting values.

---

## Sync vs async replication — the single most important tradeoff

| | Synchronous | Asynchronous |
|---|---|---|
| Leader waits for follower ack? | Yes | No |
| Data loss on leader failure? | No | **Yes — unacked writes are lost** |
| Write latency | Higher | Lower |
| Availability if follower slow/down | Leader blocks | Unaffected |

**The real-world answer is usually semi-synchronous:** one follower synchronous, the rest async. You get a guaranteed second copy without blocking on every replica.

**Interview trap:** if someone says "we use async replication," the immediate follow-up is *"so what happens to writes the leader accepted but hadn't shipped yet when it crashed?"* → they're lost, and the client thinks they succeeded. This is the crux of replication lag reasoning.

---

## Replication lag — the consistency problems it creates

Async replication means followers are behind. Three anomalies and their fixes:

1. **Read-your-own-writes** — user posts a comment, refreshes, it's gone (read hit a lagging replica). Fix: read from leader for the user's own recent writes, or track write timestamp.
2. **Monotonic reads** — user sees a comment, refreshes, it vanishes (second read hit a further-behind replica). Fix: pin each user to one replica.
3. **Consistent prefix reads** — you see an answer before the question (writes applied out of causal order). Fix: causal ordering / write to same partition.

These three are worth memorising by name — they're clean interview answers that show you understand eventual consistency concretely, not just as a buzzword.

---

## Failover (single-leader) — what actually goes wrong

When the leader dies, a follower is promoted. Things that break:

- **Lost writes** — async writes not yet replicated are gone. If that data was used elsewhere (e.g. an auto-increment ID reused), you get corruption. Real incident: GitHub, 2012 — promoted follower was behind, IDs got reused, leaked private data.
- **Split brain** — two nodes both think they're leader, both accept writes. Needs a fencing mechanism.
- **Timeout tuning** — declare the leader dead too fast → needless failovers under load; too slow → long outages.

**Interview one-liner:** *"Failover isn't free — the failure modes are lost unreplicated writes, split brain, and false positives from timeout tuning. That's why automatic failover is genuinely risky."*

---

## Multi-leader conflict resolution (SKIM the mechanics, know it exists)

If two leaders accept conflicting writes, you need a resolution strategy:
- **Last-write-wins (LWW)** — simple, lossy, clocks lie. Cassandra's default.
- **Version vectors** — track causality, detect true conflicts, let app merge.

You need to know LWW loses data and that version vectors exist. You do **not** need to implement version vectors for an SRE loop.

---

## Leaderless quorums (the one formula worth knowing)

**w + r > n** guarantees a read overlaps at least one up-to-date write.
- n = replicas, w = nodes that must ack a write, r = nodes read from.
- Common: n=3, w=2, r=2.
- Tune w down for write availability, r down for read latency — but if w + r ≤ n you can read stale data.

**Read repair** (fix stale replicas on read) and **anti-entropy** (background sync) keep replicas converging. Know the terms; skim the mechanics.

---

## Interview-ready summary

- Single-leader is the default; reach for multi/leaderless only with a concrete reason.
- Sync vs async is the load-bearing tradeoff: async = lower latency, real data-loss window on failure.
- Replication lag → read-your-writes / monotonic / consistent-prefix (know all three by name).
- Failover risks: lost writes, split brain, timeout false-positives.
- Quorum rule: w + r > n.

---

## Reading resources

- **DDIA 2nd ed, Ch 6** — primary. Read the sync/async and lag sections properly; skim log-shipping formats. (paid)
- **"Notes on Distributed Systems for Young Bloods"** — Jeff Hodges, free — excellent complement on why failure handling dominates. (free)
- **Jepsen analyses** (jepsen.io) — free — real-world consistency violations in named databases; great for interview war stories. Read one or two, don't binge. (free)
- Skip for now: Werner Vogels' original Dynamo paper unless you want leaderless depth (supplementary).
