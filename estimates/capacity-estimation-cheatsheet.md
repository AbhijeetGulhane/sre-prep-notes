# Capacity Estimation — Quick Reference

**Purpose:** the numbers to know cold and the method to apply them, built for a 5-minute glance before an interview. Pairs with `capacity-estimation-anchors.md` (design-specific numbers from individual drills); this file is the general reference.

> **How to state a number in the interview.** Every figure here is a *planning anchor*, not a spec. Say it as a range with a reason ("order of 100k, a conservative floor"), never as a bare fact. A decision built on a soft number is safe when the **margin is large** — 4× over a ceiling that's itself uncertain by 2× still holds; say that out loud. Convert **requests → operations** before comparing to any ceiling (1 request ≠ 1 op). And weld the *why* onto every claim.

---

## 1. The method — back-of-envelope in 5 moves

1. **Clarify scale.** DAU/MAU, requests/day, read:write ratio, object sizes, retention. Get these *before* computing anything — one wrong assumption poisons the chain.
2. **Load → QPS.** requests/day ÷ 86,400. Then **peak = 2–3× average**.
3. **Storage.** objects/day × bytes/object × retention. Add replication (×3) and index/overhead (~30%).
4. **Bandwidth.** QPS × payload size, both directions.
5. **Sanity check.** Does it fit one box or need a fleet? Is the bottleneck QPS, storage, or bandwidth? State which — that's the answer the interviewer wants.

---

## 2. Scale & time shortcuts (the arithmetic that makes it fast)

| Shortcut | Value | Use |
|---|---|---|
| Seconds/day | **86,400 ≈ 10⁵** | requests/day ÷ 10⁵ ≈ QPS |
| Seconds/month | ≈ 2.5 × 10⁶ | |
| Seconds/year | ≈ 3.15 × 10⁷ (**π × 10⁷** mnemonic) | |
| 1 QPS sustained | ≈ 86.4k/day ≈ 2.6M/month ≈ 31.5M/year | reverse-convert totals |
| Peak factor | **peak ≈ 2–3× average** | always size for peak |

**Quick QPS conversions:**

| Traffic | ≈ Average QPS |
|---|---|
| 1M / day | **~12** |
| 10M / day | ~120 |
| 100M / day | ~1,200 |
| 1B / day | **~11,600** |

**Data size units (2ⁿ ≈ 10ᵏ):**

| | Bytes | ≈ |
|---|---|---|
| KB | 2¹⁰ | 10³ |
| MB | 2²⁰ | 10⁶ |
| GB | 2³⁰ | 10⁹ |
| TB | 2⁴⁰ | 10¹² |
| PB | 2⁵⁰ | 10¹⁵ |

---

## 3. Latency numbers every engineer should know

The canonical set (Jeff Dean). Memorize the **orders of magnitude**, not the digits — hardware has improved, but the *ratios* are what you reason with.

| Operation | Latency | Human-scale intuition |
|---|---|---|
| L1 cache reference | 0.5 ns | |
| Branch mispredict | 5 ns | |
| L2 cache reference | 7 ns | |
| Mutex lock/unlock | 25 ns | |
| **Main memory reference** | **100 ns** | the baseline "fast" |
| Compress 1 KB (Zippy/Snappy) | 3 µs | |
| Send 1 KB over 1 Gbps | 10 µs | |
| **SSD random read (4 KB)** | **150 µs** | |
| Read 1 MB sequentially from memory | 250 µs | |
| **Round trip within same DC** | **500 µs (0.5 ms)** | the network-hop cost you pay per remote call |
| Read 1 MB sequentially from SSD | 1 ms | |
| **Disk seek** | **10 ms** | ~100× slower than SSD read |
| Read 1 MB sequentially from disk | 20 ms | |
| **Round trip CA ↔ Europe** | **150 ms** | cross-continent; why you put data near users |

**Ratios to carry in your head:** memory is ~**100,000× faster** than a disk seek. SSD random read is ~**100× faster** than disk seek. A same-DC round trip (0.5 ms) is ~**300× slower** than a memory read — which is exactly why a Redis hop per request is a real cost worth naming.

---

## 4. Throughput ceilings by component

Single-node planning floors. All **workload-dependent** — cheap O(1) ops sit at the high end, heavy queries far below. Scale past these with sharding (writes) or replicas (reads).

| Component | Ballpark (single node) | Notes |
|---|---|---|
| App/web server | **~1k–10k QPS** | light work high end; real business logic much lower |
| Redis / Memcached (in-memory) | **~100k ops/sec** | up to 500k+ pipelined + O(1); single-threaded (Redis) → Lua runs atomically |
| Relational DB (Postgres/MySQL) | reads **~5k–10k QPS**, writes **~1k–5k** | scale reads with replicas; writes with sharding |
| Wide-column (Cassandra/Dynamo) | **~10k+ writes/sec/node** | scales ~linearly with nodes; write-optimized |
| Kafka | **~100k–1M+ msg/sec/broker** | partition- and message-size-dependent; throughput-oriented |
| L7 LB / Nginx | **~10k–50k+ RPS/node** | |
| L4 LB | **~100k+ RPS/node** | dumb + fast; no app-layer inspection |
| Elasticsearch | ~1k–10k queries/sec/node | query-complexity dependent |
| Object store (S3) | **3,500 PUT / 5,500 GET per sec per prefix** | capacity effectively unlimited; scale with more prefixes |

**The multiplier that trips people:** 1 request ≠ 1 datastore op. A token-bucket check = a Lua read-modify-write (multi-op); a fixed window = INCR + EXPIRE (2 ops). Convert RPS → ops *before* dividing by the ceiling.

---

## 5. Storage — bytes per thing

| Primitive | Size |
|---|---|
| bool / char (ASCII) | 1 B |
| int | 4 B |
| long / double / timestamp | 8 B |
| UUID | 16 B |
| IPv4 / IPv6 | 4 B / 16 B |
| Unicode char | 2–4 B |

| Common object | Rough size |
|---|---|
| Short URL | ~100 B |
| Tweet / short post (text) | ~200 B – 1 KB |
| DB row + light metadata | ~100 B – 1 KB |
| Log line | ~100 B – 1 KB |
| Thumbnail image | ~10–50 KB |
| Web image (compressed) | ~200 KB – 2 MB |
| 1 min of video (compressed) | ~10–50 MB |

**Storage math template:** objects/day × bytes/object × 365 × retention_years × **replication (×3)** × **(1 + index/overhead ≈ 0.3)**.

---

## 6. Memory & cache

| Metric | Value |
|---|---|
| RAM per box | tens–hundreds of GB (commodity) |
| Cache hit ratio (healthy) | **80–95%** |
| Pareto rule | **hot 20% of data ≈ 80% of traffic** → size cache to the hot set, not the whole keyspace |
| Sharding driver | at small keyspaces (MB), throughput/availability drives sharding — **not** memory |

---

## 7. Bandwidth

| Link | Throughput |
|---|---|
| 1 Gbps | **125 MB/s** |
| 10 Gbps | 1.25 GB/s |
| 25 / 40 / 100 Gbps | 3.1 / 5 / 12.5 GB/s |

**Bandwidth = QPS × payload.** e.g. 10k RPS × 50 KB response ≈ 500 MB/s ≈ **4 Gbps egress** — check it against your NIC and your LB.

---

## 8. Availability — the nines

| Availability | Downtime / year | Downtime / month |
|---|---|---|
| 99% | 3.65 days | 7.2 hours |
| 99.9% ("three nines") | **8.77 hours** | 43.8 min |
| 99.99% ("four nines") | **52.6 min** | 4.38 min |
| 99.999% ("five nines") | **5.26 min** | 26 sec |

Each nine ≈ **10× less downtime** and typically a big step up in cost/complexity. Know your target before you architect for it.

---

## 9. Common multipliers & factors

| Factor | Default |
|---|---|
| Replication | **×3** |
| Index / storage overhead | **+~30%** |
| Peak vs average traffic | **×2–3** |
| Read:write ratio (read-heavy systems) | often **100:1 – 1000:1** — state it, it drives replica vs shard |
| Cache hit ratio | 80–95% |

---

## 10. Worked chains (the pattern, end to end)

### URL shortener
- **Assume:** 100M new URLs/day, read:write = 100:1, 5-year retention.
- **Write QPS:** 100M ÷ 10⁵ ≈ **1,150/s** → peak ~2.3k/s.
- **Read QPS:** ×100 ≈ **115k/s** → cache + read replicas territory.
- **Storage:** 100M/day × 365 × 5 ≈ 1.8×10¹¹ URLs × ~500 B ≈ **~90 TB** raw → ×3 replication ≈ 270 TB.
- **Cache:** hot 20% of daily reads → size to the working set, not the full 90 TB.
- **Bottleneck verdict:** read QPS, solved by cache + replicas; storage is large but cheap.

### News feed / timeline
- **Assume:** 200M DAU, each opens feed 10×/day.
- **Read QPS:** 2B reads/day ÷ 10⁵ ≈ **~23k/s** → peak ~50k/s.
- **Key decision:** fan-out-on-write (precompute feeds, fast reads, heavy write amplification for celebrities) vs fan-out-on-read (cheap writes, expensive reads). State the tradeoff — that's the interview.
- **Bottleneck verdict:** read latency at fan-out; hybrid (push for most, pull for high-follower accounts).

---

## 11. Rate-limiter anchors (companion file)

Design-specific numbers — Redis Lua atomicity, token-bucket R/C rules, fail-open timeout budget, 429/Retry-After — live in `capacity-estimation-anchors.md`. The generalizable ones (Redis ~100k ops/sec, request→ops multiplier, ~100 B/key) are already folded into the tables above.

---

*Keep appending: each new design that surfaces a durable number (a new component ceiling, a new per-object size) goes into the matching section above — always as **value + terse caveat**, never a naked figure.*
