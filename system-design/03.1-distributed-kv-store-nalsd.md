# Distributed KV Store — NALSD Design (Google Scale)

*Third in the system-design set. [01-url-shortener](./01-url-shortener-nalsd.md) was storage-bound, [02-rate-limiter](./02-rate-limiter-nalsd.md) was throughput-bound with TTL'd state. This one is the odd one out: it's not read-heavy-with-a-hot-path, it's a general-purpose storage primitive other systems get built on top of — the design question is availability and consistency under partition, not caching or hot-key mitigation.*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- `PUT(key, value) -> version` — write a value.
- `GET(key) -> value(s), version` — read a value. May return multiple conflicting versions ("siblings") if concurrent writes raced — see Section 4.
- `DELETE(key)` — typically implemented as a tombstone write, not a physical delete (see Section 5).
- **No range queries, no joins, no secondary indexes.** This is a deliberate scope cut, not an oversight — a pure KV store optimizes single-key access at massive scale; range/query capability belongs in a layer built on top (or a different store entirely).

### Non-functional requirements — the central design tension
- **Availability over consistency.** This store is explicitly modeled on the Dynamo lineage (Cassandra, Riak, DynamoDB): under a network partition, the system keeps accepting reads and writes on both sides of the split rather than refusing service (an **AP** choice in CAP terms). This is the single decision that shapes everything else in this document — the partitioning scheme, the replication protocol, and the conflict-resolution mechanism all exist specifically to make "stay available during a partition" work without silently losing or corrupting data.
- **Horizontal scalability** — adding nodes should scale both storage and throughput roughly linearly, and should not require a full data reshuffle (this rules out naive `hash(key) % N` sharding — see Section 3).
- **Tunable consistency**, not a fixed guarantee — the same store should be usable both for "I'd rather have stale-but-available data" (session cache) and "I need most writes visible almost immediately" (inventory count) use cases, via a runtime knob rather than two different systems.

### Scale target
Assume this store backs multiple downstream services (sessions, feature flags, shopping carts, metadata) — a shared infrastructure primitive, not one application.

- **Keys:** 1 billion
- **Average value size:** 1 KB
- **Raw data:** 1B × 1KB = **1 TB**
- **With replication factor N=3** (justified in Section 4): **3 TB total** — this is trivial for a modern cluster; storage capacity is not the binding constraint here (more on this contrast in the eventual math-pass doc).
- **Peak ops/sec:** 500,000, with a read-heavy skew typical of this workload class — **400,000 reads/sec, 100,000 writes/sec** (4:1 ratio)
- **Latency target:** p99 under 10ms for both reads and writes at the chosen quorum level (Section 4) — this store sits on the critical path of other services' requests, so it inherits their latency budgets.

---

## 2. API Design

```
PUT /kv/{key}
Request:
{
  "value": "<opaque bytes, base64 or JSON>",
  "context": "<vector clock from a prior GET, if this is an update>"
}
Response: 200 OK
{ "version": "<new vector clock>" }

GET /kv/{key}
Response: 200 OK
{
  "values": [
    { "value": "...", "version": "<vector clock A>" },
    { "value": "...", "version": "<vector clock B>" }   // present only if concurrent writes produced siblings
  ]
}

DELETE /kv/{key}
Response: 200 OK   -- internally a tombstone write, see Section 5
```

Design notes:
- `context`/`version` round-trips a vector clock (Section 4) from the client's last read back into its next write — this is how the store knows whether a write is an *update* to a value it has already seen, or a *concurrent* write racing with another client, without needing a lock.
- Returning multiple `values` on a `GET` is a deliberate design choice, not a bug surface — see conflict resolution in Section 4. A well-behaved client is expected to resolve siblings (e.g., merge a shopping cart's item sets) and write back a single reconciled value.

---

## 3. Partitioning — Consistent Hashing Ring

This is the first crux decision, playing the same role Base62-vs-MD5 played for the shortener and token-bucket-vs-leaky-bucket played for the rate limiter.

### The naive approach: `hash(key) % N`
- Simple, but **catastrophic on cluster resize.** Changing `N` (adding or removing even one node) changes the result of `% N` for nearly every key, forcing a near-total data reshuffle across the cluster every time it scales. At 1TB+ of data, this is operationally unacceptable — you cannot scale a store that has to move ~100% of its data on every capacity change.

### Consistent hashing (the standard fix)
- Map both **nodes** and **keys** onto the same fixed, circular hash space (a "ring") — e.g., a 160-bit space via SHA-1, or a simpler 32-bit space.
- A key belongs to the first node found walking clockwise from the key's position on the ring.
- **When a node is added or removed, only the keys between it and its counter-clockwise neighbor move — on average `1/N` of the total keyspace, not the whole dataset.** This is the entire point: cluster resizing becomes a local, bounded operation instead of a global one.

### The refinement consistent hashing alone doesn't solve: virtual nodes
- With each physical node owning exactly one point on the ring, key distribution across nodes is uneven — some nodes end up owning much larger arcs of the ring than others purely by chance, and a newly added node absorbs one large contiguous chunk of keys from exactly one neighbor (a hot-spot on both the losing and gaining node during rebalance).
- **Fix: each physical node is assigned many points on the ring** (typically 100–256 "virtual nodes"/vnodes), scattered pseudo-randomly. Load balances out statistically across many small arcs instead of a few large ones, and adding a node means taking small slices from *many* existing nodes simultaneously rather than one large slice from one neighbor — smoothing both the steady-state load distribution and the rebalance-time load spike.

**Decision: consistent hashing with virtual nodes (vnodes) is the standard, correct choice** for exactly the same reason Base62-with-a-counter beat MD5 truncation in the shortener doc — it avoids a scaling property (near-total reshuffle on resize) that gets *worse*, not better, as the cluster and dataset grow.

---

## 4. Replication (N) & Quorum (W + R > N)

This is the second, larger crux decision — the one that actually encodes the availability-over-consistency choice from Section 1 into a concrete, tunable protocol.

### Replication factor N
- Each key is stored not just at its ring position's owning node, but at the **next N-1 distinct physical nodes walking clockwise** (its "preference list"). This list, not a single node, is what "owns" a key.
- **N=3 is the standard default** (Dynamo, Cassandra, Riak all default here): tolerates a single node failure without any read/write disruption, and a second simultaneous failure with degraded-but-often-still-functional service, at 3x storage cost — a reasonable durability/cost trade-off for a general-purpose store. Higher N (5, 7) is chosen for data where losing it is unacceptable even under correlated multi-node failure (e.g., financial ledgers); lower N is rarely used for anything but disposable cache data.

### Quorum: W, R, and the W + R > N rule
- **W** = number of replicas that must acknowledge a write before the coordinator reports success to the client.
- **R** = number of replicas the coordinator must hear from before returning a read to the client (if they disagree, see conflict resolution below).
- **The rule: choose W and R such that W + R > N.**

**Why this specific inequality, not just "pick reasonable numbers":** with N replicas, any write quorum of size W and any read quorum of size R are each *some subset* of the N replicas. If `W + R > N`, then by the pigeonhole principle, **any read quorum and any write quorum must share at least one common replica** — there's no way to pick W nodes for a write and R nodes for a read out of only N total nodes without at least one node appearing in both sets. That shared node guarantees a subsequent quorum read will observe the most recent quorum write (or a concurrent one it must reconcile — Section 4's conflict resolution), *without* requiring all N replicas to be involved in every operation. This is what makes the system tunable rather than all-or-nothing: you get a mathematically guaranteed overlap property at a fraction of the cost of contacting every replica on every request.

### Common configurations and their trade-offs

| Config (N=3) | Write cost | Read cost | Behavior |
|---|---|---|---|
| **W=2, R=2** (W+R=4>3) | Wait for 2 of 3 acks | Wait for 2 of 3 responses | **Standard balanced default** — tolerates 1 node being slow/down on either path, moderate latency on both |
| W=3, R=1 | Wait for all 3 acks (slow, fragile — any 1 node down blocks all writes) | Wait for just 1 response (fast) | Read-optimized; writes become the availability bottleneck |
| W=1, R=3 | Wait for just 1 ack (fast) | Wait for all 3 responses (slow, fragile) | Write-optimized; reads become the bottleneck, and a down node blocks all reads |
| W=1, R=1 (W+R=2, **not** >N) | Fast | Fast | **No overlap guarantee** — a read can legitimately miss the most recent write. Valid choice only when the workload explicitly prefers raw speed over read-your-writes consistency (e.g., ephemeral view counters) |

**Decision: W=2, R=2 is the correct general-purpose default** for the stated non-functional requirements (Section 1) — it satisfies W+R>N, survives a single node failure on either the read or write path without blocking, and keeps both operations' latency bounded by "2 out of 3," not "3 out of 3."

### Sloppy quorum + hinted handoff — staying available during a failure, not just tolerating one after the fact
If one of a key's N designated replicas is actually down when a write arrives, strict quorum would either block the write or violate the durability target. The Dynamo-style fix: the coordinator **temporarily writes to the next healthy node in the ring beyond the preference list**, attaching a "hint" recording which node this write actually belongs to. When the original node recovers, the hint is replayed to it (hinted handoff), restoring the intended replica placement. This is what "sloppy" means — the write satisfies W by count, but not necessarily by the *exact* preference-list membership, in exchange for never blocking a write just because one specific node is temporarily unreachable. This is a direct, protocol-level expression of the Section 1 choice to prioritize availability.

### Conflict resolution — the bill that comes due for choosing availability
Because writes can succeed on different, non-overlapping subsets of replicas during a partition (by design — that's the whole point of AP), the same key can end up with genuinely divergent values across replicas. Two mechanisms handle this:
- **Vector clocks** track, per key, which client/coordinator wrote which version and in what causal order. On read, if the versions form a clear causal chain (B was written *after* seeing A), the newer one wins automatically. If two versions were written *concurrently* (neither saw the other), **both are returned to the client as siblings** (Section 2's API) — the store makes no unilateral judgment call about which concurrent write is "correct," because it structurally cannot know; that's an application-level semantic (e.g., union the shopping cart contents) that the store shouldn't guess at.
- **Last-write-wins (LWW) by timestamp** is a simpler alternative that avoids exposing siblings to the client at all, at the cost of silently discarding one of two genuinely concurrent writes — acceptable for data where losing a stale concurrent update is harmless (a "last seen" timestamp), unacceptable where it isn't (an inventory decrement).
- **Read repair**: whenever a quorum read detects replicas disagree, the coordinator pushes the reconciled value back to the stale replica(s) as a side effect of the read — this is the main mechanism keeping replicas converged without a separate background job for every discrepancy (background anti-entropy, Section 6, catches what read repair doesn't).

---

## 5. Read/Write Flow

**Write path:**
1. Client sends `PUT` to any node (the "coordinator" for this request — any node can serve this role, it doesn't have to be one of the key's N replicas).
2. Coordinator computes the key's position on the ring, determines its preference list of N nodes.
3. Coordinator sends the write to all N replicas in parallel (using sloppy quorum/hinted handoff if any are down).
4. Coordinator waits for **W** acknowledgments, then returns success to the client. The remaining N−W replicas apply the write asynchronously.

**Read path:**
1. Client sends `GET` to any coordinator node.
2. Coordinator determines the preference list, sends the read to all N replicas (or just enough to satisfy R with some headroom for stragglers).
3. Coordinator waits for **R** responses. If they agree, return the value. If they disagree, resolve via vector clocks (return siblings or auto-merge, Section 4) and trigger read repair on the stale replica(s).

**Delete path:** implemented as writing a special **tombstone** value, not a physical row deletion — this is necessary so the delete itself replicates correctly through the same quorum protocol and doesn't get "resurrected" by an out-of-date replica's stale value winning a later read. Tombstones are physically purged after a grace period (long enough that any node that was down during the delete has had time to recover and receive it via hinted handoff or anti-entropy).

---

## 6. Scaling Architecture

```
                         Consistent hash ring (conceptual)

                                 Node A (vnodes)
                              ↗                  ↖
                     Node D                        Node B
                              ↖                  ↗
                                 Node C (vnodes)

Client ──▶ Coordinator (any node) ──▶ preference list for key = [B, C, D]  (N=3)
                                          │
                              ┌───────────┼───────────┐
                         Write/Read   Write/Read   Write/Read
                              │           │           │
                         ┌────▼───┐  ┌────▼───┐  ┌────▼───┐
                         │ Node B │  │ Node C │  │ Node D │
                         └────────┘  └────────┘  └────────┘

Background, always running:
  - Gossip protocol: nodes exchange cluster membership/health, no central coordinator needed
  - Anti-entropy (Merkle trees): periodically compare replica hash-trees to find and repair
    divergence that read repair alone didn't catch (keys that simply haven't been read since diverging)
  - Hinted handoff replay: queued hints delivered once their target node recovers
```

**Layer-by-layer scaling notes:**

1. **No single coordinator/master node** — any node can coordinate any request, and cluster membership is disseminated via **gossip** (nodes periodically exchange state with a few random peers) rather than a central registry. This is consistent with the AP/availability-first philosophy: there's no single component whose failure takes down request routing.

2. **Anti-entropy via Merkle trees.** Read repair only fixes divergence for keys that actually get read. Cold keys that diverged during a partition and are never read again would stay silently inconsistent forever without a separate mechanism. Nodes periodically build Merkle trees (hash trees) of their key ranges and compare them with replica peers — a mismatch at the root immediately narrows down to the specific diverged sub-range without comparing every key pairwise, making background reconciliation efficient even across large key ranges.

3. **Hot-key handling** — same structural issue as in the previous two docs: a single key's N-node preference list is fixed regardless of overall cluster size, so a sufficiently hot key can still overwhelm those specific N nodes. Mitigation here is usually a read-through cache in front of the store for known-hot keys (pushing the absorption point up a layer, same principle as the shortener's CDN fix) rather than trying to solve it inside the KV store's own replication scheme.

4. **Multi-datacenter replication** extends the same preference-list idea across regions — some deployments place a subset of a key's N replicas in a second datacenter explicitly, trading extra write latency (waiting on a cross-region ack, if that replica counts toward W) for surviving a full-datacenter loss. Many production systems make cross-DC replication asynchronous and outside the local W count specifically to avoid inflating every write's latency by a cross-region round trip — another explicit availability/latency-over-strict-consistency call, consistent with this document's theme throughout.

---

## 7. Bottlenecks & Likely Follow-Up Questions

- **"What if you need strong (linearizable) consistency for some keys?"** — this design is intentionally AP; it is the wrong tool for anything needing linearizability (leader election, distributed locks, financial transaction ledgers). Those need a CP system built on a consensus protocol (Raft/Paxos, à la Zookeeper/etcd or Spanner's TrueTime-backed design) with a real leader and majority-write blocking — worth naming as an explicit "different tool for a different consistency requirement," not a gap in this design.
- **"Do vector clocks grow forever?"** — yes, in principle, with enough distinct concurrent writers touching the same key; production systems cap vector clock size and drop the oldest entries once the cap is hit, trading a small amount of causal-history precision for bounded metadata size.
- **"How do you rebalance without a full stop-the-world data move?"** — this is exactly what consistent hashing (Section 3) buys you: a resize only moves the `~1/N` fraction of keys owned by the changed range, run as a background streaming operation with rate limiting, not a maintenance-window batch job.
- **"Where's the actual bottleneck under load — replication, partitioning, or something else?"** — a full numeric answer (machine counts, the specific inequality that breaks first) is exactly what the companion math-pass doc works out, same procedure as the URL shortener and rate limiter math passes.
