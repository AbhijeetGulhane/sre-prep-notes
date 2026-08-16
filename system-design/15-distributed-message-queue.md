# Distributed Message Queue (Kafka-like) — Comprehensive NALSD Reference

*Fifteenth in the system-design set. Another black box finally opened — Kafka's been used as a component in [07-email-system](./07-email-system.md), [09-video-platform](./09-video-platform.md), [11-logging-pipeline](./11-logging-pipeline.md), and [12-observability-tracing](./12-observability-tracing.md) without this series ever designing it directly. This is that doc, and it's worth reading with those four in mind — Section 6 explicitly checks what delivery semantic each of them actually needed.*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- Producers publish messages to named **topics**.
- Topics are split into **partitions** for parallelism (Section 3).
- Consumers read via **consumer groups**, tracking their own read position (**offset**) per partition (Section 4).
- Messages are **retained** for a configured period, not deleted on read — a genuinely different model from a traditional queue, worked out precisely in Section 4.
- Configurable **delivery semantics** (Section 6).

### Non-functional requirements
- **High throughput** on both produce and consume paths.
- **Durability** — no message loss once a write is acknowledged.
- **Ordering guaranteed only within a partition, never across a whole topic** — a deliberate, load-bearing trade-off (Section 3), not a limitation to work around.
- **Horizontal scalability** on both the broker side (more partitions, more brokers) and the consumer side (more consumers, up to a real ceiling — Section 3 and Section 7).

### Scale target
- **10,000,000 messages/sec peak**, system-wide across all topics — consistent with this being the shared backbone underneath the four docs listed above.
- **Average message size: 1 KB** (a blended figure across use cases — doc 07's email metadata, doc 09's transcode job descriptors, doc 11's log lines, doc 12's spans).

---

## 2. High-Level Architecture

```
Producers ──▶ Topic (N partitions, each an ordered append-only log)
                    │
              Brokers: each partition has one LEADER broker + replica FOLLOWERS
                    │
              Consumer Group A: partitions assigned 1:1 to group members
              Consumer Group B: independently reads the same topic, own offsets
                    │
              Offsets tracked per (consumer group, partition)
```

---

## 3. Partitions — the Crux Decision

**A topic is split into N partitions; each partition is an independent, ordered, append-only log** — structurally the same durable-write-ahead-log idea as doc 07's MTA queue write, just made the literal storage model here rather than an implementation detail underneath something else.

### Ordering — a deliberate, load-bearing trade-off
**Ordering is guaranteed only within a single partition, never across a whole topic.** This isn't a limitation to apologize for — guaranteeing global ordering across every partition would require serializing all writes through one place, which would destroy the entire reason partitions exist. **This is exactly why partition key choice matters as much as it does**, and it's the sharpest, most concrete instance yet of the partitioning theme running through this entire series (doc 06's host, doc 07's user, doc 08's region, doc 10/11's series-id/service-name): **if a use case needs all of one entity's events processed in order — every event for one user, one order ID, one device — that entity must be keyed consistently so its events always land in the same partition.** This is precisely why producer client libraries let you specify a partition key (typically `hash(key) % partition_count`) rather than round-robining every message randomly across partitions.

### Partition count is also a capacity decision, not just an ordering one
Partition count sets the hard upper bound on consumer parallelism within a single group (Section 4) — worth flagging now that this same number does double duty, since it resurfaces as the doc's actual first bottleneck in Section 7.

---

## 4. Offsets and Consumer Groups

### A genuinely different mental model from a traditional queue
Unlike a traditional message queue (RabbitMQ-style), where a consumed message is **removed** from the queue, Kafka-style logs **retain** messages according to a time- or size-based retention policy, regardless of whether anyone's consumed them yet. **Consumers track their own read position — the offset — into each partition**, rather than the broker deleting messages as they're read. Worth naming this contrast precisely: it's a "pull with a bookmark" model, not a "push and remove" model.

### Why this matters in practice
Multiple **independent consumer groups** can read the exact same topic without interfering with each other, each maintaining its own offset per partition — doc 11's Flink log-processing consumer group and a hypothetical separate audit-logging consumer group could both read the same raw log topic entirely independently. **This is structurally impossible in a delete-on-consume queue.**

### The hard ceiling on parallelism
Within one consumer group, exactly **one** consumer instance reads a given partition at a time. Add more consumers to a group than there are partitions, and the extras sit completely idle — **partition count is a hard upper bound on that group's parallelism, decided largely at topic-creation time**, and Section 7 works out exactly how this becomes the real bottleneck.

### Rebalancing — a coordination problem, closing a loop from doc 14
When a consumer joins or leaves a group (crash, scale event, deploy), partitions must be reassigned among the remaining/new members — a genuine coordination problem. **Worth the explicit callback**: Kafka historically used Zookeeper for exactly this purpose (and for broker leader election), before migrating to its own internal KRaft-based consensus protocol in modern versions. The "Zookeeper-like black box" referenced casually in earlier docs, then finally opened in doc 14, turns out to be literally what Kafka itself leaned on for years for this exact job — a satisfying full-circle moment across the series.

---

## 5. Replication — Related to, but Distinct from, Doc 14's Consensus

Each partition has a configurable **replication factor** (commonly 3, consistent with the number used throughout this repo). One broker is the **leader** for a given partition, handling all reads/writes; others are **followers**, passively replicating the leader's log.

### The ISR (In-Sync Replica) mechanism, precisely
Followers sufficiently caught up with the leader (within a configurable lag threshold) are considered **in-sync**. Only ISR members are eligible for promotion to leader if the current leader fails, and — depending on the producer's `acks` setting — a write may need acknowledgment from all ISR members before being considered committed.

**Worth being precise this is a genuinely different mechanism than doc 14's Raft-style consensus**, even though it rhymes structurally (both involve a leader, both involve tracking which replicas are trustworthy). ISR is a broker-tracked *set* of replicas known to be caught up, not a strict majority-vote quorum — leader election among ISR members is coordinated via a separate **controller** broker, itself elected through the Zookeeper/KRaft consensus layer doc 14 covers. Two related but distinct replication philosophies, worth not conflating just because they share vocabulary.

---

## 6. Delivery Semantics

### At-most-once
Consumer commits its offset **before** processing the message. A crash mid-processing means the message is never retried — simplest, but risks silent loss. Philosophically similar to the fail-open posture doc 02's rate limiter took under a Redis outage, just applied to message processing instead of request admission.

### At-least-once — the practical default
Consumer commits its offset **after** successfully processing. A crash after processing but before committing means the same message gets redelivered and reprocessed on restart — **never lost, but possibly processed more than once.** This is the default combination most real systems actually use, **because most downstream processing can be made idempotent more easily than delivery can be made loss-proof.** Worth stating explicitly: **at-least-once plus an idempotent consumer is the practical default underneath every one of the four earlier docs that used Kafka**, even though none of them named the semantic outright at the time — Section 8 checks this claim against each one directly.

### Exactly-once — genuinely the hardest, and narrower than it sounds
Requires either **transactional writes** spanning both the consume/offset-commit step and the downstream side effect atomically (Kafka's actual transactions API achieves this specifically for Kafka-to-Kafka pipelines — consume from topic A, produce to topic B, commit the offset on A, all as one atomic transaction), or **idempotent producers** (a producer-assigned sequence number per partition lets the broker detect and silently drop a duplicate retry, solving the narrower "did my write actually succeed" ambiguity on the producer side).

**Worth an honest, precise caveat, matching this series' running honesty about partial fixes**: "exactly-once" in practice almost always means "effectively-once within a specific, well-defined boundary" — within Kafka-to-Kafka, or via an idempotent downstream write — **not a universal guarantee across arbitrary external side effects.** Kafka cannot make an arbitrary external HTTP call idempotent on your behalf; that's still the calling system's responsibility.

---

## 7. Capacity Math and the First Real Bottleneck

### Checking both dimensions, per doc 11's lesson
```
Bandwidth: 10,000,000 msgs/sec × 1 KB = 10 GB/sec
÷ 0.5 GB/sec/node (same broker reference as docs 07/11)
= 20 nodes (bandwidth-bound estimate)

Message count: 10,000,000 ÷ 200,000 msgs/sec/node
= 50 nodes (count-bound estimate)

Take the max: 50 nodes, count-bound — same shape as doc 11's finding
```

### The correction doc 03.3 already taught us to expect — applied a second time
The 50-node figure only accounts for incoming produce traffic. **Replication adds real additional network load**: each broker leading a partition sends that partition's data to 2 followers (replication factor 3); each broker following a partition receives that same data.
```
Total system-wide network traffic, replication included:
  Produce traffic:      10 GB/sec
  Replication traffic:  2 × 10 GB/sec = 20 GB/sec (each message additionally sent to 2 replicas)
  Total:                30 GB/sec

30 GB/sec ÷ 0.5 GB/sec/node = 60 nodes
```
**60 nodes, not 50 — a real, quantified correction from replication fan-out, the same lesson doc 03.3 taught for the KV store, now recurring a second time in this series.** Worth having "always add the replication multiplier before finalizing a node count" as a fully automatic habit by this point, not a special case to remember to check.

**Disk-level sanity check, worth doing explicitly rather than skipping**: total write I/O across the fleet is produce traffic × replication factor = 10 GB/sec × 3 = 30 GB/sec, spread across 60 nodes ≈ 0.5 GB/sec/node of actual disk write throughput — comfortably within a single modern NVMe SSD's sequential write capability. Disk isn't the constraint here; network is.

### The first real bottleneck — the same recurring shape, a fourth time
Both throughput (60 nodes) and storage are comfortably solved by horizontal scaling. **The real structural bottleneck is Section 3 and Section 4's partition count** — a design-time parameter, chosen once, that silently caps a completely different, unrelated dimension later.

```
Worked example: a topic created with 20 partitions. Message volume grows over
time, and keeping up now requires 60 consumer instances in the group.

The extra 40 consumer instances sit completely idle — no matter how much
broker-side capacity exists — because Kafka's consumer-group model hard-caps
active consumers-per-partition at exactly 1.
```

**This is structurally the same bottleneck shape as doc 05's fixed-width machine-ID field and doc 10's cardinality ceiling** — a fourth instance of the same pattern in this series: a fixed, hard-to-change design-time parameter, chosen for one reason (here, ordering guarantees per Section 3), becomes an unrelated capacity ceiling later, once the system has scaled past what that original parameter anticipated. Worth having this as a fully crystallized, generalizable pattern by now, not a coincidence noticed four separate times.

### Iterate — and the honest cost of the fix
Increasing partition count on an existing topic is possible, but **it does not retroactively repartition already-written historical data**, and it has a real, easy-to-miss side effect: since the partition mapping is typically `hash(key) % partition_count`, **changing partition count changes which partition a given key maps to** — a key's future messages can land in a different partition than its earlier messages did, breaking the per-key ordering guarantee exactly at the repartitioning boundary. The same honest "doesn't fully fix it for free" posture as doc 06's quadtree-doesn't-fix-pole-distortion finding and doc 08's honest caveats — worth stating plainly rather than presenting "just add more partitions" as a costless capacity increase.

---

## 8. Which Delivery Semantic Did Each Earlier Doc Actually Need?

A concrete exercise worth doing explicitly, since none of the four earlier docs named this at the time:

| Doc | Queue use | Semantic actually needed |
|---|---|---|
| 07 (email MTA) | Inbound message durability before ack | At-least-once + idempotent downstream (duplicate delivery attempt is survivable if the recipient side dedupes by Message-ID) |
| 09 (video transcode) | Transcode job dispatch | At-least-once (a duplicate transcode job wastes compute, never produces an incorrect result) |
| 11 (log pipeline) | Log line ingestion | At-least-once (duplicate log lines are a minor nuisance, not corruption) |
| 12 (span/tracing pipeline) | Span ingestion | At-least-once (same tolerance as logs) |

**None of them needed exactly-once.** Worth stating as the actual takeaway: at-least-once plus idempotent consumers is not a compromise those docs settled for — it's the correct, sufficient answer for every one of them, and reaching for Kafka's transactional exactly-once machinery would have been unjustified complexity for all four.

---

## 9. Further Reading

- **"Kafka: a Distributed Messaging System for Log Processing"** (LinkedIn, the original paper) — **free**, **essential**.
- **Kafka's own documentation on the replication protocol and ISR** — **free**, **essential** — the direct source for Section 5's precision.
- **Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying abstraction"** — **free**, **essential** — the foundational piece on why the log-based model this entire doc is built around is genuinely powerful, not just a Kafka implementation detail.
- **Kafka's transactions/exactly-once semantics documentation** — **free**, **supplementary** — for going deeper on Section 6's nuance.

---

## 10. Summary

| Dimension | Value | Binding constraint? |
|---|---|---|
| Bandwidth-bound estimate | 20 nodes | No — count-bound wins |
| Count-bound estimate | 50 nodes | No — replication correction wins |
| **Replication-corrected node count** | **60 nodes** | The actual throughput answer, after applying the now-familiar fan-out correction |
| Disk I/O | ~0.5 GB/sec/node | No — comfortably within a single NVMe SSD |
| **Partition count** | **A fixed, design-time ceiling on consumer parallelism** | **Yes — the real first bottleneck, fourth instance of this exact shape in the series** |

## 11. Follow-up Questions

- **"How many partitions should a topic start with?"** — provision for a reasonable multiple of expected peak consumer parallelism, not just current need, precisely because Section 7 showed the fix for under-provisioning isn't free — this is a decision worth erring generous on rather than growing into later.
- **"What happens to consumer offsets during a rebalance?"** — a consumer's committed offset for a partition is preserved and picked up by whichever consumer the partition gets reassigned to, which is exactly what makes rebalancing safe rather than a data-loss event — the offset, not the consumer instance, is the durable source of truth for "how far has this group read."
- **"Why not just always use exactly-once if it's available?"** — real overhead: transactional writes cost meaningfully more throughput than at-least-once delivery, and Section 8 shows none of this repo's actual use cases needed the guarantee in the first place — reaching for the strongest available guarantee by default, rather than the one the workload actually requires, is the same category of over-engineering doc 08 flagged for defaulting to S2 everywhere.
