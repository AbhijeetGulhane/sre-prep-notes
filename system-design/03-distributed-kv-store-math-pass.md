# Distributed KV Store — NALSD Math Pass

*Applies the numbers to [03-distributed-kv-store-nalsd.md](./03-distributed-kv-store-nalsd.md), same procedure as [01-url-shortener-math-pass.md](./01-url-shortener-math-pass.md) and [02-rate-limiter-math-pass.md](./02-rate-limiter-math-pass.md). This one breaks in a genuinely different way than either prior pass — not on raw throughput or on a hot key, but on a probability calculation that only shows up once you actually multiply it out across a billion keys.*

---

## 0. Fixed inputs (carried from doc 03)

- Peak ops/sec: **500,000 total** — 400,000 reads/sec, 100,000 writes/sec
- Data: 1B keys × 1KB avg = 1 TB raw → **3 TB with N=3 replication**
- Replication/quorum: **N=3, W=2, R=2**
- Node reference class: **20,000 ops/sec/node** sustained (point reads/writes on SSD-backed storage, includes network overhead), **1 TB usable storage/node**

---

## 1. First-pass machine count — the naive version, and why it's wrong before we even finish it

**Naive approach: node count = client-facing QPS ÷ per-node capacity.**
```
500,000 req/sec ÷ 20,000 ops/sec/node = 25 nodes
```
This looks like a complete first pass, same shape as doc 01 and doc 02's Section 1. **It's wrong, and not by a rounding error** — it silently assumes one client request equals one backend operation. Doc 03's own read/write flow (Section 5) says otherwise: a write goes to **all N=3 replicas**, and a read is served by **R=2 replicas**. The naive number never accounts for replication fan-out at all. This has to be caught before any of the rest of this pass means anything.

---

## 2. The correct throughput math — replication fan-out

**Writes:** doc 03 Section 5 — the coordinator sends every write to all N replicas in parallel (even though it only *waits* for W before returning to the client, all N still execute):
```
100,000 client writes/sec × N=3 replicas = 300,000 backend write-ops/sec
```

**Reads:** the coordinator queries R replicas per client read (the common-path behavior; occasional full-N reads for read-repair sampling are a smaller addition, ignored here for the primary estimate):
```
400,000 client reads/sec × R=2 = 800,000 backend read-ops/sec
```

**Total backend ops/sec:**
```
300,000 + 800,000 = 1,100,000 backend ops/sec
```

**Compare to the naive Section 1 number:** 1,100,000 vs. 500,000 — the real backend load is **2.2x the client-facing QPS**, entirely from replication fan-out. Recompute node count:
```
1,100,000 ÷ 20,000 ops/sec/node = 55 nodes
```

**55 nodes, not 25.** The naive first pass understated the cluster by more than half. This is the load-bearing correction of this whole math pass: for any replicated system, **"machine count = client QPS ÷ per-node capacity" is the wrong formula** — it has to be "machine count = backend ops/sec (after fan-out) ÷ per-node capacity." Worth carrying forward as a general rule for any future replicated-system math pass, not just this one.

---

## 3. Storage math

```
3 TB (total, replicated) ÷ 1 TB/node = 3 nodes
```

**Storage needs 3 nodes; throughput needs 55.** This cluster is **throughput-bound by a wide margin** (~18x) — same shape as the rate limiter's math pass, and the opposite of the URL shortener's storage-bound DB layer. Consistent with the general pattern established across all three systems so far: **TTL'd or per-operation-heavy workloads (rate limiter, and now this store's replicated ops) tend to be throughput-bound; permanent-record accumulation (the shortener) tends to be storage-bound.** Worth having that generalization ready — it's the kind of pattern-recognition an interviewer is actually listening for when they ask "why is the cluster this size."

---

## 4. First real numerical bottleneck — quorum loss probability at 1B keys

Section 2 sized the *cluster* correctly. But cluster-aggregate health isn't what determines whether any individual **key** stays available — a specific key only lives on its own preference list of **N=3 specific nodes** (doc 03, Section 3), regardless of how many total nodes the cluster has. This is the same structural pattern as the rate limiter's hot-key and the shortener's hot-link findings — **aggregate cluster headroom doesn't protect a resource pinned to a small, fixed subset of nodes** — but here it shows up as a probability calculation, not a load-concentration one.

**Setup:** assume a modest, realistic per-node daily failure probability, independent across nodes for this first pass (correlated failures are addressed in Section 6):
```
p = 0.001  (0.1% chance a given node is down on a given day — a reasonable single-node figure for a well-run fleet)
```

**A key stays available for both reads and writes only if at least 2 of its 3 replicas are up** (since W=2 and R=2 both require 2 acks/responses out of 3). The key loses quorum if **2 or more of its 3 replicas are simultaneously down**:
```
P(exactly 2 of 3 down) = C(3,2) × p² × (1−p) = 3 × (0.001)² × 0.999 ≈ 3 × 10⁻⁶
P(all 3 down) = p³ = 10⁻⁹  (negligible next to the above)

P(a given key loses quorum on a given day) ≈ 3 × 10⁻⁶
```

**That looks tiny.** Now multiply by the actual key count from Section 0:
```
1,000,000,000 keys × 3 × 10⁻⁶ = 3,000 keys/day
```

**~3,000 keys per day go quorum-unavailable somewhere in the cluster, on a system where only ~0.1% of individual nodes are ever down at once.** This is the real first bottleneck, and it's a genuinely surprising one if you only checked cluster-aggregate throughput and storage (Sections 2–3 both looked comfortable) — a tiny per-key probability, multiplied across a billion keys, produces a concrete and operationally significant daily failure count. This is structurally the same "birthday paradox" shape as the collision-retry problem the shortener doc rejected MD5 truncation for (Section 4 of doc 01) — small per-item probabilities stop being negligible once the item count is large enough, and 1B is large enough.

---

## 5. Iterate — raise N, recheck the probability, recheck the cost it adds

**Option: N=5, with a majority quorum (W=3, R=3, satisfying W+R=6>5).** A key now loses quorum only if **3 or more of its 5 replicas** are down simultaneously:
```
P(exactly 3 of 5 down) = C(5,3) × p³ × (1−p)² = 10 × (0.001)³ × (0.999)² ≈ 10 × 10⁻⁹ = 10⁻⁸
(higher-failure terms, 4-of-5 and 5-of-5, are smaller still and negligible here)

1,000,000,000 keys × 10⁻⁸ = 10 keys/day
```

**3,000/day → 10/day — a ~300x improvement**, from raising N alone. This is a large, clearly worthwhile reliability gain — but recompute what it costs before declaring victory, same discipline as every prior iteration in this series.

**Cost of N=5 — redo Sections 2 and 3:**
```
Storage: 1 TB raw × 5 replicas = 5 TB ÷ 1 TB/node = 5 nodes (was 3)

Writes:  100,000 × N=5 = 500,000 backend write-ops/sec (was 300,000)
Reads:   400,000 × R=3 = 1,200,000 backend read-ops/sec (was 800,000, since R went 2→3)
Total:   500,000 + 1,200,000 = 1,700,000 backend ops/sec (was 1,100,000)

Nodes needed: 1,700,000 ÷ 20,000 ops/sec/node = 85 nodes (was 55)
```

**N=5 costs 30 additional nodes (55 → 85, a 55% increase) to buy a 300x reduction in daily quorum-loss incidents.** Whether that trade is worth it depends entirely on what's actually stored in this key space — for session data or cache-tier keys, 3,000 stale/unavailable keys a day across a billion might be entirely tolerable; for anything where losing quorum on a key has real business consequence, the 30-node cost is cheap insurance. **This is exactly the kind of trade-off worth stating as a trade-off in an interview, not resolving unilaterally** — the correct N depends on the workload doc 03 never fully pinned down (it deliberately described this as a shared, general-purpose store).

---

## 6. A caveat the independence assumption glosses over: correlated failures

Section 4's math assumed node failures are **independent** — but the actual dominant real-world risk is **correlated** failure: an entire rack losing power, or a full availability zone going down, takes out many nodes simultaneously, not one at a time by coincidence. If a key's 3 (or 5) replicas happen to sit in the same rack or AZ, a single correlated event — not a rare multi-node coincidence — can take out 2-of-3 or 3-of-5 at once, making the real-world quorum-loss rate far higher than Section 4's independent-failure estimate.

**The fix isn't a different N — it's failure-domain-aware placement**: when assigning a key's N replicas via the consistent-hashing ring (doc 03, Section 3), explicitly constrain the preference list to span **at least N distinct racks/AZs**, not just N distinct physical nodes. This doesn't change any of the arithmetic above — it changes whether the independence assumption the arithmetic relies on is actually true in production. Worth stating explicitly as a gap between the clean probability model and what it takes to make that model's assumptions hold.

---

## 7. Final sized design (after iteration)

| | Naive first pass | Corrected (N=3) | If raised to N=5 |
|---|---|---|---|
| Backend ops/sec | 500,000 (client QPS only — wrong) | **1,100,000** (fan-out corrected) | 1,700,000 |
| Nodes (throughput) | 25 | **55** | 85 |
| Nodes (storage) | — | 3 | 5 |
| Binding constraint | — | **Throughput, ~18x over storage** | Throughput, still dominant |
| Quorum-loss rate | not modeled | **~3,000 keys/day** | ~10 keys/day |
| Failure-domain placement | not modeled | **Required** — independence assumption above doesn't hold without it | Required |

**What this pass changed from the doc 03 design:** doc 03 qualitatively specified N=3/W=2/R=2 as "the standard default" and mentioned hot keys and multi-datacenter placement as scaling considerations. This pass is what shows the client-QPS-to-machine-count formula was wrong by more than 2x once fan-out is included, and — the more interesting finding — that the real first bottleneck isn't throughput or storage at all, but a probabilistic quorum-loss rate that only becomes visible once you multiply a small per-key probability across a billion keys.

---

## 8. Follow-up: same procedure, different axis

- **Key count grows** (1B → 10B): storage scales linearly (3TB → 30TB, still not the binding constraint at these node counts), and — more importantly — **Section 4's quorum-loss count scales linearly too** (3,000/day → 30,000/day at N=3), since it's `keys × fixed per-key probability`. This is worth checking explicitly rather than assuming only throughput needs revisiting when key count grows.
- **Ops/sec grows** (500K → 1M): redo Section 2's fan-out math directly — node count driven by throughput scales with it, storage and quorum-loss rate are unaffected (they depend on key count and per-node reliability, not request rate).
- **Per-node reliability changes** (better or worse hardware, `p` changes): Section 4's probability is quadratic (N=3) or cubic (N=5) in `p`, so this is the most sensitive input in the whole pass — halving `p` from 0.001 to 0.0005 drops the N=3 quorum-loss rate by 4x (750/day), a bigger lever than most of the architectural choices in this document. Worth checking this input first if the real-world number ever needs to move.
