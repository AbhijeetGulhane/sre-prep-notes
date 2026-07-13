# URL Shortener — NALSD Design (Google Scale)

NALSD = Needs, Approach, Layers, Scaling, Deployment/Discuss. This walkthrough follows that shape but keeps your six required sections front and center.

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- `shorten(long_url) -> short_url` — given a long URL, return a short code.
- `redirect(short_code) -> long_url` — given a short code, redirect the client.
- Optional: custom aliases, expiration, click analytics, abuse/malware checks.

### Non-functional requirements
- **Availability > Consistency** for the redirect path (an outage on redirects is visible to the entire internet; a few seconds of staleness on a brand-new link is invisible). This is the single most important design decision and it drives DB choice, caching, and replication strategy below.
- Low latency redirects (sub-100ms p99, ideally single-digit ms from cache/CDN).
- Short codes must be unique, non-guessable-enough for security-sensitive use cases (not a strict requirement, but worth flagging).
- Durable — a shortened link must not be lost.

### Scale target (given: 100B URLs)
Assume the system is designed for a **10-year horizon** holding 100B total URLs.

**Writes (new shortens):**
- 100B URLs / 10 years ≈ 10B/year ≈ 27M/day ≈ **~317 writes/sec average**
- Real traffic is bursty (viral campaigns, marketing pushes) → assume 5x peak factor → **~1,500 writes/sec peak**

**Reads (redirects):**
- URL shorteners are extremely read-heavy. Industry rule of thumb is **100:1 to 1000:1** read:write. Use a conservative 100:1 for capacity planning, and call out that some products (bit.ly-like) see 1000:1.
- Average: 317 × 100 ≈ **~31,700 reads/sec average**
- Peak (5x): **~150,000 reads/sec peak**

This read/write asymmetry is the single biggest driver of the architecture: **cache-first, read-optimized, eventually-consistent** system.

**Storage:**
- Per-record size estimate: short_code (7B) + long_url (avg ~100–500B) + metadata (created_at, user_id, expiry, flags) ≈ **~500 bytes/record**
- 100B × 500B = **50 TB raw**
- With replication factor 3 (standard for durability across zones): **~150 TB total**
- This is trivial for a modern distributed store — storage is *not* the bottleneck; QPS and hot-key fan-out are.

**Bandwidth:**
- Peak reads: 150,000 QPS × ~500B response ≈ **~75 MB/s** at peak (before CDN offload — with CDN this drops by an order of magnitude or more for hot links)
- Writes: 1,500 QPS × 500B ≈ **~750 KB/s** — negligible

**Key space check (do this early, it determines short_code length):**
- Base62 alphabet `[a-zA-Z0-9]`, 62 characters
- 62^6 ≈ 56.8B — *not enough* headroom for 100B with growth margin
- 62^7 ≈ 3.52 trillion — comfortably supports 100B+ with orders of magnitude of headroom
- **Decision: 7-character short codes.**

---

## 2. API Design

```
POST /api/v1/shorten
Request:
{
  "long_url": "https://example.com/some/very/long/path?query=1",
  "custom_alias": "optional-string",     // optional
  "expires_at": "2027-01-01T00:00:00Z",  // optional
  "user_id": "optional-for-auth'd users"
}
Response: 201 Created
{
  "short_url": "https://sho.rt/aZ3kX9q",
  "short_code": "aZ3kX9q",
  "long_url": "https://example.com/...",
  "created_at": "...",
  "expires_at": "..."
}

GET /{short_code}
-> 301 or 302 redirect to long_url (see Section 5)
-> 404 if not found / expired
-> 410 Gone if intentionally deactivated

GET /api/v1/analytics/{short_code}   (internal/authenticated)
-> click counts, referrers, geo breakdown, timestamps

DELETE /api/v1/urls/{short_code}     (auth'd owner only)
```

Design notes:
- Shorten is a low-QPS write path — can afford stronger consistency and more validation (malicious URL scanning, rate limiting per user/IP).
- Redirect is the hot path — must be minimal-hop: edge cache → app cache → DB, in that priority order.
- Idempotency: if the same long_url is submitted twice by the same user, decide whether to dedupe (return existing short_code) or always mint new — dedup requires a `long_url_hash -> short_code` reverse index (see schema).

---

## 3. Database Schema & Choice

### Schema (logical, store-agnostic)
```
Table: urls
  short_code   VARCHAR(7)   PK
  long_url     TEXT
  user_id      BIGINT       (nullable, indexed)
  created_at   TIMESTAMP
  expires_at   TIMESTAMP    (nullable)
  is_active    BOOLEAN
  click_count  BIGINT       (denormalized counter, see analytics note below)

Table: url_reverse_index   (for dedup, optional)
  long_url_hash  VARCHAR(64)  PK   -- SHA-256 of normalized long_url
  short_code     VARCHAR(7)
```

### Access pattern analysis
- 100% of redirect traffic is a **point lookup by short_code** (primary key, no range scans, no joins).
- Writes are simple inserts, no complex transactions needed.
- No need for multi-row ACID transactions across URLs.

This access pattern is the textbook case for a **key-value / wide-column NoSQL store** over a relational database:

| Requirement | Relational (sharded MySQL/Postgres) | NoSQL (Cassandra / DynamoDB / Bigtable) |
|---|---|---|
| Point lookups by PK at 150K+ QPS | Needs manual sharding, connection pooling gets painful at this scale | Native horizontal scaling, built for this |
| Write availability during node failure | Harder — typically leader-based, failover has downtime window | Tunable consistency (e.g., quorum writes), multi-master, no single leader |
| Schema flexibility (adding click metadata later) | Migrations are painful at 100B rows | Schema-less column families |
| Global multi-region | Possible but heavy (Spanner-style, or manual multi-master) | Designed for it (Cassandra multi-DC, DynamoDB Global Tables, Bigtable regional replication) |

**Decision: Cassandra or Bigtable-style wide-column store**, partitioned by `short_code` (or a hash of it, to avoid hot ranges — see below). This is consistent with actual industry practice (bit.ly-scale systems and Google-internal systems like Bitly/Goo.gl used exactly this kind of key-value backend).

If the team strongly prefers relational for tooling/ops maturity reasons, a **sharded MySQL/Postgres with consistent-hash sharding on short_code** is a legitimate fallback — call this out as a trade-off discussion point in an interview: NoSQL wins on horizontal write/read scaling and multi-region story; relational wins on tooling, ad-hoc querying, and operational familiarity if the org already runs it well.

**Reverse index (`long_url -> short_code`) is a separate table**, not a secondary index on the main table — this avoids an expensive scan-heavy index on a huge text column and lets you shard it independently by `hash(long_url)`.

---

## 4. Short URL Generation — Base62 Counter vs. MD5 Truncation

This is the crux design decision. Two candidate approaches:

### Option A: MD5/hash truncation
- Take `MD5(long_url)`, base62-encode the digest, truncate to 7 chars.
- **Problem 1 — collisions at scale.** With 100B entries and a 7-char space of ~3.5T, birthday-paradox collision probability is non-trivial and *grows with every insert*. Every write needs a collision check-and-retry loop against the DB, which adds tail latency and — worse — that latency gets *worse over time* as the table fills. This is a scaling anti-pattern: correctness degrades as a function of table size.
- **Problem 2 — determinism without dedup control.** Same long_url always hashes the same way, which is fine for natural dedup, but if you *want* two different short_codes for the same URL (e.g., separate campaign tracking) you need to salt it, which reopens the collision-checking problem.

### Option B: Counter + Base62 encoding (recommended)
- Maintain a **globally unique, monotonically increasing integer ID**, then Base62-encode it directly to produce the short_code. No hashing, no collisions, no retry loop, ever.
- Example: ID `125_000_000_000` → Base62 encode → deterministic 7-char string, guaranteed unique because the ID itself is guaranteed unique.

**The catch: how do you generate a globally unique counter at 1,500 writes/sec across many app servers without a single point of contention?**

Solution — **pre-allocated ID ranges (ticket server / range allocator pattern)**, the same idea Snowflake and Zookeeper-based ID generators use:
- A small, highly-available coordination service (e.g., backed by Zookeeper, etcd, or a simple atomic-increment row in a small DB) hands out **blocks of IDs** (e.g., 1M at a time) to each application server on startup/exhaustion.
- Each app server then locally increments through its own block with zero coordination overhead for the next ~1M requests.
- If a server crashes with unused IDs in its block, those IDs are simply wasted (never reused) — totally fine given a 3.5T ID space and 100B target usage (~3% utilization even without any waste).

**Verdict: Base62-encoded counter, via a range-allocator ID generator, is the correct choice at this scale.** It trades a small amount of range-allocation infrastructure for the complete elimination of collision-retry latency — and that latency would otherwise be an *increasing function of table size*, which is exactly the kind of degradation you must design out for a 100B-row system.

(Side note for completeness: encoding order can leak information — sequential codes reveal creation order and approximate volume. If unguessability matters, bit-shuffle or XOR the counter with a fixed permutation before Base62 encoding. This adds zero collision risk since it's still a bijection over the same ID space.)

---

## 5. Redirect Flow — 301 vs 302

| | 301 Moved Permanently | 302 Found (Temporary) |
|---|---|---|
| Browser caching | Cached by browser/intermediate proxies — repeat visits **skip your server entirely** | Not cached — **every click hits your server** |
| Server load | Lower — huge win at 150K QPS peak | Higher — every redirect is a real request |
| Analytics accuracy | Poor — you lose visibility into repeat clicks once cached | Accurate — every click is observable (referrer, geo, device, timestamp) |
| Flexibility to change/deactivate a link later | Risky — cached clients may never re-check with your server | Safe — you always control the destination |

**Trade-off is real and this is a legitimate design discussion, not a clear-cut answer:**
- Pure link-shortening-as-infrastructure (no analytics need) → **301** is more scalable, since browser/CDN caching absorbs most repeat traffic for free.
- Product that sells click analytics / needs to revoke or redirect links dynamically (which is the realistic case for a Google/bit.ly-style product) → **302** is standard, precisely so every click is observable and every redirect is authoritative.

**Recommended hybrid for Google scale:** use **302 at the application layer** (so your servers see every click for analytics/abuse detection), but put a **CDN edge cache in front with a short TTL** (e.g., 60s) keyed by short_code. This gets you:
- Near-301 performance for hot/viral links (edge cache absorbs the burst),
- Near-302 analytics fidelity (you still see a click every ~60s worth of aggregated traffic, plenty for reporting and abuse detection, and full accuracy resumes automatically once TTL expires),
- Full control to deactivate/change a link within one TTL window.

This is exactly how most large-scale shorteners actually operate in practice.

---

## 6. Scaling to 100B URLs — Full Architecture

```
                         ┌─────────────┐
Client ──HTTPS──▶  CDN / Edge Cache (redirects, short TTL)
                         └──────┬──────┘
                                │ cache miss
                     ┌──────────▼──────────┐
                     │   Global Load Balancer │
                     └──────────┬──────────┘
                    ┌────────────┼────────────┐
              ┌─────▼─────┐ ┌────▼─────┐ ┌────▼─────┐
              │ App Server │ │App Server│ │App Server│   (stateless, horizontally scaled)
              └─────┬─────┘ └────┬─────┘ └────┬─────┘
                    │             │             │
            ┌───────▼─────────────▼─────────────▼───────┐
            │     Distributed Cache (Redis/Memcached)     │  ← absorbs ~90%+ of read QPS
            │     sharded by short_code, LRU + TTL        │
            └───────────────────┬──────────────────────┘
                                 │ cache miss
            ┌────────────────────▼───────────────────────┐
            │  Wide-column store (Cassandra/Bigtable)      │
            │  sharded/partitioned by hash(short_code)     │
            │  replicated ×3 across zones / regions        │
            └───────────────────────────────────────────────┘

            ┌───────────────────────────────────┐
            │  Range-based ID Allocator service   │  (Zookeeper/etcd backed,
            │  hands out 1M-ID blocks per server  │   used only on the write path)
            └───────────────────────────────────┘
```

**Layer-by-layer scaling decisions:**

1. **CDN / edge caching** — Absorbs the majority of redirect traffic for hot/viral links close to the user, cutting cross-region latency and origin load dramatically. This is the first line of defense against the 150K QPS peak.

2. **Sharding key = hash(short_code), not short_code directly.** Since IDs are sequential (from the counter), sharding directly by short_code would create a hot, moving "last shard" that absorbs 100% of new writes and recent-link reads (recent links are disproportionately clicked). Hashing the key spreads both writes and hot reads evenly across the cluster — this is the standard fix for the sequential-ID hot-shard problem.

3. **Distributed cache (Redis/Memcached) in front of the DB**, sharded independently from the DB (via consistent hashing so adding/removing cache nodes doesn't invalidate the whole cache). At 100:1 read:write ratio with realistic access skew (a small fraction of links get most clicks — classic power law), a cache hit rate of 90%+ is very achievable, meaning the DB only ever sees a fraction of the 150K peak QPS.

4. **Hot-key / viral-link handling.** A single link going viral can dwarf normal traffic (think a link posted from a celebrity account). Mitigations:
   - Multiple cache replicas of the same hot key (client-side or cache-side "hot key splitting") to avoid a single cache node bottleneck.
   - Local in-process (app server) micro-cache with a very short TTL (sub-second) as a last line of defense before hitting the distributed cache.

5. **Multi-region deployment.** Each region has its own full stack (app servers, cache, DB replica) to serve reads locally with low latency. Writes go to the region closest to the user and replicate asynchronously (async replication is acceptable here because a brand-new short link being invisible in another region for a few hundred ms is a non-issue — this is the availability > consistency call from Section 1 paying off).

6. **ID allocator availability.** This is the one component on the write path that must not become a single point of failure. Mitigate by running it as a small, highly-available cluster (Zookeeper/etcd already gives you this), and by handing out large-enough ID blocks (1M+) that allocator calls are rare (~1 per server per few minutes at 1,500 writes/sec spread across many app servers) — so even a brief allocator outage doesn't stall writes.

7. **Analytics path is decoupled.** Click events are fired asynchronously to a queue (Kafka/Pub-Sub) rather than synchronously incrementing a counter in the hot redirect path — this keeps the redirect's critical path to a single cache/DB read with no write-amplification on every click.

---

## 7. Bottlenecks & Likely Follow-Up Questions

- **"What if the cache goes down?"** — DB should be provisioned to survive a full cache-miss storm (thundering herd) at least briefly; use request coalescing (single in-flight DB read per short_code, other concurrent requests wait on it) to avoid duplicate DB load during cache warm-up.
- **"How do you prevent short_code enumeration / scraping all URLs?"** — rate limit by IP/user on the shorten endpoint; consider the bit-shuffle-the-counter trick from Section 4 to make codes non-sequential-looking.
- **"How do you handle expired/deleted links at 100B scale without a full table scan?"** — TTL-based expiry at the storage layer (Cassandra/Bigtable both support native TTL on cells) rather than a batch cleanup job.
- **"Consistency model?"** — explicitly eventual consistency on the redirect path (a link may take up to replication lag, typically ms, to be visible everywhere); this is fine because Section 1 established availability as the priority.
