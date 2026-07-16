# Distributed KV Store — Conflict Resolution: Vector Clocks vs Last-Write-Wins

*Companion to [03-distributed-kv-store-nalsd.md](./03-distributed-kv-store-nalsd.md) — Section 4 of that doc introduced this as one line ("vector clocks vs LWW"); this doc is the full mechanics behind it, worked through with actual numbers.*

---

## 1. Why this problem exists at all

This only comes up because of a choice already made in doc 03: **W < N** (Section 4's W=2, R=2, N=3 default) means a write can succeed without every replica agreeing on it, and sloppy quorum means a write can even land on a node outside the key's normal preference list temporarily. Two clients can therefore both write to the same key, each believing they're the only writer, with neither replica set ever seeing the other's write before both report success. When a later read hits replicas holding both versions, **something has to decide what "the" value is.** That's the entire problem this doc is about.

This is the direct cost of the Section 1 choice (availability over consistency) in doc 03 — a system that blocked writes during any disagreement wouldn't need this machinery at all, and would trade it for lower availability instead.

---

## 2. Last-Write-Wins (LWW)

### Mechanics
Every write is tagged with a timestamp. When two versions of a key are found to conflict, keep the one with the higher timestamp and silently discard the other.

### Why it's attractive
- **O(1) metadata** — a single timestamp, doesn't grow no matter how many clients write to the key over its lifetime.
- Trivial to implement, trivial to compare two versions, no sibling-handling burden on the client.

### Why it loses data — worked example
Two clients, unaware of each other, both read the same shopping cart value at the same starting point:

```
t=100: Client A reads cart = {}  (empty)
t=101: Client A adds item X  -> writes cart = {X},  timestamp 101
t=102: Client B reads cart = {}  (still hasn't seen A's write yet — read went to a different replica)
t=103: Client B adds item Y  -> writes cart = {Y},  timestamp 103
```

Under LWW, timestamp 103 > timestamp 101, so **B's write wins outright and A's write is discarded** — not merged, not flagged, just gone. The final cart is `{Y}`. Item X, a completely legitimate concurrent addition, silently vanishes with no error, no log entry from the client's perspective, and no way to know it ever happened. This is the failure mode that matters: **it's not that LWW is wrong sometimes — it's that it's wrong silently**, which is worse than an explicit conflict a client has to handle.

### The clock-skew problem, on top of the semantic problem
Even setting aside the "concurrent writes should merge" issue, LWW requires trustworthy timestamps. Clock synchronization across a distributed cluster (NTP-based) still has real skew — commonly single-digit to tens of milliseconds even when well-configured, more under NTP server issues or virtualized hosts. For writes that land within that skew window, **"last" isn't even reliably well-defined** — the timestamp ordering can disagree with the true wall-clock ordering, so LWW can pick the *earlier* write as the winner and not even know it. This is a second, independent way LWW loses data beyond the semantic merge problem above.

---

## 3. Vector Clocks

### Mechanics
Instead of one timestamp, each value carries a **map of `{coordinator_id: counter}`** — one counter per node that has ever coordinated a write to this key. A client's write includes the vector clock (`context`, doc 03 Section 2's API) it last read; the coordinator handling the write increments its own entry in that map before storing the new version.

**Comparing two vector clocks, VC1 and VC2:**
- **VC1 dominates VC2** (VC1 happened-after VC2, causally) if VC1's counter is ≥ VC2's counter for *every* coordinator entry, and strictly greater for at least one. If VC1 dominates, VC2 is safely superseded and discarded — no data loss, because VC1's writer had already seen VC2's value when it wrote.
- **Neither dominates** (some entries higher in VC1, others higher in VC2) → the writes are **concurrent** — neither writer knew about the other's write. Both versions are kept and returned to the client as siblings (doc 03 Section 2's API shape).

### Worked example — same shopping cart scenario, with vector clocks

```
Client reads key, gets v1 with clock {S1: 1}          (S1 = coordinator node that handled the original write)

Client A reads v1, writes an update (adds item X)
  Coordinator for A's write is S2 → increments its own entry
  v2 = {S1: 1, S2: 1}

Client B independently reads v1 (before A's write has propagated to B's replica)
  Client B writes an update (adds item Y)
  Coordinator for B's write is S3 → increments its own entry
  v3 = {S1: 1, S3: 1}
```

**Compare v2 = {S1:1, S2:1} against v3 = {S1:1, S3:1}:**
- S1: equal (1 = 1)
- S2: v2 has 1, v3 has 0 → v2 ahead here
- S3: v2 has 0, v3 has 1 → v3 ahead here

**Neither dominates** — v2 is ahead on S2, v3 is ahead on S3. This is exactly the concurrent-write case. **Both v2 and v3 are kept and returned to the client as siblings** on the next `GET` — nothing is silently discarded, in contrast to LWW's outright loss of A's write above.

**Resolving the siblings:** the client (which has the domain knowledge the store doesn't) merges the two cart contents into `{X, Y}` and writes back with a clock that dominates both inputs:
```
v4 = {S1: 1, S2: 1, S3: 1}   (dominates both v2 and v3 — every entry ≥, with at least one strictly greater... 
                               here it's ≥ on all three vs v2 and vs v3 individually, so it causally supersedes both)
```
Once this write is acknowledged, v2 and v3 are both superseded and eligible for cleanup — the conflict is resolved *correctly*, with no data loss, because the merge decision (union the item sets) was made by the layer that actually understands what a shopping cart is.

### Why Dynamo specifically chose this over LWW
The original Dynamo paper's canonical example is exactly the shopping cart above — Amazon's actual production reasoning was that **losing a customer's added item is a worse failure mode than asking the client to occasionally merge two versions.** Vector clocks trade a small amount of client-side complexity (handling siblings) for a hard correctness guarantee (no concurrent write is ever silently discarded) — and critically, they push the *merge semantics* to the application layer, where domain knowledge lives, rather than having the storage layer guess via a timestamp it has no context to interpret correctly.

---

## 4. The cost vector clocks don't advertise: unbounded growth

Vector clocks aren't free, and it's worth being explicit about the trade-off rather than presenting them as a strictly-better option.

**The problem:** every distinct coordinator node that has ever handled a write to a given key adds a permanent entry to that key's vector clock. A key that's popular enough to be coordinated by many different nodes over its lifetime (which, at Google scale, is exactly the kind of key this store is built for) can accumulate a clock with **hundreds or thousands of entries** — each `GET`/`PUT` now has to serialize, transmit, and compare that entire growing map, which is real, unbounded overhead per operation on exactly the hottest keys.

**The fix used in production (Dynamo's actual approach):** cap the vector clock at a fixed size (Dynamo's paper uses ~10 entries) and **drop the oldest entry by timestamp** once the cap is exceeded. This trades a small amount of causal precision for a bounded metadata size: in the rare case where the dropped entry was still relevant (an old write that genuinely hadn't been superseded yet gets its clock entry pruned), the system may treat it as concurrent with a much newer write when it technically wasn't — but this is judged an acceptable rare-case cost against the alternative of unbounded per-operation overhead on hot keys.

**A secondary mitigation:** keying clock entries by **coordinator node**, not by individual client, keeps the practical number of distinct entries much smaller than it would be in a system with millions of individual client-side actors — there are far fewer coordinator nodes in a cluster than there are clients calling into it.

---

## 5. Decision matrix

| | Last-write-wins | Vector clocks |
|---|---|---|
| Metadata size | O(1), fixed | Grows with distinct writers, needs capping in production |
| Data loss on concurrent writes | **Yes, silently** | No — concurrent writes preserved as siblings |
| Depends on synchronized clocks | Yes — vulnerable to clock skew | No — purely causal, no wall-clock dependency |
| Client complexity | None — store always returns one value | Client must handle and merge siblings |
| Correct "winner" is well-defined | Only when writes aren't truly concurrent | Always — concurrency is detected, not guessed at |
| Best fit | Data where losing a stale concurrent write is harmless (e.g., "last seen" timestamp, ephemeral view state) | Data where merging concurrent updates is meaningful and losing one silently is unacceptable (shopping carts, collaborative state, anything with real business value per write) |

**Recommendation, matching doc 03's stated non-functional requirements:** vector clocks are the correct choice for a general-purpose store whose whole design premise (Section 1 of doc 03) is availability without silent data loss. LWW is a legitimate, simpler choice only for a specific, narrower class of data where the application has already decided that losing a stale concurrent write is an acceptable, harmless outcome — that's a call to make explicitly per-key-type, not a default to fall back on because it's simpler to implement.

---

## 6. Where this is heading next: CRDTs

Worth naming as a natural follow-up, since it's the direction the field has actually moved: **Conflict-free Replicated Data Types (CRDTs)** are data structures (counters, sets, maps) with merge functions that are mathematically guaranteed to converge to the same result regardless of the order operations are applied in — a **G-Counter** (grow-only counter) or an **OR-Set** (observed-remove set) can be merged automatically, correctly, with no siblings ever surfaced to the client at all. The trade-off: CRDTs only work for data types with a well-defined, associative-and-commutative merge operation baked in ahead of time (a set union, a counter sum) — they can't handle arbitrary opaque blobs the way vector-clock-plus-client-resolution can, so they're a narrower but more automatic tool, not a strict replacement.

---

## 7. Follow-up questions

- **"What happens to old siblings that never get read and merged?"** — same mechanism as doc 03 Section 6's anti-entropy: siblings sitting unread indefinitely are a real operational concern, and some systems apply a fallback (e.g., LWW as a last resort after a long enough sibling-age threshold) specifically to bound how long true ambiguity can persist, accepting the small data-loss risk only once the alternative (unbounded sibling accumulation) becomes the bigger problem.
- **"Does every key need vector clocks, or can you choose per key-type?"** — production systems commonly let this be configurable per column-family/bucket precisely because the matrix in Section 5 has genuinely different right answers for different data — this is the practical version of "don't default to the simpler option without deciding explicitly."
- **"How does this interact with N/W/R from doc 03?"** — directly: a higher W reduces the *frequency* of concurrent-write conflicts (more replicas must agree before a write succeeds, narrowing the window where two writes can both succeed without seeing each other), but never eliminates them as long as W < N — conflict resolution isn't an edge case, it's a permanent feature of any quorum system with W < N, which is precisely why doc 03 chose W=2 (not W=3) as its default in the first place.
