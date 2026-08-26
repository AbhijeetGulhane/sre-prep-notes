# System Design — Quick Reference

The single reference for capacity estimation and standard limits, built for a 5-minute glance before an interview and fast lookup during one.

**Structure:** two parts, because they hold different kinds of number. **Part 1** = ballparks you *compute with* (do arithmetic). **Part 2** = fixed constants you *recall* (protocol caps, encoding capacities, defaults). **Worked chains** at the end apply both.

---

## Contents

**[How to use this reference](#how-to-use-this-reference)** · **[The method](#the-method)**

*Part 1 — compute with:* [Scale and time shortcuts](#scale-and-time-shortcuts) · [Latency numbers](#latency-numbers) · [Throughput ceilings](#throughput-ceilings) · [Storage: bytes per thing](#storage-bytes-per-thing) · [Memory and cache](#memory-and-cache) · [Bandwidth](#bandwidth) · [Availability: the nines](#availability-the-nines) · [Common multipliers](#common-multipliers)

*Part 2 — recall:* [ID generation and encoding](#id-generation-and-encoding) · [Consistency, replication and quorum](#consistency-replication-and-quorum) · [Consistent hashing](#consistent-hashing) · [Datastore hard limits](#datastore-hard-limits) · [Geospatial](#geospatial) · [Media and video](#media-and-video) · [Metrics and time-series](#metrics-and-time-series) · [Logging](#logging) · [Distributed tracing](#distributed-tracing) · [Bloom filters](#bloom-filters) · [Message queue](#message-queue) · [Web crawler politeness](#web-crawler-politeness) · [Email](#email) · [Server and network limits](#server-and-network-limits)

*[Worked chains](#worked-chains)*

---

## How to use this reference

Every figure here is a *planning anchor*, not a spec. State it as a range with a reason ("order of 100k, a conservative floor"), never as a bare fact. A decision built on a soft number is safe when the **margin is large** — 4× over a ceiling that's itself uncertain by 2× still holds; say that out loud. Convert **requests → operations** before comparing to any ceiling (1 request ≠ 1 op). Weld the *why* onto every claim. For Part 2, most constants are configurable defaults — say "default / typical," name the knob, move on.

---

## The method

Back-of-envelope in five moves:

1. **Clarify scale.** DAU/MAU, requests/day, read:write ratio, object sizes, retention. Get these *before* computing — one wrong assumption poisons the chain.
2. **Load → QPS.** requests/day ÷ 86,400. Then **peak = 2–3× average**.
3. **Storage.** objects/day × bytes/object × retention. Add replication (×3) and index/overhead (~30%).
4. **Bandwidth.** QPS × payload, both directions.
5. **Sanity check.** One box or a fleet? Is the bottleneck QPS, storage, or bandwidth? Name which — that's the answer.

---

# Part 1: Numbers you compute with

## Scale and time shortcuts

| Shortcut | Value | Use |
|---|---|---|
| Seconds/day | **86,400 ≈ 10⁵** | requests/day ÷ 10⁵ ≈ QPS |
| Seconds/month | ≈ 2.5 × 10⁶ | |
| Seconds/year | ≈ 3.15 × 10⁷ (**π × 10⁷** mnemonic) | |
| 1 QPS sustained | ≈ 86.4k/day ≈ 2.6M/month ≈ 31.5M/year | reverse-convert totals |
| Peak factor | **peak ≈ 2–3× average** | always size for peak |

**QPS conversions:**

| Traffic | ≈ Average QPS |
|---|---|
| 1M / day | **~12** |
| 10M / day | ~120 |
| 100M / day | ~1,200 |
| 1B / day | **~11,600** |

**Data size units (2ⁿ ≈ 10ᵏ):** KB 2¹⁰≈10³ · MB 2²⁰≈10⁶ · GB 2³⁰≈10⁹ · TB 2⁴⁰≈10¹² · PB 2⁵⁰≈10¹⁵

## Latency numbers

Canonical set (Jeff Dean). Memorize **orders of magnitude and ratios**, not digits — hardware has improved but the ratios are what you reason with.

| Operation | Latency |
|---|---|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 25 ns |
| **Main memory reference** | **100 ns** |
| Compress 1 KB (Snappy) | 3 µs |
| Send 1 KB over 1 Gbps | 10 µs |
| **SSD random read (4 KB)** | **150 µs** |
| Read 1 MB sequentially from memory | 250 µs |
| **Round trip within same DC** | **500 µs (0.5 ms)** |
| Read 1 MB sequentially from SSD | 1 ms |
| **Disk seek** | **10 ms** |
| Read 1 MB sequentially from disk | 20 ms |
| **Round trip CA ↔ Europe** | **150 ms** |

**Ratios to carry:** memory ~**100,000× faster** than a disk seek · SSD random read ~**100× faster** than disk seek · a same-DC round trip (0.5 ms) is ~**300× slower** than a memory read — which is why a Redis hop per request is a real, nameable cost.

## Throughput ceilings

Single-node planning floors, all **workload-dependent** — cheap O(1) ops sit high, heavy queries far below. Scale past with sharding (writes) or replicas (reads).

| Component | Ballpark (single node) | Notes |
|---|---|---|
| App/web server | **~1k–10k QPS** | light work high end; real logic much lower |
| Redis / Memcached | **~100k ops/sec** | 500k+ pipelined + O(1); Redis single-threaded → Lua atomic |
| Relational DB (PG/MySQL) | reads **~5k–10k QPS**, writes **~1k–5k** | reads scale with replicas, writes with sharding |
| Wide-column (Cassandra/Dynamo) | **~10k+ writes/sec/node** | scales ~linearly; write-optimized |
| Kafka | **~100k–1M+ msg/sec/broker** | partition & message-size dependent |
| L7 LB / Nginx | **~10k–50k+ RPS/node** | |
| L4 LB | **~100k+ RPS/node** | dumb + fast, no app-layer inspection |
| Elasticsearch | ~1k–10k queries/sec/node | query-complexity dependent |
| Object store (S3) | **3,500 PUT / 5,500 GET per sec per prefix** | scales with prefixes; capacity ~unlimited |

**The multiplier that trips people:** 1 request ≠ 1 datastore op. Token-bucket check = a Lua read-modify-write (multi-op); fixed window = INCR + EXPIRE (2 ops). Convert RPS → ops *before* dividing by the ceiling.

## Storage: bytes per thing

| Primitive | Size | | Common object | Rough size |
|---|---|---|---|---|
| bool / char (ASCII) | 1 B | | Short URL | ~100 B |
| int | 4 B | | Tweet / short post | ~200 B – 1 KB |
| long / double / timestamp | 8 B | | DB row + metadata | ~100 B – 1 KB |
| UUID | 16 B | | Log line | ~100 B – 1 KB |
| IPv4 / IPv6 | 4 B / 16 B | | Thumbnail | ~10–50 KB |
| Unicode char | 2–4 B | | Web image (compressed) | ~200 KB – 2 MB |
| | | | 1 min video (compressed) | ~10–50 MB |

**Storage math template:** objects/day × bytes/object × 365 × retention_years × **replication (×3)** × **(1 + overhead ≈ 0.3)**.

## Memory and cache

| Metric | Value |
|---|---|
| RAM per box | tens–hundreds of GB (commodity) |
| Cache hit ratio (healthy) | **80–95%** |
| Pareto rule | **hot 20% of data ≈ 80% of traffic** → size cache to the hot set |
| Sharding driver | at small keyspaces (MB), **throughput/availability drives sharding, not memory** |

## Bandwidth

| Link | Throughput |
|---|---|
| 1 Gbps | **125 MB/s** |
| 10 Gbps | 1.25 GB/s |
| 25 / 40 / 100 Gbps | 3.1 / 5 / 12.5 GB/s |

**Bandwidth = QPS × payload.** e.g. 10k RPS × 50 KB ≈ 500 MB/s ≈ **4 Gbps egress** — check against NIC and LB.

## Availability: the nines

| Availability | Downtime / year | Downtime / month |
|---|---|---|
| 99% | 3.65 days | 7.2 hours |
| 99.9% (three nines) | **8.77 hours** | 43.8 min |
| 99.99% (four nines) | **52.6 min** | 4.38 min |
| 99.999% (five nines) | **5.26 min** | 26 sec |

Each nine ≈ **10× less downtime** and a step up in cost/complexity. Know your target before architecting for it.

## Common multipliers

| Factor | Default |
|---|---|
| Replication | **×3** |
| Index / storage overhead | **+~30%** |
| Peak vs average traffic | **×2–3** |
| Read:write ratio (read-heavy) | often **100:1 – 1000:1** — state it, it drives replica vs shard |
| Cache hit ratio | 80–95% |

---

# Part 2: Limits you recall

## ID generation and encoding
*Shows up in: unique-ID generator, URL shortener, sharded primary keys.*

**Snowflake ID (64-bit):**

| Field | Bits | Capacity |
|---|---|---|
| Sign (unused) | 1 | always 0 |
| Timestamp (ms) | 41 | **~69 years** |
| Machine/node ID | 10 | **1,024 nodes** |
| Sequence | 12 | **4,096 IDs / ms / node** |
| **Total** | 64 | **~4M IDs/sec/node** |

| Encoding | Capacity | | Integer type | Max |
|---|---|---|---|---|
| UUID | 128-bit (v4 = 122 random) | | int32 | **~2.15 B** (2³¹−1) |
| base62, 6 chars | 62⁶ ≈ **56.8 B** | | uint32 | ~4.29 B |
| base62, 7 chars | 62⁷ ≈ **3.5 T** | | int64 | **~9.2 × 10¹⁸** |

## Consistency, replication and quorum
*Shows up in: distributed KV store, cache, lock service, message queue.*

| Concept | Standard |
|---|---|
| Replication factor (N) | **3** (Dynamo, Cassandra, Kafka, HDFS) |
| Quorum for strong consistency | **R + W > N** — e.g. N=3, W=2, R=2 |
| Fast writes / slow reads | W=1, R=N |
| Fast reads / slow writes | W=N, R=1 |
| PACELC | if Partition: A or C; **E**lse: **L**atency or **C**onsistency |

## Consistent hashing
*Shows up in: KV store, cache, sharded datastore, load distribution.*

| Metric | Value |
|---|---|
| Virtual nodes per physical node | **~100–256** — smooths distribution, shrinks variance |
| Key remap on node add/remove | **~1/N of keys** (vs `hash % N` remapping ~all) |
| Right tool, right layer | shard the **datastore** (each key → one shard), never pin **traffic** to nodes (breaks statelessness) — same technique, opposite verdict |

## Datastore hard limits
*Shows up in: everywhere you pick a store — know the ceiling first.*

| Store | Hard limit |
|---|---|
| Redis | value/string **512 MB**; up to 2³² keys |
| Memcached | **1 MB** item (default) |
| Kafka | message **~1 MB** default (`message.max.bytes`) |
| DynamoDB | item **400 KB** |
| MongoDB | BSON document **16 MB** |
| Cassandra | partition **< 100 MB / < 100k cells** (practical) |
| PostgreSQL | field 1 GB (TOAST); table 32 TB |
| S3 | object **5 TB**; single PUT 5 GB; multipart above |

## Geospatial
*Shows up in: proximity service, ride-hailing, "nearby" search.*

| Geohash length | Approx cell | Use | | Alternative | Note |
|---|---|---|---|---|---|
| 4 | ~39 × 20 km | city | | S2 (Google) | spherical, no pole distortion |
| 5 | ~4.9 × 4.9 km | district | | H3 (Uber) | hexagonal, uniform neighbors |
| 6 | ~1.2 × 0.6 km | **neighborhood** | | Quadtree | adapts to density |
| 7 | ~153 × 153 m | block | | 1° latitude | ≈ **111 km** |
| 8 | ~38 × 19 m | building | | | |

Each added geohash char shrinks the cell ~×8.

## Media and video
*Shows up in: video platform, streaming, image/CDN service.*

| Resolution | Bitrate | Storage / hour |
|---|---|---|
| 480p | ~1.5 Mbps | ~0.7 GB |
| 720p | ~3 Mbps | ~1.35 GB |
| **1080p** | **~5 Mbps** | **~2.25 GB** |
| 4K (2160p) | ~25 Mbps | ~11 GB |
| Audio | ~128 kbps | — |

Storage = Mbps × 3600 ÷ 8 = GB/hour · HLS/DASH segment **2–10 s** (6 s common) · multiple renditions → **~1.5–2× original** · CDN edge-caches, origin on miss.

## Metrics and time-series
*Shows up in: monitoring/alerting, observability.*

| Metric | Standard |
|---|---|
| Scrape interval | **15 s** default (10–60 s) |
| Sample size on disk | **~1–2 bytes/sample** (compressed) |
| Active series | metric × label-value combos — **watch cardinality explosion** |
| Samples/sec | active series ÷ scrape interval |
| Local retention | ~15 days default; long-term → remote store |

## Logging
*Shows up in: logging pipeline, audit systems.*

| Metric | Standard |
|---|---|
| Log line size | **~100 B – 1 KB** (structured JSON: 1–2 KB) |
| Compression ratio | **~5–10:1** (gzip on text) |
| Retention tiers | hot (days) → warm → cold (S3/Glacier) |
| Ingestion | GB/day or MB/s — buffer with a queue ahead of indexing |

## Distributed tracing
*Shows up in: observability/tracing.*

| Metric | Standard |
|---|---|
| Sampling rate | **0.1–1%** head-based; tail-based keeps interesting traces |
| Span size | ~few hundred B – 1 KB |
| Trace | tens of spans across services, correlated by trace ID |

## Bloom filters
*Shows up in: web crawler URL dedup, cache filtering, KV read optimization.*

| Metric | Standard |
|---|---|
| Bits/element (1% FPR) | **~9.6 bits** |
| Bits/element (0.1% FPR) | **~14.4 bits** |
| Optimal hash count k | (m/n) × ln 2 |
| Property | no false negatives; can't delete (use counting BF) |

## Message queue
*Shows up in: distributed message queue, event streaming, log backbone.*

| Metric | Standard (Kafka-style) |
|---|---|
| Per-partition throughput | **~10 MB/s** (conservative floor; often higher) |
| Message size | **~1 MB** default max |
| Replication factor | **3** |
| Retention | time-based (**7 days** default) or size-based |
| Consumer parallelism | **bounded by partition count** (one consumer/partition/group) |
| Ordering | guaranteed **within a partition**, not across |

## Web crawler politeness
*Shows up in: web crawler, scraper, polite external fetcher.*

| Metric | Standard |
|---|---|
| Per-domain rate | respect `robots.txt` crawl-delay; **~1 req/sec/domain** typical |
| Dedup | bloom filter over seen URLs |
| Frontier | priority queue per domain (politeness + freshness) |
| Scale reference | open web ≈ **billions of pages** |

## Email
*Shows up in: email system.*

| Metric | Standard |
|---|---|
| Attachment cap | **25 MB** (Gmail send) |
| Protocols | SMTP (send), IMAP/POP (retrieve) |
| Mailbox | GBs/user; metadata-heavy (headers, threading, flags) |

## Server and network limits
*Shows up in: high-connection services — gateways, LBs, chat, streaming.*

| Metric | Limit |
|---|---|
| TCP ports per IP | **65,535** (16-bit) — bounds conns per (src,dst) tuple |
| File descriptors | default soft **1,024**; tune to 100k–1M |
| Concurrency milestones | **C10K** → **C10M** |
| TLS handshake | +1 RTT (TLS 1.3) / +2 RTT (TLS 1.2) |
| Same-DC round trip | ~0.5 ms (see latency table) |

---

## Worked chains

The pattern end to end — method + numbers applied.

### URL shortener
- **Assume:** 100M new URLs/day, read:write = 100:1, 5-year retention.
- **Write QPS:** 100M ÷ 10⁵ ≈ **1,150/s** → peak ~2.3k/s.
- **Read QPS:** ×100 ≈ **115k/s** → cache + read replicas.
- **Storage:** 100M × 365 × 5 ≈ 1.8×10¹¹ URLs × ~500 B ≈ **~90 TB** → ×3 ≈ 270 TB.
- **Key length:** base62⁷ ≈ 3.5T >> 1.8×10¹¹ needed → **7 chars suffice**.
- **Bottleneck verdict:** read QPS → cache + replicas; storage large but cheap.

### News feed / timeline
- **Assume:** 200M DAU, 10 feed-opens/day each.
- **Read QPS:** 2B/day ÷ 10⁵ ≈ **~23k/s** → peak ~50k/s.
- **Key decision:** fan-out-on-write (fast reads, write amplification for celebrities) vs fan-out-on-read (cheap writes, expensive reads). State the tradeoff — that's the interview.
- **Bottleneck verdict:** read latency at fan-out → hybrid (push for most, pull for high-follower accounts).

---

*Keep appending here: each durable number goes in its matching section — Part 1 if you compute with it, Part 2 if you recall it — always **value + terse caveat**, never a naked figure.*
