# Capacity Estimation Anchors

Living reference of ballpark numbers for system-design drills. Seeded from the rate-limiter design (Aug 26 2026). Keep appending as new designs surface new anchors.

**How to use these.** Every number here is a *planning anchor*, not a spec. State it as a range with its reason ("order of 100k, a conservative floor"), never as a bare fact. A decision built on a soft number is safe when the **margin is large** — if you're 4× over a ceiling that is itself uncertain by 2×, the conclusion still holds, and saying that out loud is the strong move. A number without its caveat is the exact failure mode this doc exists to prevent.

---

## Redis

| Metric | Value | Notes / caveat |
|---|---|---|
| Single-instance throughput | ~100k ops/sec (conservative floor) | Modern HW + pipelining + O(1) commands hit 500k+. Drops hard with slow network, large payloads, or non-O(1) commands. Use as a deliberately conservative planning floor, not a ceiling. |
| Threading model | Single-threaded command execution | A Lua script runs atomically start-to-finish; concurrent requests for the same key **serialize** at the shard. This is *why* Lua buys cross-node atomicity — the shard is the single serialization point. |
| Memory footprint | Trivial at this scale | 50k keys × 100 B = 5 MB. At this scale sharding is driven by **throughput / availability, not memory**. |

## Storage / keys

| Metric | Value | Notes / caveat |
|---|---|---|
| Rate-limit key size | ~100 bytes | Counter + timestamp + light metadata. ×2–3 if storing tiers / richer metadata. |
| Keyspace (this design) | 5 MB | 50k keys × 100 B. Fits one instance with enormous headroom. |

## Gateway / compute

| Metric | Value | Notes / caveat |
|---|---|---|
| Single gateway node throughput | ~10k req/sec | Rule of thumb. Fleet size = aggregate RPS ÷ per-node (200k ÷ 10k = 20 nodes). Do **not** blur this with the Redis ceiling — they differ by ~10×. |

## Request → ops conversion (the multiplier)

| Metric | Value | Notes / caveat |
|---|---|---|
| Ops per rate-limit check | 1 request ≠ 1 Redis op | Fixed window = INCR + EXPIRE = 2 ops. Token bucket = Lua read-modify-write = multi-op. **Convert RPS → ops before comparing to the Redis ceiling.** 200k RPS × ~2 ≈ 400k ops → over one instance → shard. |
| Script cost assumption | O(1), microseconds | Token-bucket script touches 2 fields + arithmetic, so op-count is the right sizing unit. If the script looped over many keys, per-script CPU time on the single thread would become the bottleneck instead — state this assumption before an interviewer opens it. |

## Latency

| Metric | Value | Notes / caveat |
|---|---|---|
| Intra-DC network hop (to Redis) | ~0.5–1 ms | Paid inline on the request path, per check. Fine for a low-latency API, but know it's there. |
| Fail-open timeout budget | ~5–10 ms | Fast timeout before falling back to a local counter. Failing open on a *hang* still kills your latency SLO — trip a circuit breaker so you stop dialing a dead shard. |

## Algorithm — token bucket

| Metric | Value | Notes / caveat |
|---|---|---|
| Refill rate R | = steady-state limit (100/sec) | Tokens added per second. |
| Capacity C | ≥ R for burst tolerance | C = R → no burst banking. Burst tolerance **requires C > R** (e.g. C=200 lets a quiet client spend a 2× spike). State the burst you're allowing. |
| Stored fields | `tokens`, `last_refill` | Lazy recompute: `tokens = min(C, tokens + (now − last_refill) × R)`. No background refill job over 50k buckets. |

## Distribution / sharding

| Metric | Value | Notes / caveat |
|---|---|---|
| Consistent-hashing reshuffle | ~1/N keys move when adding a shard | vs `hash % N`, which remaps ~all keys and strands ~all counters. This is *why* consistent hashing specifically. |
| Right tool, right layer | Shard the datastore, not the traffic | Consistent hashing on the **Redis tier** (each key's counter → exactly one shard) = correct. Pinning **gateway traffic** to nodes = wrong (breaks statelessness / reintroduces sticky routing). Same technique, opposite verdict — decided by whether pinning helps or hurts. |

## HTTP — over-limit response

| Metric | Value | Notes / caveat |
|---|---|---|
| Status | `429 Too Many Requests` | A bare 429 is a slammed door with no sign — violates "give the client info to back off." |
| `Retry-After` | seconds or HTTP-date | Tells the client when to return; for token bucket, compute it honestly as time-to-next-token. Without it, clients retry blindly and worsen the overload. |
| `X-RateLimit-Limit` / `-Remaining` / `-Reset` | on **every** response | Lets well-behaved clients self-throttle *before* they hit 429. |
