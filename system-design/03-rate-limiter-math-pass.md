# Rate Limiter — NALSD Math Pass

*Applies the numbers to [02-rate-limiter-nalsd.md](./02-rate-limiter-nalsd.md). This doc is the "show your arithmetic, find where it breaks, redesign" pass — Workbook Ch. 12 format. Every number here is derived, not assumed; where an earlier doc's number turns out to be wrong once you actually compute it, that's called out explicitly rather than smoothed over.*

---

## 0. Fixed inputs (carried from doc 02)

- Peak QPS to rate-limit: **150,000 req/sec**
- Registered identities: 500M users
- Window: 60s, with current+previous window both live → effective TTL ≈ 120s
- Redis node reference class: 4 vCPU, single-threaded-per-shard command execution
- Gateway node reference class: 8 vCPU

---

## 1. First-pass machine count — naive single-machine check

**Question: could one Redis instance handle this at all?**

Benchmarked single-node Redis throughput for simple atomic commands (INCR, or a short Lua EVAL like the rate-limit script) on modern hardware: **~80,000–100,000 ops/sec**, single-threaded execution per shard. Use the conservative end: **80,000 ops/sec/node**.

```
150,000 req/sec (target) ÷ 80,000 ops/sec (1 node) = 1.875
```

**Breaks immediately.** One node can't do it — not close. Minimum viable shard count on throughput alone: **⌈1.875⌉ = 2 shards.**

This is the expected outcome (doc 02 already assumed a cluster), but it's worth computing explicitly rather than asserting it, because the *margin* by which it breaks matters for what comes next.

---

## 2. Storage math — and where doc 02's informal estimate was wrong

Doc 02 estimated memory loosely as "a few billion active (key, window) counters... hundreds of GB." Let's actually compute it, because "active" is doing a lot of unexamined work in that sentence.

**Naive (wrong) version — treats every registered user as holding a live key:**
```
500M users × 5 endpoints × 2 window-keys × 100 bytes/key
= 500,000,000 × 5 × 2 × 100
= 500,000,000,000 bytes = 500 GB
```
At 500GB, dividing by a realistic node's usable cache RAM (~24GB after overhead):
```
500 GB ÷ 24 GB/node ≈ 21 nodes
```
This makes it look like **memory**, not throughput, is the binding constraint — 21 nodes for memory vs. 2 nodes for throughput. If you stopped here, you'd oversize the cluster by ~10x.

**Correct version — TTL means only *currently active* users hold live keys, not all registered users.**

At 150,000 req/sec, and assuming (conservatively, worst-case) that every request in a 120-second TTL window could be from a distinct user:
```
150,000 req/sec × 120 sec (TTL window) = 18,000,000 unique active users touched in the live window
```
Simplify to one limiter key per user (drop the 5-endpoints multiplier for this pass — it's a linear scaler you can reapply if a specific service needs it):
```
18,000,000 users × 2 window-keys × 100 bytes/key
= 3,600,000,000 bytes ≈ 3.6 GB
```

**This is the actual number: ~3.6 GB, not 500 GB.** It fits on a single node's RAM with room to spare. Memory is not a scaling constraint for this system at this traffic level — it's throughput (Section 1) that drives shard count, not storage.

**Takeaway to carry forward:** the informal "hundreds of GB" figure in doc 02 was an artifact of conflating *registered* users with *concurrently active* users. TTL-bounded state means the working set tracks live traffic, not total user base — this is a generally reusable insight for any TTL'd counter/cache system, worth remembering for future NALSD math passes.

---

## 3. IOPS / throughput math — gateway layer

Redis isn't the only machine count to size. The gateway/middleware layer making the calls needs its own math.

Assume a gateway node can sustain **~10,000 req/sec** (accounts for the HTTP handling plus the ~1ms round trip to Redis plus app-level overhead — a conservative, commonly-cited figure for a service-mesh-fronted app node doing a synchronous downstream call, not a raw reverse-proxy number).

```
150,000 req/sec ÷ 10,000 req/sec/node = 15 nodes (minimum, no headroom)
```

Standard SRE practice: **provision for N+1 at minimum, typically 2x for a system that must survive a full zone loss** without shedding load. Apply 2x:
```
15 × 2 = 30 gateway nodes
```

**Connection count sanity check:** 30 gateway nodes × a modest connection pool (say 50 persistent connections each to Redis) = 1,500 total connections into the Redis cluster. Default Redis max-clients is 10,000 — no issue here, but this is exactly the kind of check worth running explicitly rather than assuming, since connection exhaustion is a real, commonly-hit ceiling in production Redis deployments once a fleet grows large.

---

## 4. First real bottleneck — redundancy headroom, not raw throughput

Section 1 found the throughput-minimum shard count is **2**. Check what 2 shards actually gives you in practice:

```
2 shards × 80,000 ops/sec = 160,000 ops/sec aggregate capacity
Headroom over 150,000 target = 160,000 - 150,000 = 10,000 (6.7% headroom)
```

**This breaks under failure, not under load.** If you lose one shard (node failure, deploy, AZ issue — routine events at Google scale, not edge cases):
```
Remaining capacity: 1 shard × 80,000 = 80,000 ops/sec
Target: 150,000 ops/sec
Deficit: 150,000 - 80,000 = 70,000 ops/sec (47% of required capacity gone)
```
A single node loss collapses the cluster to less than half of what's needed — the system falls over the moment routine infrastructure churn happens, well before any traffic anomaly. **This is the first genuine numerical bottleneck**, and it's a redundancy-math problem, not an algorithm or storage problem.

### Iterate: add a third shard

```
3 shards × 80,000 ops/sec = 240,000 ops/sec aggregate
Losing any 1 shard: 2 × 80,000 = 160,000 ops/sec remaining
160,000 > 150,000 target → survives a single-shard failure with 10,000 ops/sec (6.7%) to spare
```
**Resolved for single-failure tolerance**, though the post-failure margin is thin — worth flagging as a still-tight number rather than declaring victory. A 4th shard would give a more comfortable post-failure margin (3 × 80,000 = 240,000 vs. target, 60% headroom even after a loss) — a legitimate cost/safety-margin trade-off to raise explicitly rather than silently pick one.

**Revised minimum: 3–4 Redis primary shards** (plus 1 replica each for HA reads/failover, so 6–8 physical Redis nodes total) — this is the number that actually survives the failure mode the naive Section-1 calculation didn't account for.

---

## 5. Second bottleneck, uncovered by iterating further — the hot key

Sharding by `hash(key)` distributes *different* keys across shards, but **a single key always lives on exactly one shard** — consistent hashing doesn't split one key's traffic across multiple nodes.

**Model an abusive/viral client hammering a single rate-limit key at 50,000 req/sec** (a very plausible number — a single misconfigured internal service, a scraper, or a genuine DDoS against one account):

```
That key's shard must absorb: 50,000 req/sec (all from this one key)
Plus its fair share of the other traffic: (150,000 - 50,000) ÷ 3 shards ≈ 33,300 req/sec
Total load on that one shard: 50,000 + 33,300 = 83,300 req/sec
Shard capacity: 80,000 req/sec
```
**Breaks again — by 3,300 req/sec (4%) on that one shard**, even though the *cluster-wide* aggregate (240,000 capacity vs. 150,000 + the hot key's excess) has plenty of room. This is the classic hot-key/noisy-neighbor failure: **aggregate cluster headroom is irrelevant when one key's traffic exceeds what a single shard can carry**, because that key cannot be spread across shards by the sharding scheme alone.

### Iterate: split the hot key

Standard fix — instead of one counter key per rate-limited identity, use **N sub-keys** for high-traffic keys (`ratelimit:{key}:{shard_index}` where `shard_index = hash(request_id) % N`), summing across sub-keys (or using a cheaper probabilistic approximation) to enforce the aggregate limit, while distributing the *storage and command load* across multiple physical shards.

With N=10 sub-keys spread across the 3 physical shards:
```
50,000 req/sec ÷ 10 sub-keys ≈ 5,000 req/sec per sub-key
Distributed roughly evenly across 3 shards ≈ 1,667 req/sec of hot-key load per shard
```
Added to each shard's existing fair share (~33,300 req/sec): **~35,000 req/sec per shard**, well under the 80,000 ceiling. **Resolved** — and notice the fix didn't require adding a single machine; it required changing the *keying scheme* for a specific class of key. This is a useful general lesson: not every bottleneck is solved by scaling out — some are solved by changing what's being sharded.

---

## 6. Final sized design (after two iterations)

| Layer | Naive first pass | After iteration | Driving constraint |
|---|---|---|---|
| Redis shards (primary) | 2 (throughput minimum) | **3–4** | Single-shard-failure survival, not raw throughput |
| Redis nodes (incl. replicas) | 2 | **6–8** | Add 1 replica/shard for HA |
| Redis memory/node | N/A | **~3.6 GB total cluster-wide** — not a constraint at all | TTL bounds working set to active users, not registered users |
| Gateway nodes | 15 (throughput minimum) | **30** | Standard 2x zone-failure headroom |
| Hot-key handling | none (breaks at 50K req/sec on one key) | **key-splitting, N=10 sub-keys** | Per-shard ceiling is independent of cluster-aggregate headroom |

**What this pass changed from the doc 02 design:** doc 02 correctly identified *that* sharding and hot-key handling were needed, in qualitative terms. This pass is what turns that into defensible numbers — and it surfaced that the actual memory estimate was 100x smaller than the informal figure, while the actual redundancy and hot-key requirements were understated. That asymmetry (storage: over-scoped, redundancy/hot-key: under-scoped) is the kind of thing a pure qualitative design pass misses and a math pass catches.

---

## 7. Follow-up: where does this break next as the product grows?

Quick growth check, worth having ready if pushed further in an interview: if active concurrent users (not registered users) grow 3x (product goes from 18M to 54M concurrently-active), memory scales linearly and trivially (3.6GB → ~10.8GB, still a non-issue), but **throughput scales with request volume, not user count directly** — if QPS also roughly triples to ~450,000 req/sec, redo Section 4's math: `450,000 ÷ 80,000 ≈ 5.6 → 6 shards minimum`, and redo the failure-tolerance check the same way before trusting that number. The point isn't the specific future number — it's that **the math pass is a repeatable procedure**, not a one-time answer: same four steps (naive count → storage check → failure-tolerance check → hot-key check) every time the target QPS changes materially.
