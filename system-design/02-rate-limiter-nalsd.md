# Rate Limiter — NALSD Design (Google Scale)

*Previous: [01-url-shortener-nalsd.md](./01-url-shortener-nalsd.md) — natural follow-up, since a rate limiter is one of the components you'd bolt onto that shortener's `/shorten` endpoint to stop abuse.*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- Given a **key** (user_id, API key, IP address, or a composite like `user_id+endpoint`), allow or reject a request based on whether it exceeds a configured rate (e.g., 100 requests/minute).
- Return enough information for the client to back off correctly: allowed/denied, remaining quota, reset time.
- Support **different limits for different consumers** (free tier vs paid tier, per-endpoint limits).

### Non-functional requirements
- **Low added latency** — a rate limiter sits on the hot path of *every* request in the system it protects, so it must add single-digit milliseconds, not tens.
- **Correctness under concurrency** — the classic failure mode is a race condition that lets a client burst past its limit when many requests hit multiple servers simultaneously (the "check-then-increment" race, covered in Section 5).
- **Fail-open vs fail-closed decision, made explicit up front:** if the rate limiter's backing store is unavailable, do you let traffic through (fail-open, prioritizing availability) or block it (fail-closed, prioritizing protection)? For most consumer-facing APIs, **fail-open** is the right default — a rate limiter outage should degrade you to "unprotected" not "fully down," since the limiter is a defense-in-depth layer, not the primary system.
- Distributed — must work correctly across a fleet of stateless app servers, not just a single machine.

### Scale target
Assume this rate limiter protects a Google-scale API surface (analogous QPS to the shortener's redirect path).

- **Requests to check: ~150,000 QPS peak** (reuse the redirect-path peak from the shortener doc, since a rate limiter would sit in front of exactly that kind of hot path).
- **Unique keys tracked:** assume 500M active users/API keys, each potentially rate-limited per-endpoint → realistically **a few billion active (key, window) counters** at any given time if you're limiting per-user-per-endpoint-per-minute.
- **Storage per key:** a counter is tiny — 8 bytes for a count, plus a timestamp/TTL, plus key overhead ≈ **~50–100 bytes per active key** in a store like Redis.
  - Billions of keys × 100 bytes ≈ **hundreds of GB** — this fits comfortably in an in-memory store's aggregate cluster memory, which is why Redis (not a disk-backed DB) is the standard choice here — see Section 4.
- **Write amplification:** every single incoming request is *both* a read (what's the current count?) and a write (increment it) against the limiter's backing store. At 150K QPS this means the backing store needs to sustain **~150K+ ops/sec of atomic read-modify-write operations**, not just simple reads. This is the single biggest capacity number driving the design — it's why naive "SELECT then UPDATE" against a relational DB falls over immediately at this scale, and why the design converges on an in-memory store with atomic primitives (Section 5).

---

## 2. API Design

The rate limiter is typically not a public-facing API in itself — it's a library/middleware or a sidecar that the API gateway or each service calls internally. But it's useful to define its contract explicitly:

```
Internal call: check_and_increment(key, limit, window_seconds) -> RateLimitResult

RateLimitResult:
{
  "allowed": true,
  "limit": 100,
  "remaining": 37,
  "reset_at": "2026-07-14T10:31:00Z"   // when the window resets
}
```

On the client-facing HTTP response, this surfaces as standard headers so clients can self-throttle:
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1752489060
Retry-After: 42
```

Design notes:
- `429` is the correct status code (not `403` — this is a rate condition, not a permissions issue), and `Retry-After` is what well-behaved clients (and your own internal service-to-service callers) use to back off without hammering you immediately after the window resets.
- Keying strategy matters: rate limit by `user_id` when authenticated (fair regardless of how many devices/IPs a user has), fall back to `IP` for unauthenticated traffic, and often layer **both** a per-user and a global per-IP limit to catch abuse that spreads across many fake accounts from one source.

---

## 3. Algorithm Choice — Token Bucket vs Leaky Bucket vs Sliding Window

This is the crux decision, same role as Base62-vs-MD5 was in the shortener doc.

### Token Bucket
- A bucket holds up to `capacity` tokens. Tokens refill at a fixed rate (e.g., 10/sec). Each request consumes 1 token; if the bucket is empty, reject.
- **Key property: allows bursts.** If a client has been idle and tokens have accumulated up to capacity, it can fire a burst of requests all at once, as long as it stays within the average rate over time.
- This matches real client behavior well — a mobile app that's been backgrounded and then does a burst of sync calls on foreground shouldn't get needlessly throttled if it's still within its longer-term budget.

### Leaky Bucket
- Requests enter a queue (the "bucket") and are processed ("leaked") at a fixed, constant rate, regardless of arrival burstiness. If the queue is full, new requests are dropped.
- **Key property: smooths output to a strictly constant rate.** No bursts get through, ever — even legitimate ones.
- This is the right choice when the *downstream* system genuinely cannot handle variable load (e.g., protecting a fixed-capacity legacy system or a third-party API with a hard constant-rate contract), because it converts bursty input into constant output.
- Trade-off: worse UX for legitimate bursty clients, and it requires an actual queue (more implementation complexity / more state) rather than just a counter.

### Sliding Window (Log or Counter variant)
- **Sliding window log:** store a timestamp for every request in the window; count entries in the last `window_seconds` on each check. Perfectly accurate, but **storage grows with request volume** — at 150K QPS, storing a timestamp per request per key is expensive (this is a real scaling problem, not a theoretical one).
- **Sliding window counter (the practical version):** keep two fixed counters — current window and previous window — and compute a weighted estimate: `current_count + previous_count × (overlap fraction)`. This approximates a true sliding window with **fixed, small storage** (just two integers per key) and is what's actually used in production systems.
- **Key property: most accurate rate enforcement, no edge burst exploit.** Fixed-window counters (a simpler, naive approach not discussed above) have a known flaw — a client can send `limit` requests in the last second of one window and another `limit` requests in the first second of the next window, getting `2×limit` through in a 2-second span. The sliding window counter closes this gap.

### Decision matrix

| | Bursts allowed | Output smoothness | Storage cost | Accuracy | Implementation complexity |
|---|---|---|---|---|---|
| Token bucket | Yes (up to capacity) | Bursty | O(1) per key | Approximate but well-behaved | Low |
| Leaky bucket | No | Perfectly smooth | O(queue depth) per key | Exact for output rate | Medium (needs a queue) |
| Sliding window log | N/A (exact) | N/A | O(requests) per key — grows with traffic | Exact | Medium-high |
| Sliding window counter | Slight (bounded) | Smooth-ish | O(1) per key | Very close approximation | Medium |

**Recommendation for a general-purpose, Google-scale API gateway rate limiter: Token bucket for the primary per-client limit** (it matches real traffic patterns and legitimate burst behavior, and it's the cheapest to implement and reason about at 150K+ QPS with O(1) state per key), **layered with a sliding window counter for stricter, security-sensitive limits** (e.g., login attempts, password reset requests) where the fixed-window edge-burst exploit is a real risk worth the extra implementation cost.

Leaky bucket is the right call specifically when you're **shaping output toward a downstream system with a hard constant-rate ceiling** (e.g., calling a third-party API with a strict rate contract) rather than protecting your own API from clients — worth stating explicitly in an interview, since "which algorithm is best" without naming what you're protecting is an incomplete answer.

---

## 4. Placement — Where in the Stack Does This Live?

Four candidate placements, not mutually exclusive:

| Placement | Pros | Cons |
|---|---|---|
| **Client-side** | Zero server cost, immediate feedback | Trivially bypassable — never sufficient alone, only a UX nicety |
| **Edge / CDN layer** | Stops abuse before it reaches your infrastructure at all — cheapest place to reject traffic | Coarse-grained (usually IP-based only, no user-identity awareness at the edge) |
| **API Gateway** | Central choke point, sees all traffic, can apply per-user/per-endpoint policy consistently | Gateway itself must scale to full traffic volume; becomes a critical dependency |
| **Per-service (sidecar/middleware)** | Fine-grained, service-specific limits; survives even if the gateway's global limiter has a gap | Distributed state — every service instance needs a consistent view of the same counters, which is the hard distributed-systems problem this design has to solve |

**Recommended layering for Google scale — defense in depth, cheapest checks first:**
1. **Edge/CDN**: coarse IP-based limiting, rejects obvious floods before they cost you any compute.
2. **API Gateway**: the primary, authoritative per-user/per-API-key rate limit, backed by the shared Redis cluster (Section 5) — this is where the token bucket algorithm from Section 3 actually runs.
3. **Per-service, optional**: an additional, stricter limit on a specific expensive endpoint (e.g., a search endpoint that hits a slow backend), independent of the general gateway limit.

The gateway layer is where most of the engineering complexity lives, so that's the focus of Section 5.

---

## 5. Redis INCR/EXPIRE Pattern — Implementation & the Race Condition

### The naive pattern
```
key = "ratelimit:{user_id}:{endpoint}:{window_start}"
count = INCR(key)          -- atomically increments, creates key at 1 if absent
if count == 1:
    EXPIRE(key, window_seconds)   -- set TTL only on first increment in this window
if count > limit:
    reject
else:
    allow
```

**Why `INCR` and not `GET` then `SET`:** `INCR` is a single atomic Redis command. If you instead did `GET key`, computed `value + 1` in your app code, then `SET key value+1`, you'd have a classic **check-then-act race condition** — two concurrent requests from the same client, hitting two different app servers at the exact same moment, could both `GET` the same pre-increment value, both compute "I'm still under the limit," and both `SET` the same incremented value — letting a client burst past its limit exactly when it's under the most concurrent load (which, not coincidentally, is exactly when you most need the limiter to work). `INCR` sidesteps this because Redis executes it atomically server-side — there's no read-modify-write gap for a race to occur in.

### The subtler race: the `INCR` + `EXPIRE` pair itself is not atomic
The two-line pattern above still has a tiny window between `INCR` and `EXPIRE` where, in a pathological case, a key could exist without a TTL if the process crashes between the two calls (leaking a counter that never expires). The fix, and the actual production-grade pattern, is a **Lua script executed atomically via `EVAL`**:

```lua
-- rate_limit.lua
local current = redis.call("INCR", KEYS[1])
if tonumber(current) == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
end
return current
```
Redis executes the entire Lua script as a single atomic operation — no other command can interleave between the `INCR` and the `EXPIRE`, closing the gap completely. This is the standard production pattern (used by things like the `redis-cell` module and most hand-rolled gateway limiters) rather than two separate round-trips from the app server.

### Fixed window vs the sliding window counter, implemented
The pattern above implements a **fixed window** (`window_start` bucketed to, say, the current minute) — simple, but has the edge-burst exploit from Section 3. To upgrade to the **sliding window counter**, keep two keys (current and previous window) and combine them at read time:

```
current_key  = "ratelimit:{key}:{current_window}"
previous_key = "ratelimit:{key}:{previous_window}"

current_count  = GET(current_key)  or 0
previous_count = GET(previous_key) or 0
elapsed_fraction = (now - current_window_start) / window_seconds

estimated_count = current_count + previous_count * (1 - elapsed_fraction)

if estimated_count >= limit: reject
else: INCR(current_key) via the same atomic Lua pattern above
```

### Distributed correctness: why Redis (and not per-server local counters)
If each app server kept its own local in-memory counter, a client hitting `N` different app servers behind a load balancer would effectively get `N × limit` — the limiter would silently fail exactly when you have the most horizontal scale, which is backwards. **A shared, external, atomic counter store is non-negotiable for a correct distributed rate limiter** — this is why Redis (single-threaded per-shard, so per-key operations serialize naturally) is the standard backbone, not a coincidence of popularity.

---

## 6. Scaling to Google Scale (150K+ ops/sec)

```
Client ──▶ CDN (coarse IP limiting) ──▶ Load Balancer
                                              │
                          ┌───────────────────┼───────────────────┐
                    ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
                    │API Gateway │       │API Gateway │       │API Gateway │   (stateless)
                    │ + limiter  │       │ + limiter  │       │ + limiter  │
                    │  middleware│       │  middleware│       │  middleware│
                    └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
                          └───────────────────┼───────────────────┘
                                    ┌──────────▼──────────┐
                                    │   Redis Cluster       │
                                    │   sharded by hash(key)│
                                    │   (Redis Cluster mode │
                                    │    or client-side      │
                                    │    consistent hashing) │
                                    └───────────────────────┘
```

1. **Redis Cluster, sharded by `hash(key)`.** A single Redis instance tops out well below 150K ops/sec of atomic increments once you account for network overhead and other traffic sharing the box — so the counter keyspace is partitioned across many Redis shards using consistent hashing on the rate-limit key, same sharding principle as the shortener's DB layer.

2. **Local, short-TTL caching of "definitely allowed" decisions at the gateway**, as an optimization: if a key is nowhere near its limit (e.g., under 50% utilized), the gateway can cache that fact locally for a few hundred milliseconds and skip the Redis round-trip entirely for the next few requests from that key, only reconciling with Redis periodically. This trades a small amount of enforcement precision (a client could theoretically slip a few extra requests through right at the cache boundary) for a large reduction in Redis load — an explicit precision-vs-load trade-off worth naming.

3. **Regional Redis clusters, not one global cluster.** Same reasoning as the shortener's multi-region DB: a single global Redis cluster would mean every gateway request pays cross-region round-trip latency on every single check, which is untenable at single-digit-millisecond latency budgets. Instead, run **regional Redis clusters** and accept that a client's true global rate limit is enforced per-region (with a slightly generous per-region allocation, e.g., global_limit / number_of_regions × safety_factor) rather than perfectly globally — another explicit availability/latency-over-perfect-consistency trade-off, consistent with the theme from the shortener doc.

4. **Key expiry keeps memory bounded automatically.** Because every rate-limit key carries a TTL (Section 5), Redis's own eviction handles cleanup — there's no batch job needed to prune stale counters, unlike a naive disk-backed implementation.

5. **Fail-open circuit breaker on Redis unavailability**, per the Section 1 non-functional requirement: gateway middleware wraps the Redis call in a short timeout (a few ms) and, on timeout or error, defaults to `allowed: true` rather than blocking all traffic — protecting overall system availability at the cost of temporarily losing rate-limit enforcement, which is the correct trade for a defense-in-depth component.

---

## 7. Bottlenecks & Likely Follow-Up Questions

- **"What if a single key becomes a hot key in the Redis cluster?"** — same problem class as the shortener's viral-link hot key. A single abusive or misconfigured client hammering one counter can overload the Redis shard owning that key. Mitigation: the local-caching optimization in Section 6.2 naturally dampens this, since a key that's clearly over-limit gets cached as "reject" locally and stops hitting Redis at all.
- **"How do you rate-limit at multiple granularities simultaneously (per-user AND per-IP AND global)?"** — run independent checks against independent keys/algorithms and reject if *any* fails; order them cheapest-first (global counter check before a per-user Lua script) to shed load fast.
- **"Why not use a database instead of Redis?"** — disk-backed writes at 150K+ atomic read-modify-write ops/sec would require a disk-based DB to essentially behave like an in-memory store anyway (heavy caching, WAL tuning); Redis being purpose-built as in-memory with atomic single-threaded-per-shard semantics is a better fit than fighting a general-purpose DB into this shape — direct parallel to why the shortener's DB choice favored a purpose-fit store over forcing a relational DB to do something it's not optimized for.
- **"Consistency model?"** — explicitly approximate/eventually-consistent across regions (Section 6.3), exact within a single region's Redis cluster (Section 5's atomic Lua script guarantees this locally). Same "availability and latency over perfect global consistency" philosophy as the URL shortener design.
