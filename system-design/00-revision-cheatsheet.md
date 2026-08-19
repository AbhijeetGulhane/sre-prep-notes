# System Design — Revision Cheatsheet

*One page per topic. Crux decisions, envelope estimates, first bottleneck only — full reasoning lives in the numbered docs. Read the "recurring patterns" section first; it's the highest-density part.*

---

## Recurring Patterns Across the Whole Series (say these out loud if stuck)

- **Accept-fast, durable-queue, process-async** (WAL pattern): doc 01's ID allocator, doc 07's MTA, doc 09's transcode queue, doc 11's Kafka ingestion, doc 15 generally. When something takes too long to do synchronously, durably queue it and ack immediately.
- **Partition by natural entity, not always hash**: doc 06 (host), doc 07 (user_id, range not hash — inbox is a range scan), doc 08 (region), doc 10/11 (series/time). Hash spreads load; range-by-natural-key preserves locality for "give me one entity's data" reads. Pick based on the *dominant read pattern*.
- **Check bytes AND count, take the max, never assume**: doc 07/09 were bandwidth-bound (bytes), doc 10 was count-bound (tiny payloads), doc 11's Kafka was ALSO count-bound (opposite of 07/09) while its ES layer was volume-bound. The discipline is checking every time, not pattern-matching the last answer.
- **Fixed design-time parameter becomes an unrelated ceiling later**: doc 05 (machine-ID bits, broken by k8s autoscale), doc 10 (cardinality, broken by a bad label), doc 15 (partition count, broken by consumer growth). Four instances — a very safe bet as "where does this break" answer.
- **Blob storage vs. real DB, always split**: doc 06 (crawled content), doc 07 (attachments), doc 09 (GCS vs Spanner). Big binary → object storage. Structured/queryable → proper DB.
- **Three ways to get machines to agree**: doc 03.1 = tolerate & merge (quorum + vector clocks, AP). doc 09 = avoid via synchronized clocks (TrueTime + Paxos, CP). doc 14 = avoid via majority-vote leader (Raft/Paxos/ZAB, CP). Know which one a system needs before reaching for one.
- **Not every "add more nodes" fixes it**: doc 06's politeness ceiling (policy, no fix) vs. doc 14's cross-region latency (physics, no fix) vs. doc 14's own write-throughput (MORE nodes = WORSE latency here — the one true exception to "scale by adding nodes" in the whole series).
- **Replication fan-out correction**: doc 03.3 (KV store, 2.2x) and doc 15 (Kafka, 50→60 nodes) both under-counted until replication traffic was added explicitly. Always multiply.
- **Some bottlenecks aren't in the math at all**: doc 12's tracing — all capacity numbers were comfortable; the real risk (clock skew, instrumentation coverage) is a correctness/adoption problem invisible to any node-count calculation.

---

## 01 — URL Shortener
- Scale: 100B URLs, 100:1 read:write, 150K peak read QPS.
- **Envelope**: 100B URLs → 7-char Base62 (62⁷≈3.5T headroom) → avg 500B/record → 50TB raw → ×3 replication → 150TB. Reads: 31.7K avg → ×5 peak → 150K/sec.
- **Crux**: Base62 + counter/range-allocator, NOT MD5 hash — avoids collision-retry latency that *worsens* as the table fills.
- DB: NoSQL/wide-column default; sharded MySQL = named fallback.
- Redirect: hybrid 302 + short-TTL CDN edge cache (not plain 301).
- **Storage-bound** (38 nodes @ 4TB/node) not throughput-bound (only needs 2 @ 15K ops/sec/node) — 19x overprovisioned on storage covers redundancy for free.
- Hot link breaks a single cache node at **~56%** traffic share; CDN drops origin load to ~3.3 req/sec regardless of raw volume.
- Tip: state your read:write ratio assumption out loud — ByteByteGo used 10:1, we used 100:1, both defensible.

## 02 — Rate Limiter
- **Envelope**: 150K peak req/sec (reused shortener target) → Redis node @ 80K ops/sec → 2 shards min by throughput, but 3 needed for single-shard-failure survival (2 shards leaves only 6.7% headroom). Hot key math: node breaks once one key exceeds ~56-60% of total traffic share on its shard.
- Token bucket (bursts OK, default) vs leaky bucket (smooths, for protecting a fixed-rate downstream) vs sliding-window counter (closes fixed-window edge exploit, use for security-sensitive limits).
- Redis: `SET key val NX PX ttl` atomic — never separate `INCR`+`EXPIRE` (orphan-key race).
- Layered placement: edge (IP) → gateway (user/key) → per-service (endpoint).
- **Fail-open** on Redis outage — availability over enforcement.
- Stampede (one hot key): SETNX lock + jittered/probabilistic early expiry (XFetch).
- Thundering herd (cold cache, many keys): different problem — warm before serving (snapshot/top-N/shadow traffic), roll node-by-node, rate-limit the DB as insurance.
- Cold cache = **~20x** DB overload (95%→0% hit ratio flips 50K→1M req/sec).

## 03 — Distributed KV Store
- **Envelope**: 1B keys × 1KB avg = 1TB raw → ×N=3 → 3TB. 500K ops/sec (400K read, 100K write). Fan-out correction: writes hit all N=3, reads hit R=2 → backend ops ≈ 1.1M/sec (2.2x client QPS) → 55 nodes @ 20K ops/sec/node, not naive 25.
- AP/Dynamo: consistent hashing + vnodes, N=3, W=2 R=2 (**W+R>N** guarantees quorum overlap).
- Sloppy quorum + hinted handoff for write availability during a down node.
- **Vector clocks, not LWW** — LWW silently drops concurrent writes. Cap clock size (~10 entries), drop oldest.
- **Real bottleneck: quorum-loss probability**, not throughput/storage — 3×10⁻⁶/key/day at N=3, × 1B keys = ~3,000/day losing quorum, invisible until multiplied out.
- N=3→N=5 buys 300x reliability (10/day vs 3000/day) for ~55% more nodes (55→85).
- Correlated failures (rack/AZ) break the independence assumption — need failure-domain-aware placement.

## 04 — Distributed Cache
- **Envelope**: 1M ops/sec peak, target 95% hit ratio → only 50K ops/sec should reach DB. Working set 250M keys × 2KB = 500GB. Cache: 10 nodes (throughput-bound, 100K ops/sec/node). DB at point estimate: 4 nodes (@15K ops/sec/node); at 85% floor: 10 nodes (150K/sec ÷ 15K).
- Cache-aside (default, resilient) / write-through (no miss, slow writes) / write-back (fast, real durability risk) / write-around (avoids polluting cache with write-once data).
- Approximate LRU: sample ~5 keys, evict oldest of sample — real Redis behavior.
- **Invalidation race**: a stale read can populate the cache *after* invalidation — mitigated by short TTL, never eliminated.
- Cache degrades **gracefully** (Zipf curve: halving cache size only costs ~9pp hit ratio); DB has **zero headroom by default** — a 5pt hit-ratio dip (95→90%) doubles DB load. Fix: provision DB against a floor (85%), not a point estimate.

## 05 — Unique ID Generator (Snowflake)
- **Envelope**: 64-bit = 1 sign + 41 timestamp + 10 machine-id + 12 sequence. 2⁴¹ms ≈ 69.7yr. 2¹² = 4096 ids/ms/machine = 4.096M ids/sec/machine. Cluster theoretical max: 4.19B ids/sec (1024 machines) — target of 1M/sec cluster-wide is 4000x under this ceiling.
- **Real bottleneck: machine-ID space (1024)** — broken by k8s pod autoscale (address-space exhaustion, e.g. 5000 pods > 1024 slots), not by load.
- Fix: widen machine-ID field (free, steals from seq's huge headroom — total bit budget for machine+seq stays constant either way) or a small fixed generator pool + batched fetch, or lease/TTL reuse (delays, doesn't remove the ceiling).
- Clock rollback: even 1ms matters — fail loud, never proceed silently.

## 06 — Web Crawler
- **Envelope**: 1B pages/month → 386/sec avg, 1,158/sec peak. Bloom filter: 10B unique URLs, 1% FP → m=−(n·ln p)/(ln2)² ≈ 11.2GB, k≈7 hash functions (vs 400GB for an exact set — 35x smaller).
- Frontier: Mercator 2-level (front queues = priority, back queues = 1/host + politeness timestamp).
- Politeness = per-host token bucket (literally doc 02, host-scoped).
- **Partition by host, not hash(URL)** — one worker must own full politeness state per host.
- **Real bottleneck: NO engineering fix.** Politeness ceiling — 60M-page site @ 1req/s = **1.9 years**, regardless of cluster size. Aggregate throughput only comes from breadth (many hosts), never depth.

## 07 — Email System
- **Envelope**: 500M users × 25 msgs/day = 12.5B/day → 144.7K avg, 434K peak msgs/sec. Bandwidth: 434K × 75KB = 31.8GB/sec → 26 nodes @ 1.25GB/sec (10Gbps NIC), vs naive count-only estimate of 9 nodes. Storage: 937.5TB/day raw → 2.81PB/day ×3 replication. Search: 434K/sec ÷ 5K docs/sec/node = 87 nodes.
- MTA: accept-fast-durable-queue-async — never hold the SMTP connection through the full pipeline.
- Storage: Bigtable-style, **range-partition by user_id + reverse_timestamp** — opposite of the KV store's hash default, because inbox reads are range scans.
- **Search indexing is the biggest layer (87 nodes, 3x ingestion's 26)** — indexing, not the front door, dominates.
- Hot mailbox (4000x normal) IS fixable — Bigtable auto tablet-split (unlike doc 06's ceiling).

## 08 — Proximity Service
- **Envelope**: 500M POIs, ~100 bytes/tree-node → 50GB in-memory quadtree (fits one machine, no headroom → triggers regional sharding). 100K peak queries/sec ÷ 5K/sec/node = 20 nodes (trivial).
- Geohash: bit-interleave grid, prefix search, O(1) write, bad density skew, bad pole distortion, boundary problem (need 3x3 neighbor search).
- Quadtree: adaptive subdivision fixes density skew, does **not actually fix pole distortion** (still a flat plane) — don't overclaim this.
- S2: sphere→cube→Hilbert curve→64-bit cell ID — the only one that *genuinely* fixes pole distortion.
- Real bottleneck: **query-time variance**, not storage — dense area = CPU/ranking-bound, sparse area = traversal/radius-expansion-bound. Same skew, two shapes.

## 09 — Video Platform
- **Envelope**: 500hr/min uploaded × 5Mbps avg → 18.75GB/sec ingestion bandwidth → 15 nodes. Daily raw: 720K hrs/day × 2.25GB/hr = 1.62PB/day. Transcode: 720K hrs × 0.5 compute-hr/hr = 360K compute-hrs/day ÷ 24 = **15,000 continuous workers** (1000x the 15-node ingestion layer).
- Upload: same accept-fast-async pattern as doc 07, resumable/chunked.
- ABR: multiple renditions + HLS/DASH segments + manifest; **client** picks bitrate, not server.
- Metadata: **Spanner (CP, TrueTime+Paxos)** — the concrete answer to doc 03.1's "what about strong consistency" question. View counts stay approximate (doc 02/04-style counters, NOT Spanner).
- Bottleneck fix is a **latency lever, not capacity** — parallelizing one video's transcode redistributes fixed compute-hours across time, doesn't reduce them.
- Presigned URLs: client uploads direct to GCS, bypassing app tier — corrects the naive ingestion-node math.

## 10 — Monitoring & Alerting
- **Envelope**: 100K hosts × 1000 series = 100M active series ÷ 10s interval = 10M data points/sec. Ingestion: 10M ÷ 200K/sec/node = 50 nodes (count-bound, checked explicitly). Storage: 10M × 1.37 bytes (Gorilla compression) = 13.7MB/sec → 1.18TB/day (tiny, vs docs 07/09's PB-scale).
- Pull (Prometheus): "absence = signal" structurally, free backpressure. Needs Pushgateway bridge for ephemeral jobs.
- **Real bottleneck: cardinality, not throughput/storage** — one bad label = millions of new series in minutes (~2KB/series index overhead → 10M new series = 20GB unplanned memory), same shape as doc 05's address-space exhaustion.
- Alerting: group by label (avoid 50-page storms), inhibition, `for: 5m` hysteresis (avoid flapping).

## 11 — Logging Pipeline
- **Envelope**: 100K hosts × 150/sec peak = 15M lines/sec, 500B avg → 7.5GB/sec (bandwidth: 15 nodes) vs 15M÷200K/sec/node=75 nodes (count-bound wins, opposite of docs 07/09). ES indexing: 15M÷20K docs/sec/node = **750 nodes**. After Flink filters ~90% of low-value logs: 750→75.
- Kafka → Flink → Elasticsearch → Kibana; Kafka = 6th instance of the durable-queue pattern.
- Partition by **service/hash** — no locality benefit for cross-service search, closer to doc 03.1's hash-spread logic than doc 07's range logic.
- **ES indexing = 750 nodes, 10x Kafka** — second confirmation indexing dominates any full-text pipeline.

## 12 — Observability (Tracing)
- **Envelope**: 500K req/sec × 10 hops = 5M spans/sec at 100% sampling, ~200B/span → 1GB/sec (cheaper than doc 11's logging pipeline in both bytes and count). At realistic 5% sampling: 250K spans/sec persisted.
- `traceparent` header propagates trace_id + parent_span_id every hop; each service mints its own child span_id.
- Sampling: head-based (cheap, upfront, **misses** interesting traces) vs tail-based (buffers all, catches errors, but still pays full collection cost). Hybrid: 1–10% baseline + always-keep error/latency override.
- **Key lesson**: all capacity numbers here are comfortable — the hardest problem (clock skew, instrumentation coverage) is invisible to the math entirely.

## 13 — Isolation Levels (databases/, not NALSD)
- No envelope math — pure transaction theory doc.
- Dirty read → prevented by Read Committed+. Phantom read → ANSI: only Serializable guarantees it.
- Lost update → **not** prevented by Read Committed; Snapshot Isolation only prevents it with *added* write-write conflict detection.
- Read skew → prevented by Snapshot Isolation/Serializable, not Read Committed.
- Write skew (bonus): SI's conflict detection does **not** catch this, only true Serializable does.
- Tip: never trust an isolation-level *name* across databases — Postgres "Repeatable Read" = Snapshot Isolation; MySQL's is lock-based and stronger than the standard requires.

## 14 — Distributed Lock Service (Chubby/ZooKeeper)
- **Envelope**: deliberately small system — 5-node ensemble, ~10K ops/sec target. A single Paxos/ZAB leader sustains 10K–50K ops/sec alone, already meeting the target at N=1's throughput. Sizing formula is NOT load-based: fault tolerance = (N-1)/2 → N=5 tolerates 2 failures. Cross-region round trip: NYC↔London ≈70ms, NYC↔Tokyo ≈150ms+ (pure physics, bounds write latency regardless of node count).
- Split-brain prevention: any 2 majorities in an odd-N ensemble **must** overlap by ≥1 node — structural, not a merge strategy.
- **Fencing tokens**: monotonic token per grant; the *protected resource*, not the lock service, rejects stale tokens. Doesn't prevent the race, prevents the harm.
- **Math is inverted here**: more nodes = worse write latency (bigger majority needed) — the one true exception to "add nodes to scale" in this whole series.

## 15 — Distributed Message Queue (Kafka)
- **Envelope**: 10M msgs/sec, 1KB avg → 10GB/sec bandwidth (20 nodes) vs 10M÷200K/sec/node=50 nodes (count-bound wins) → +replication fan-out (2x extra network traffic) → 30GB/sec total → **60 nodes**, not naive 50.
- Partition = ordering boundary, only within, never across — key choice is everything.
- Consumer parallelism hard-capped at partition count — extras sit idle.
- Delivery: at-most-once (may lose) / **at-least-once + idempotent consumer = practical default** / exactly-once (real but narrow).
- Checked against docs 07/09/11/12: **all four only ever needed at-least-once**.
- **Real bottleneck: partition count** — 4th instance of "fixed parameter becomes unrelated ceiling." Fix (add partitions) breaks per-key ordering at the boundary — not free.
