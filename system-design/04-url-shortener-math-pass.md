# URL Shortener — NALSD Math Pass

*Applies the numbers to [01-url-shortener-nalsd.md](./01-url-shortener-nalsd.md), same procedure as [03-rate-limiter-math-pass.md](./03-rate-limiter-math-pass.md). Every number here is derived, not assumed. Where the shortener's math turns out to break for a *different* reason than the rate limiter's did, that contrast is called out explicitly — it's the more useful takeaway than either number alone.*

---

## 0. Fixed inputs (carried from doc 01)

- Peak reads (redirects): **150,000 req/sec**
- Peak writes (shortens): **1,500 req/sec**
- Total logical data: 50 TB raw → **150 TB with 3x replication**
- Short code space: Base62, 7 chars (~3.5 trillion, target 100B URLs)
- DB node reference class: wide-column store node, 4 TB usable SSD, sustains ~15,000 point-read ops/sec (conservative — combines compute + disk + network overhead for a single node under real conditions)
- Cache node reference class: Redis/Memcached, ~40 GB usable RAM after overhead, ~100,000 ops/sec sustained

---

## 1. First-pass machine count — naive single-machine check

**Could one DB node handle the redirect read path alone?**
```
150,000 req/sec (peak reads) ÷ 15,000 ops/sec (1 DB node) = 10
```
Breaks by 10x if reads hit the DB directly — this is exactly why doc 01's architecture puts a cache in front of the DB rather than serving redirects straight from storage. Worth computing explicitly rather than just asserting "add a cache," because the *margin* (10x, not 2x) tells you the cache isn't optional headroom — it's structurally load-bearing for this design to work at all.

---

## 2. Storage math — and the first contrast with the rate limiter

**Node count driven by storage:**
```
150 TB (total, with replication) ÷ 4 TB/node = 37.5 → 38 nodes
```

**Node count driven by throughput**, assuming the cache absorbs 90%+ of reads (doc 01's target) so the DB only sees cache misses plus all writes:
```
Cache misses:  150,000 × (1 − 0.90) = 15,000 reads/sec
Writes:        1,500/sec (writes go straight to DB, bypassing cache)
Total DB ops:  15,000 + 1,500 = 16,500 ops/sec

16,500 ÷ 15,000 ops/sec/node = 1.1 → 2 nodes
```

**Storage requires 38 nodes; throughput requires 2.** The DB cluster size is **storage-bound by roughly 19x** — the opposite of the rate limiter's DB layer, which was throughput-bound (2–3 shards needed for QPS, only 3.6 GB of actual data). The reason is structural, not incidental: **rate-limit counters expire (TTL-bounded working set); shortened URLs are permanent records that accumulate forever.** Same math-pass procedure, opposite binding constraint — worth stating explicitly, because "shard count is storage-bound" vs "throughput-bound" changes what you optimize first (disk/replication tuning vs. CPU/connection tuning) and it's easy to default to whichever one you solved last time.

---

## 3. Cache layer math

**Memory-driven sizing.** Real-world redirect traffic has strong temporal locality — most clicks land on recently-created or currently-trending links, not a uniform sample of all 100B URLs ever created. Estimate the working set from actual daily traffic rather than the full corpus:
```
Average reads/day: 31,700/sec (doc 01 average, not peak) × 86,400 sec/day ≈ 2.74B read events/day
Assume high repeat-locality → est. ~100M distinct short_codes actually touched per day
(this is the parameter to validate against real traffic once live; treat as a first estimate)

100,000,000 codes × 500 bytes/record = 50 GB working set
```
```
50 GB ÷ 40 GB usable/node ≈ 1.25 → 2 nodes (memory-driven)
```

**Throughput-driven sizing:**
```
Cache must serve ~90% of 150,000 peak = 135,000 reads/sec
135,000 ÷ 100,000 ops/sec/node = 1.35 → 2 nodes (throughput-driven)
```

Both dimensions agree: **2 nodes minimum.** Apply the standard 2x zone-failure headroom (same convention as the rate limiter pass): **4 cache nodes baseline.** Unlike the DB layer, cache sizing here is *not* the layer with a large margin between constraints — which is a signal to check the next thing carefully, since there's little slack to absorb a surprise.

---

## 4. First real bottleneck — a single viral link on a single cache node

Hashing spreads *different* short_codes across cache nodes, but **one short_code always lives on exactly one node** — same structural issue as the rate limiter's hot key, applied here to a single viral link instead of a single abusive client.

**Solve for the breaking point** — what share of total system traffic would one link need to capture to overwhelm a single 100,000 ops/sec cache node, accounting for that node also carrying its normal 1/4 fair share of everything else (4-node cluster from Section 3)?

```
Let x = that link's share of total 150,000 req/sec traffic
Load on the hot node = x·150,000 + (150,000 − x·150,000) ÷ 4  ≤  100,000

Solving:
x·150,000 + 37,500 − x·37,500 = 100,000
x·112,500 = 62,500
x ≈ 0.556  (≈56%)
```

**At roughly 56%+ of total system traffic concentrated on one link, that link's cache node breaks — with the rest of the cluster sitting comfortably under capacity the entire time.** This is the same shape of result as the rate limiter's hot-key finding: aggregate cluster headroom is irrelevant once a single key's traffic exceeds what its one assigned node can carry. 56% sounds extreme, but it's exactly the profile of a real event a global-scale shortener sees — a single URL in a major breaking-news alert, an outage status page, or a post from an account with an enormous, highly-synchronized follower base.

---

## 5. Iterate — this is exactly what the CDN layer in doc 01 is for

Doc 01 (Section 6.1) already put a CDN edge cache with a short TTL in front of the origin, specifically to absorb hot/viral traffic. This pass is what shows numerically *why* that architectural choice, and not just "add more cache nodes," is the right fix.

**Without CDN**, all 56%+ of raw client traffic for the hot link reaches the origin cache tier directly (Section 4's breaking scenario).

**With CDN edge caching** (TTL = 60s, ~200 globally-distributed edge PoPs, per doc 01 Section 5's hybrid 302+CDN design): each edge node serves its own cached copy to all clients hitting that edge, and only refreshes from the origin once per TTL window:
```
Origin-facing requests for that link ≈ (number of edge PoPs) ÷ (TTL seconds)
                                     = 200 ÷ 60
                                     ≈ 3.3 req/sec
```
**Origin load for even a maximally-viral link drops from ~84,000 req/sec (56% of 150,000) to ~3.3 req/sec** — regardless of how many millions of real clients are hitting the link, because the edge layer absorbs the raw volume and the origin only ever sees refresh traffic. The hot-key problem is fully resolved at the origin cache/DB tier without adding a single origin node — the fix is architectural (push the absorption point to the edge), not "buy more capacity," mirroring the rate limiter pass's lesson that not every bottleneck is solved by scaling out.

(The CDN's own edge capacity has to absorb the raw client volume, but that's explicitly outsourced to the CDN provider's already-massive, globally-distributed footprint — a legitimate and standard architectural move, worth naming as a deliberate choice rather than an assumption glossed over.)

---

## 6. Redundancy check — does the DB layer need deliberate fixing, like the rate limiter's did?

The rate limiter's naive 2-shard DB layer had almost no failure headroom (Section 4 of the math pass doc) and needed a deliberate 3rd shard added. Check whether the URL shortener's DB layer has the same problem:

```
Nodes needed for throughput: 2 (Section 2)
Nodes actually provisioned (storage-driven): 38
Losing any 1 of 38 nodes: 37 nodes remain, vs. 2 needed for throughput
```
**No iteration needed here.** Because the DB cluster is storage-bound by ~19x, it's already so overprovisioned relative to its throughput requirement that single-node (or even many-node) failures are a rounding error against capacity. This is worth stating explicitly rather than skipping past: **it's a case where solving for one constraint (storage) incidentally over-solves another (redundancy) for free** — the opposite of the rate limiter, where solving for throughput left redundancy dangerously thin and had to be fixed separately. Don't assume this pattern generalizes; it's a consequence of this specific 19x gap, not a rule.

---

## 7. Final sized design (after iteration)

| Layer | Naive first pass | After iteration | Driving constraint |
|---|---|---|---|
| DB nodes | 2 (throughput minimum) | **38** | Storage (150 TB ÷ 4 TB/node), not throughput |
| DB redundancy | — | **Already covered**, no separate fix needed | 19x storage-driven overprovisioning absorbs node loss |
| Cache nodes | 2 (throughput/memory minimum) | **4** | Standard 2x zone-failure headroom |
| Hot-link handling | none (breaks at ~56% traffic share on one link) | **CDN edge cache, 60s TTL, ~200 PoPs** | Per-node ceiling is independent of cluster-aggregate headroom — architectural fix, not more nodes |

**What this pass changed from the doc 01 design:** doc 01 qualitatively specified a cache-first architecture, a storage-bound sharded DB, and a CDN layer for hot links. This pass turns "cache-first" into an actual node count (4, not an unspecified "some"), confirms the DB layer really is storage-bound rather than assuming it, and computes the exact traffic-share threshold (~56%) where the hot-link problem doc 01 mentioned qualitatively actually starts to bite — and shows the CDN's TTL/PoP math is what makes that threshold irrelevant in practice, rather than just asserting "CDN handles it."

---

## 8. Follow-up: same procedure, different growth axis

Unlike the rate limiter (where growth in *active users* was the relevant axis), the shortener has **two independent growth axes** worth checking separately, because they drive different layers:

- **Corpus growth** (total URLs ever created, e.g. 100B → 200B): scales **storage node count linearly** (38 → ~76), has **no effect on cache or DB throughput node counts**, since those are driven by request rate, not corpus size.
- **Traffic growth** (peak QPS, e.g. 150,000 → 300,000): scales **cache and DB-throughput node counts**, has **no effect on storage node count**.

Re-running Sections 1–4 independently for whichever axis actually changed — rather than re-deriving the whole cluster from scratch — is the reusable procedure here, same as the rate limiter pass's closing point: identify which fixed input moved, and only recompute the sections whose formula depends on that input.
