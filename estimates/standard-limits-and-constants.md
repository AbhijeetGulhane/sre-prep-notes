# Standard Limits & Constants

**Purpose:** the fixed limits and canonical constants you *recall* in an interview — protocol caps, encoding capacities, quorum rules, per-domain defaults. Companion to `capacity-estimation-cheatsheet.md` (which holds the ballparks you *compute* with). Numbers here are industry defaults; most are configurable, so state them as "default / typical," name the knob, and move on.

Each section tags the archetype it shows up in, but the numbers are general — they transfer to any prompt with the same shape.

---

## 1. ID generation & encoding
*Shows up in: unique-ID generator, URL shortener, any sharded primary key.*

**Snowflake ID (64-bit layout):**

| Field | Bits | Capacity |
|---|---|---|
| Sign (unused) | 1 | always 0 |
| Timestamp (ms) | 41 | **~69 years** of ms |
| Machine/node ID | 10 | **1,024 nodes** |
| Sequence | 12 | **4,096 IDs / ms / node** |
| **Total** | 64 | **~4M IDs/sec/node** |

| Encoding | Capacity |
|---|---|
| UUID | 128-bit (v4 = 122 random bits) — globally unique, no coordination |
| base62, 6 chars | 62⁶ ≈ **56.8 billion** |
| base62, 7 chars | 62⁷ ≈ **3.5 trillion** |
| base64 (URL-safe) | denser but `-`/`_` chars; base62 avoids them |

**Integer ranges (will it overflow?):**

| Type | Max |
|---|---|
| int32 | **~2.15 billion** (2³¹−1) |
| uint32 | ~4.29 billion |
| int64 | **~9.2 × 10¹⁸** (2⁶³−1) |

---

## 2. Consistency, replication & quorum
*Shows up in: distributed KV store, cache, lock service, message queue.*

| Concept | Standard |
|---|---|
| Replication factor (N) | **3** (default across Dynamo, Cassandra, Kafka, HDFS) |
| Quorum for strong consistency | **R + W > N** — e.g. N=3, W=2, R=2 |
| Fast writes / slow reads | W=1, R=N |
| Fast reads / slow writes | W=N, R=1 |
| PACELC | if Partition: choose A or C; **E**lse: choose **L**atency or **C**onsistency |

---

## 3. Consistent hashing
*Shows up in: KV store, cache, sharded datastore, load distribution.*

| Metric | Value |
|---|---|
| Virtual nodes per physical node | **~100–256** — smooths distribution, shrinks variance |
| Key remap on node add/remove | **~1/N of keys** (vs `hash % N` remapping ~all) |
| Why vnodes | one physical node owns many ring segments → even load + graceful rebalancing |

---

## 4. Datastore hard limits (item / value / object sizes)
*Shows up in: everywhere you pick a store — know the ceiling before you design around it.*

| Store | Hard limit |
|---|---|
| Redis | value/string **512 MB** max; keyspace up to 2³² keys |
| Memcached | **1 MB** item (default, configurable) |
| Kafka | message **~1 MB** default (`message.max.bytes`) |
| DynamoDB | item **400 KB** |
| MongoDB | BSON document **16 MB** |
| Cassandra | partition — keep **< 100 MB / < 100k cells** (practical, not hard) |
| PostgreSQL | field 1 GB (TOAST); table 32 TB |
| S3 | object **5 TB** max; single PUT 5 GB; multipart above |

---

## 5. Geospatial
*Shows up in: proximity service, ride-hailing, "nearby" search.*

**Geohash precision (cell size shrinks ~×8 per added char):**

| Length | Approx cell | Use |
|---|---|---|
| 4 | ~39 × 20 km | city |
| 5 | ~4.9 × 4.9 km | district |
| 6 | ~1.2 × 0.6 km | **neighborhood** (common for "nearby") |
| 7 | ~153 × 153 m | block |
| 8 | ~38 × 19 m | building |

| Alternative | Note |
|---|---|
| S2 cells (Google) | spherical, hierarchical, no pole distortion |
| H3 (Uber) | hexagonal — uniform neighbor distance |
| Quadtree | adapts to density (splits busy regions) |
| 1° latitude | ≈ **111 km** (longitude shrinks toward poles) |

---

## 6. Media & video
*Shows up in: video platform, streaming, image/CDN service.*

| Resolution | Bitrate | Storage / hour |
|---|---|---|
| 480p | ~1.5 Mbps | ~0.7 GB |
| 720p | ~3 Mbps | ~1.35 GB |
| **1080p** | **~5 Mbps** | **~2.25 GB** |
| 4K (2160p) | ~25 Mbps | ~11 GB |
| Audio | ~128 kbps | — |

| Concept | Standard |
|---|---|
| Storage formula | Mbps × 3600 ÷ 8 = GB/hour |
| HLS/DASH segment | **2–10 s** (6 s common) |
| Renditions | store multiple qualities → **~1.5–2× original** total |
| Delivery | CDN edge caching; origin only on miss |

---

## 7. Metrics & time-series
*Shows up in: monitoring/alerting, observability.*

| Metric | Standard |
|---|---|
| Scrape interval | **15 s** default (10–60 s range) |
| Sample size on disk | **~1–2 bytes/sample** (compressed) |
| Active series | = metric × label-value combinations — **watch cardinality explosion** |
| Samples/sec | active series ÷ scrape interval |
| Local retention | ~15 days default; long-term → remote store |

---

## 8. Logging
*Shows up in: logging pipeline, audit systems.*

| Metric | Standard |
|---|---|
| Log line size | **~100 B – 1 KB** (structured JSON: 1–2 KB) |
| Compression ratio | **~5–10:1** (gzip on text) |
| Retention tiers | hot (days, fast) → warm → cold (S3/Glacier, cheap) |
| Ingestion unit | GB/day or MB/s — buffer with a queue (Kafka) ahead of indexing |

---

## 9. Distributed tracing
*Shows up in: observability/tracing.*

| Metric | Standard |
|---|---|
| Sampling rate | **0.1–1%** head-based for high volume; tail-based keeps interesting traces |
| Span size | ~few hundred B – 1 KB |
| Trace | tens of spans across services; correlated by trace ID |

---

## 10. Bloom filters (membership / dedup)
*Shows up in: web crawler URL dedup, cache filtering, KV store read optimization.*

| Metric | Standard |
|---|---|
| Bits per element (1% FPR) | **~9.6 bits** |
| Bits per element (0.1% FPR) | **~14.4 bits** |
| Optimal hash count k | (m/n) × ln 2 |
| Property | no false negatives; false positives only; can't delete (use counting BF) |

---

## 11. Message queue
*Shows up in: distributed message queue, event streaming, log pipeline backbone.*

| Metric | Standard (Kafka-style) |
|---|---|
| Per-partition throughput | **~10 MB/s** (conservative planning floor; often higher) |
| Message size | **~1 MB** default max |
| Replication factor | **3** |
| Retention | time-based (**7 days** default) or size-based |
| Consumer parallelism | **bounded by partition count** — one consumer per partition per group |
| Ordering | guaranteed **within a partition**, not across |

---

## 12. Web crawler politeness
*Shows up in: web crawler, scraper, any polite external fetcher.*

| Metric | Standard |
|---|---|
| Per-domain rate | respect `robots.txt` crawl-delay; **~1 req/sec/domain** typical |
| Dedup | bloom filter over seen URLs |
| Frontier | priority queue per domain; politeness + freshness scheduling |
| Scale reference | open web ≈ **billions of pages** |

---

## 13. Email
*Shows up in: email system.*

| Metric | Standard |
|---|---|
| Attachment cap | **25 MB** (Gmail send) |
| Protocols | SMTP (send), IMAP/POP (retrieve) |
| Mailbox | GBs/user; metadata-heavy (headers, threading, flags) |

---

## 14. Server & network limits
*Shows up in: any high-connection service — gateways, LBs, chat, streaming.*

| Metric | Limit |
|---|---|
| TCP ports per IP | **65,535** (16-bit) — bounds concurrent connections per (src,dst) tuple |
| File descriptors | default soft **1,024**; tune to 100k–1M for high-concurrency servers |
| Concurrency milestones | **C10K** (10k conns) → **C10M** (10M conns) |
| TLS handshake | +1 RTT (TLS 1.3) / +2 RTT (TLS 1.2) on connection setup |
| Same-DC round trip | ~0.5 ms (see latency table in cheatsheet) |

---

*Keep appending: when a new design surfaces a durable protocol cap or canonical default, add it to the matching section — labelled **default / typical**, with the knob named, never a bare figure.*
