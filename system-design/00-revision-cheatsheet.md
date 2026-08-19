# System Design — Revision Cheatsheet

*One page per topic. Crux decisions, key numbers, first bottleneck, tips only — full reasoning lives in the numbered docs. Read the "recurring patterns" section first; it's the highest-density part.*

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
- **Crux**: Base62 + counter/range-allocator, NOT MD5 hash — avoids collision-retry latency that *worsens* as the table fills.
- 7-char codes (62⁷ ≈ 3.5T ≫ 100B).
- DB: NoSQL/wide-column default; sharded MySQL = named fallback.
- Redirect: hybrid 302 + short-TTL CDN edge cache (not plain 301).
- **Storage-bound** (38 nodes) not throughput-bound (only needs 2) — 19x overprovisioned on storage covers redundancy for free.
- Hot link breaks a single cache node at **~56%** traffic share; CDN drops origin load to ~3.3 req/sec regardless of raw volume.
- Tip: state your read:write ratio assumption out loud — ByteByteGo used 10:1, we used 100:1, both defensible.

## 02 — Rate Limiter
- Token bucket (bursts OK, default) vs leaky bucket (smooths, for protecting a fixed-rate downstream) vs sliding-window counter (closes fixed-window edge exploit, use for security-sensitive limits).
- Redis: `SET key val NX PX ttl` atomic — never separate `INCR`+`EXPIRE` (orphan-key race).
- Layered placement: edge (IP) → gateway (user/key) → per-service (endpoint).
- **Fail-open** on Redis outage — availability over enforcement.
- Stampede (one hot key): SETNX lock + jittered/probabilistic early expiry (XFetch).
- Thundering herd (cold cache, many keys): different problem — warm before serving (snapshot/top-N/shadow traffic), roll node-by-node, rate-limit the DB as insurance.
- Cold cache = **~20x** DB overload (95%→0% hit ratio flips 50K→1M req/sec).

## 03 — Distributed KV Store
- AP/Dynamo: consistent hashing + vnodes, N=3, W=2 R=2 (**W+R>N** guarantees quorum overlap).
- Sloppy quorum + hinted handoff for write availability during a down node.
- **Vector clocks, not LWW** — LWW silently drops concurrent writes. Cap clock size (~10 entries), drop oldest.
- **Real bottleneck: quorum-loss probability**, not throughput/storage — tiny per-key risk × 1B keys = thousands/day, invisible until multiplied out.
- N=3→N=5 buys 300x reliability for ~55% more nodes.
- Correlated failures (rack/AZ) break the independence assumption — need failure-domain-aware placement.
- Fan-out gotcha: backend ops ≈ 2.2x client QPS (writes hit all N, reads hit R).

## 04 — Distributed Cache
- Cache-aside (default, resilient) / write-through (no miss, slow writes) / write-back (fast, real durability risk) / write-around (avoids polluting cache with write-once data).
- Approximate LRU: sample ~5 keys, evict oldest of sample — real Redis behavior.
- LFU: probabilistic decaying 8-bit counter, not a naive counter (stale popularity must fade).
- **Invalidation race**: a stale read can populate the cache *after* invalidation — mitigated by short TTL, never eliminated.
- Stampede (one key) → SETNX lock. Cold cache (many keys) → warming + gradual ramp + DB rate-limit, a *different* problem.
- Cache degrades **gracefully** (Zipf curve); DB has **zero headroom by default** — a 5pt hit-ratio dip (95→90%) doubles DB load. Fix: provision DB against a floor (85%), not a point estimate — costs real nodes (4→10).

## 05 — Unique ID Generator (Snowflake)
- 64-bit = 1 sign + 41 timestamp(ms, custom epoch) + 10 machine-id + 12 sequence.
- 41 bits ≈ 69.7yr; 12-bit seq = 4.096M ids/sec/machine — throughput is never the bottleneck (4000x headroom).
- **Real bottleneck: machine-ID space (1024)** — broken by k8s pod autoscale (address-space exhaustion), not by load.
- Fix: widen machine-ID field (free, steals from seq's huge headroom) or a small fixed generator pool + batched fetch, or lease/TTL reuse (delays, doesn't remove the ceiling).
- Clock rollback: even 1ms matters — fail loud, never proceed silently.

## 06 — Web Crawler
- Frontier: Mercator 2-level (front queues = priority, back queues = 1/host + politeness timestamp).
- Politeness = per-host token bucket (literally doc 02, host-scoped).
- **Partition by host, not hash(URL)** — one worker must own full politeness state per host.
- Bloom filter: 10B URLs @ 1% FP ≈ **11GB** (35x smaller than exact set), 7 hash functions.
- URL dedup ≠ content dedup: exact hash for byte-identical, shingling/MinHash/SimHash for near-dup.
- **Real bottleneck: NO engineering fix.** Politeness ceiling — 60M-page site @ 1req/s = **1.9 years**, regardless of cluster size. Aggregate throughput only comes from breadth (many hosts), never depth.

## 07 — Email System
- MTA: accept-fast-durable-queue-async — never hold the SMTP connection through the full pipeline.
- Queue partitioned by recipient (per-mailbox ordering).
- Storage: Bigtable-style, **range-partition by user_id + reverse_timestamp** — opposite of the KV store's hash default, because inbox reads are range scans.
- Search: separate inverted index (can't search a row store), sharded per-user.
- Threading: Message-ID/In-Reply-To/References (primary), subject-normalize + participant overlap (fallback).
- Math: **bandwidth-bound** ingestion (26 nodes, not 9 naive) — small avg size makes bytes matter.
- **Search indexing is the biggest layer (87 nodes, 3x ingestion)** — indexing, not the front door, dominates.
- Hot mailbox (4000x normal) IS fixable — Bigtable auto tablet-split (unlike doc 06's ceiling).

## 08 — Proximity Service
- Geohash: bit-interleave grid, prefix search, O(1) write, bad density skew, bad pole distortion, boundary problem (need 3x3 neighbor search).
- Quadtree: adaptive subdivision fixes density skew, does **not actually fix pole distortion** (still a flat plane) — don't overclaim this.
- S2: sphere→cube→Hilbert curve→64-bit cell ID — the only one that *genuinely* fixes pole distortion. Google's choice, justified at true global scale.
- Recommendation: quadtree (in-memory serving) + geohash (DB persistence) + S2 reserved for pole-inclusive global need.
- Partition by region, not hash.
- Real bottleneck: **query-time variance**, not storage — dense area = CPU/ranking-bound, sparse area = traversal/radius-expansion-bound. Same skew, two shapes.

## 09 — Video Platform
- Upload: same accept-fast-async pattern as doc 07, resumable/chunked.
- ABR: multiple renditions + HLS/DASH segments + manifest; **client** picks bitrate, not server.
- Transcode: a DAG (per-video/per-rendition/per-language), parallelized across and within renditions.
- Metadata: **Spanner (CP, TrueTime+Paxos)** — the concrete answer to doc 03.1's "what about strong consistency" question. View counts stay approximate (doc 02/04-style counters, NOT Spanner).
- Math: upload bandwidth-bound (15 nodes). **Transcoding dominates (15,000 workers, ~1000x ingestion)**.
- Bottleneck fix is a **latency lever, not capacity** — parallelizing one video's transcode redistributes fixed compute-hours across time, doesn't reduce them.
- Presigned URLs: client uploads direct to GCS, bypassing app tier — corrects the naive ingestion-node math.
- Video dedup: exact hash + perceptual fingerprint (Hamming distance) — the video analog of doc 06's text dedup.

## 10 — Monitoring & Alerting
- Pull (Prometheus): "absence = signal" structurally, free backpressure. Needs Pushgateway bridge for ephemeral jobs.
- Push (StatsD): needs its own rate limiter (doc 02-style), silence is ambiguous, UDP = accepted loss for zero app-blocking.
- Default: pull + Pushgateway hybrid.
- TSDB: Gorilla delta-of-delta compression ≈ **1.37 bytes/point** (vs 16 naive) — why 10M pts/sec is tractable.
- **Real bottleneck: cardinality, not throughput/storage** — one bad label = millions of new series in minutes, same shape as doc 05's address-space exhaustion.
- Alerting: group by label (avoid 50-page storms), inhibition (suppress redundant downstream alerts), `for: 5m` hysteresis (avoid flapping).
- Discipline win: checked bytes-vs-count explicitly and found count wins here (small payloads) — didn't just reapply doc 07/09's answer.

## 11 — Logging Pipeline
- Kafka → Flink → Elasticsearch → Kibana; Kafka = 6th instance of the durable-queue pattern.
- Partition by **service/hash** — no locality benefit for cross-service search, closer to doc 03.1's hash-spread logic than doc 07's range logic.
- Flink justified by **state** (enrichment/dedup/windowed agg needs checkpointing) — else just use Kafka Connect.
- ES: index-per-day = cheap bulk delete via index-drop, not per-doc delete.
- Math: Kafka is **count-bound (75 nodes)**, not bandwidth-bound (15) — opposite of doc 07/09, confirms "check both, don't assume."
- **ES indexing = 750 nodes, 10x Kafka** — second confirmation indexing dominates any full-text pipeline.
- Fix: Flink filters/samples before indexing (drop DEBUG, sample INFO 1-in-100) → 750→75 nodes.

## 12 — Observability (Tracing)
- `traceparent` header propagates trace_id + parent_span_id every hop; each service mints its own child span_id.
- Sampling: head-based (cheap, upfront, **misses** interesting traces) vs tail-based (buffers all, catches errors, but still pays full collection cost). Hybrid: 1–10% baseline + always-keep error/latency override.
- 5% sampling does ~all the cost reduction; tail/error-override is a small refinement, not the primary lever.
- Clock skew: child span can appear to start before parent — mitigated via the span tree structure, not raw timestamps; same root cause as doc 05's clock rollback.
- Unifying mechanism: trace_id threaded into every log line/metric — the actual point of "observability," but requires disciplined adoption everywhere.
- **Key lesson**: all capacity numbers here are comfortable (cheaper than doc 11 in both bytes and count) — the hardest problem (skew, adoption) is invisible to the math entirely.

## 13 — Isolation Levels (databases/, not NALSD)
- Dirty read → prevented by Read Committed+.
- Phantom read → ANSI: only Serializable guarantees it; real DBs (InnoDB locking, PG snapshot) often prevent it anyway under the same level name, for different reasons.
- Lost update → **not** prevented by Read Committed; Snapshot Isolation only prevents it with *added* write-write conflict detection (not inherent). Fix: atomic UPDATE, `SELECT FOR UPDATE`, or version-column CAS.
- Read skew → prevented by Snapshot Isolation/Serializable, not Read Committed.
- Write skew (bonus, easy to confuse with lost update): disjoint writes on overlapping reads violate an invariant — SI's conflict detection does **not** catch this, only true Serializable does.
- Tip: never trust an isolation-level *name* across databases — Postgres "Repeatable Read" = Snapshot Isolation; MySQL's is lock-based and stronger than the standard requires.

## 14 — Distributed Lock Service (Chubby/ZooKeeper)
- Third consistency mechanism: doc 03.1 = tolerate/merge, doc 09 = synchronized clocks, doc 14 = **majority-vote consensus**.
- Split-brain prevention: any 2 majorities in an odd-N ensemble **must** overlap by ≥1 node — structural, not a merge strategy.
- Ephemeral node + session lease: auto-releases on crash, does **not** fully solve "slow but alive" (GC pause).
- **Fencing tokens**: monotonic token per grant; the *protected resource*, not the lock service, rejects stale tokens. Doesn't prevent the race, prevents the harm — needs downstream integration (real adoption cost, same shape as doc 12's trace_id).
- Leader election = same primitive as a lock (race to create the same ephemeral node).
- **Math is inverted here**: more nodes = worse write latency (bigger majority needed) — the one true exception to "add nodes to scale" in this whole series. Sizing formula = fault tolerance `(N-1)/2`, not load/capacity.
- Cross-region latency = pure physics, no fix — a *different kind* of unfixable than doc 06's policy ceiling.

## 15 — Distributed Message Queue (Kafka)
- Partition = ordering boundary, only within, never across — key choice is everything.
- "Pull with a bookmark" (retain + consumer tracks offset) vs traditional "push and delete" — enables independent consumer groups on the same topic.
- Consumer parallelism hard-capped at partition count — extras sit idle.
- ISR replication ≠ doc 14's Raft — related but distinct (broker-tracked caught-up set + controller election, not strict majority quorum).
- Delivery: at-most-once (may lose) / **at-least-once + idempotent consumer = practical default** / exactly-once (real but narrow — only within Kafka-to-Kafka txns or idempotent-producer scope).
- Checked against docs 07/09/11/12: **all four only ever needed at-least-once** — evidence against reflexively reaching for exactly-once.
- Math: replication fan-out correction again (50→60 nodes).
- **Real bottleneck: partition count** — 4th instance of "fixed parameter becomes unrelated ceiling." Fix (add partitions) breaks per-key ordering at the boundary — not free.
