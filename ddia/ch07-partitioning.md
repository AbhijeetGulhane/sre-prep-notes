# DDIA Ch 7 — Partitioning (Sharding)

> 2nd edition numbering. Read for **tradeoffs and hot spots**; skim the secondary-index partitioning deep dive (one pass).
> Interview level: *"How do you shard this, and where's the hot spot?"*

---

## Why partition

Replication copies the same data everywhere; partitioning **splits different data across nodes** so a dataset larger than one machine — and a write load higher than one machine — can be handled. Almost always combined with replication: each partition is itself replicated.

The entire game is **spreading load evenly**. An uneven split (a *hot spot*) means one node does most of the work and you've gained nothing.

---

## The two partitioning strategies

### Key-range partitioning
Assign contiguous ranges of the key to each partition (A–F, G–M, ...).

- **Pro:** range scans are efficient — keys are sorted, adjacent data lives together.
- **Con:** easy to create hot spots. Timestamp keys → all recent writes hit one partition. Sequential IDs → same.
- Used by: HBase, Bigtable.

### Hash partitioning
Hash the key, assign hash ranges to partitions. Spreads writes evenly.

- **Pro:** uniform load distribution, kills the timestamp hot spot.
- **Con:** you lose efficient range scans — adjacent keys land on different partitions.
- Used by: Cassandra, DynamoDB, MongoDB (hashed shard key).

**The tradeoff in one line:** range = good scans, bad distribution; hash = good distribution, no scans. Pick based on whether your access pattern is range-heavy or point-lookup-heavy.

---

## Hot spots — the failure mode that matters

Even with hash partitioning, a **single hot key** ruins you: a celebrity's user ID, a viral tweet, a Black Friday product. Hashing the key doesn't help because it's *one* key — all traffic still lands on one partition.

Fixes:
- **Add a random suffix** to the hot key (split it across N partitions) — but now reads must query all N and merge. Manual, application-level.
- **Application-level caching** in front of the hot key.

**Interview trap:** "hash partitioning solves hot spots" is *wrong* for a single hot key. Hashing spreads *many* keys evenly; it does nothing for *one* overloaded key. Knowing this distinction is a strong signal.

---

## Rebalancing — moving data as the cluster grows

When you add nodes, data must move. The key insight: **don't use `hash mod N`** — changing N (adding a node) remaps almost every key, causing a massive reshuffle.

Approaches that avoid this:
- **Fixed number of partitions** — create way more partitions than nodes up front (e.g. 1000 partitions, 10 nodes); moving whole partitions between nodes is cheap. Simple and common.
- **Dynamic partitioning** — split/merge partitions as they grow/shrink (like key-range in HBase).
- **Consistent hashing** — minimises remapping when nodes change; know the term and that it exists.

Know *why `mod N` is bad* and that *fixed-partition-count* is the usual fix. Skim the rest.

---

## Request routing — how a client finds the right partition

Three models:
1. Client contacts any node, which forwards.
2. A routing tier sits in front.
3. Client is partition-aware and connects directly.

The routing metadata (which partition is on which node) is usually tracked by a separate coordination service — **ZooKeeper** is the classic answer. Know this ties back to consensus (Ch 10).

---

## Secondary indexes (SKIM — know the two shapes)

- **Local / document-partitioned index** — each partition indexes only its own docs. Writes are cheap (one partition); reads must **scatter-gather** across all partitions. Used by Elasticsearch, MongoDB.
- **Global / term-partitioned index** — the index itself is partitioned by term. Reads hit one partition; writes must update multiple. 

One-line tradeoff: local = cheap writes / expensive reads; global = expensive writes / cheap reads. That's enough — don't go deeper for an SRE loop.

---

## Interview-ready summary

- Partitioning = split different data to scale beyond one machine; goal is even load.
- Range (good scans, hot-spot-prone) vs hash (even, no scans).
- Single hot key defeats hashing — needs suffix-splitting or caching.
- Never `hash mod N` — use fixed partition count for cheap rebalancing.
- Routing metadata usually lives in ZooKeeper (→ consensus).

---

## Reading resources

- **DDIA 2nd ed, Ch 7** — primary. Read partitioning strategies + hot spots + rebalancing; skim secondary indexes. (paid)
- **"Consistent Hashing" — original explainer** (many free versions, e.g. Toptal's) — free — worth 15 min because it comes up in system design too. (free)
- **DynamoDB / Cassandra partition-key docs** — free — concrete real-world framing of hot-key problems. (free)
- Skip for now: term-partitioned index internals (supplementary).
