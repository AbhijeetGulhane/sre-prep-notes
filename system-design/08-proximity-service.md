# Proximity Service — Comprehensive NALSD Reference

*Eighth in the system-design set. Grounded in ByteByteGo's "Proximity Service" and "Quadtree" guides (bytebytego.com/guides), same content as the video, plus the broader geospatial-indexing literature for the S2 detail their guide only gestures at ("there are other more complicated algorithms... let me know if you're interested").*

*Callbacks throughout: [03.1-distributed-kv-store](./03.1-distributed-kv-store-nalsd.md) and [06-web-crawler](./06-web-crawler.md) (partition-by-natural-key over hash, a third time), [07-email-system](./07-email-system.md) (index freshness vs. staleness trade-off), [05-unique-id-generator](./05-unique-id-generator.md) (compact 64-bit IDs via clever bit-packing, S2's cell IDs are the same idea applied to geometry).*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
Following ByteByteGo's own clean split, worth preserving exactly:
- **Business Service**: add/update/delete point-of-interest (POI) records — standard CRUD, low write volume, not the interesting part of this design.
- **Location-Based Service (LBS)**: given a location and radius, return nearby POIs — the read path, and where all the design complexity actually lives.

### Non-functional requirements
- **Low latency** — this is an interactive, user-facing query (map pins, search-while-typing), not a batch job.
- **Read-heavy, by a wide margin** — searches vastly outnumber POI edits, the same read/write skew shape as doc 01's shortener, just for a different reason (people search constantly; businesses update their listing rarely).
- **Approximate is fine** — a "nearby restaurants" query doesn't need mathematically exact distance ranking; this permission to approximate is precisely what justifies grid/cell-based indexing over brute-force exact-distance computation, and it's worth stating explicitly as the requirement that licenses everything in Sections 3–5.

### Scale target
- **500 million POIs** worldwide.
- **100,000 search queries/sec** at peak.
- POI update rate: comparatively tiny (order 100s/sec) — not a capacity driver anywhere in this design.

### The problem, stated plainly
Storing raw `(lat, lng)` and computing the distance from the query point to *every* POI at query time is an O(n) scan per query — at 500M POIs and 100,000 queries/sec, that's not a design, it's a description of an outage. The fix in every technique below is the same underlying idea: **collapse 2D space into something that supports an efficient indexed lookup** — either a 1D string with prefix-search semantics, or a hierarchical tree/cell structure.

---

## 2. Option A: Geohash

### Mechanism (ByteByteGo's own construction, paraphrased)
Divide the globe into four quadrants along the prime meridian and equator, encoding each half with a bit (latitude below/above 0, longitude below/above 0). Recursively subdivide each resulting grid into four smaller grids, alternating which coordinate (longitude, then latitude) contributes the next bit. The resulting bitstring is grouped and Base32-encoded into a compact string — more characters means a smaller, more precise cell.

**Why this makes proximity search efficient**: nearby locations tend to share a common string **prefix**, so a proximity query becomes a prefix match (`geohash LIKE '9q8yy%'`) — directly servable by a standard B-tree index on the geohash column, no special spatial index engine required.

### Properties
- **O(1) to compute** — no rebalancing, no tree maintenance, just a direct calculation from coordinates. A strong fit if write volume were high (it isn't, for this design's numbers, but worth knowing as a general property).
- **Uneven density is a real limitation, stated directly in the source material**: a downtown grid cell can hold thousands of POIs while an ocean cell holds none — **the same hot-partition shape that's shown up in every prior doc in this series** (doc 02's abusive client, doc 03.1's popular record, doc 06's popular host, doc 07's hot mailbox), just manifesting geographically this time.
- **Boundary problem, worth naming explicitly since it's easy to miss**: two points meters apart but straddling a grid-cell edge can have completely different prefixes — a naive "query my own cell's prefix" search silently misses genuinely nearby results across that boundary. The standard fix: search the point's own cell **plus its 8 neighboring cells** (a 3×3 search), not just the one the query point happens to fall in.

---

## 3. Option B: Quadtree

### Mechanism (ByteByteGo's "Quadtree" guide)
Start with a single root node representing the entire map. Recursively subdivide any node into four children whenever it holds more than a set threshold of points. **Subdivision is data-driven, not uniform** — this is the direct fix for geohash's density problem: a dense downtown area gets many small, deep cells; a sparse ocean region stays one large, shallow cell, and every leaf ends up holding a roughly bounded number of points regardless of geography.

**A specific architectural detail worth being precise about, straight from the source**: a quadtree here is explicitly an **in-memory structure, not a database index** — each LBS server builds its own full tree in RAM from the underlying POI store at startup, and serves queries directly from that in-memory structure for low latency.

### The freshness question this raises
If the tree is built once at startup from the database, **a newly added business isn't searchable until the tree is rebuilt or incrementally updated** — the exact same staleness trade-off doc 07's search-indexing section faced for email, just applied to POI data instead of message content. Production systems handle this with either periodic full rebuilds, incremental in-memory tree updates on write, or — matching doc 07's approach — accepting a bounded amount of staleness as a deliberate trade rather than a bug, given Section 1 already established this system tolerates approximate results.

### Properties
- **O(log n) insertion/query** — more computationally involved than geohash's O(1), in exchange for naturally balanced handling of skewed real-world data.
- **Does this actually fix pole distortion?** Worth being honest here: a classic quadtree built directly over a flat lat/lng bounding box, the way both this design and the source material describe it, **doesn't geometrically fix the same pole-convergence distortion geohash has** — it's still a flat-plane subdivision. What it does fix is the *practical* symptom (uneven point density per cell), because real-world POI data simply isn't spread near the poles anyway. That's a meaningfully different claim from "quadtree solves distortion," and it's worth not overstating the difference to an interviewer.

---

## 4. Option C: S2 (Google's choice)

### Mechanism
Rather than working in flat latitude/longitude at all, S2 **projects the sphere onto a cube** (six faces), recursively subdivides each face into a quadtree-like cell hierarchy (levels 0 through 30, coarsest to finest), and — the genuinely distinctive part — **linearizes that hierarchy using a Hilbert space-filling curve** into a single 64-bit cell ID.

**Why the Hilbert curve matters, specifically**: it's constructed so that cells with numerically close 64-bit IDs are reliably geographically close too, including across what would otherwise be awkward cell boundaries under a simpler bit-interleaving scheme like geohash's. This directly improves on Section 2's boundary problem — it doesn't eliminate the need to check neighboring cells entirely, but the locality guarantee is materially stronger.

**Why sphere-based, specifically**: geohash and quadtree both operate on a flat lat/lng plane, where lines of longitude visibly converge near the poles — a real, geometric distortion baked into the model itself, not just a data-density artifact. S2's cube-sphere projection keeps cell *area* far more consistent globally, which is the genuine, principled fix Section 3 couldn't honestly claim for quadtree.

**A nice callback**: S2's 64-bit cell ID is a compact, directly-indexable integer — conceptually the same move as doc 05's Snowflake ID, just packing geometric locality into the bits instead of temporal ordering and machine identity.

### Why this is Google's specific choice
Google Maps/Earth and BigQuery GIS all use S2. The added complexity (cube projection, Hilbert curve construction) is justified specifically at **true global, pole-inclusive, all-latitude scale** — worth naming as a deliberate match-the-tool-to-the-actual-requirement decision, the same instinct doc 04's cache-vs-KV-store reasoning made: S2's sophistication is real engineering cost, appropriate at Google's actual scale, and likely over-engineering for a smaller or regionally-scoped service.

---

## 5. Comparison and Recommendation

| | Geohash | Quadtree | S2 |
|---|---|---|---|
| Structure | Fixed grid, bit-interleaved string | Adaptive tree, data-density-driven | Sphere→cube→cell hierarchy, Hilbert-linearized |
| Write cost | O(1) | O(log n) | O(1)-ish (direct cell computation) |
| Uneven density | Poor — hot cells, empty cells | **Good** — adaptive subdivision bounds points/leaf | Good — adaptive levels, plus true area uniformity |
| Pole/global distortion | Poor | Not actually fixed, just less visible with real data | **Best** — genuinely sphere-native |
| Where it lives | DB-indexable string column | In-memory, per-server, built at startup | DB-indexable 64-bit integer |
| Boundary correctness | Needs explicit 3×3 neighbor-cell search | Natural via tree traversal | Better locality via Hilbert curve, still needs a region-cover step |
| Best fit | Simple, DB-native, write-heavy scenarios | Skewed real-world density, in-memory low-latency serving | True global scale, pole-inclusive, highest accuracy needs |

**Recommendation for this design's actual numbers**: use a **quadtree as the primary in-memory serving structure** on each LBS server (Section 3) — it directly addresses the density skew that Section 1's target data (real businesses, real cities) will actually exhibit, and low query latency is a stated non-functional requirement. **Persist the underlying POI data with a geohash column** in the durable store, so it stays efficiently range/prefix-queryable independent of any one server's in-memory tree, and so each LBS server has an efficient way to (re)build its tree from storage. **Reserve S2** specifically for the tier this system isn't targeting — true global, pole-inclusive, Google-Maps-class accuracy — rather than defaulting to it everywhere; that's a legitimate, honest scoping decision, not a cop-out.

---

## 6. Partitioning — By Region, Not by Hash, for the Third Time in This Series

If a single LBS server's in-memory quadtree can't hold the whole world's POI data (Section 7 checks this directly), the natural partition key is **geographic region** — shard by continent or country, not by `hash(POI id)`.

This is the same underlying lesson as doc 07's range-partition-by-user choice, and doc 06's partition-by-host: **the dominant access pattern determines the right partition key, and it isn't always a hash.** A search centered on Manhattan has no reason to touch a shard holding Australian POIs — hash-partitioning would scatter geographically related data across the cluster for no benefit, exactly the failure mode doc 07's Section 4 argued against for mailboxes. Here, partitioning by region is **both** a memory-capacity decision and a query-relevance decision pointing the same direction, which is a stronger justification than either one alone.

---

## 7. Capacity Math and the First Real Bottleneck

### In-memory index footprint
```
500,000,000 POIs × ~100 bytes/tree-node (lat/lng + id + child pointers)
= 50,000,000,000 bytes ≈ 50 GB
```
**Fits on a single modern high-memory machine (64–128GB is standard) — but tightly, with no growth headroom and no redundancy for that one machine's failure.** This is the concrete trigger for Section 6's regional sharding: not because 50GB is unmanageable in the abstract, but because a single point of failure holding the *entire* world's search index, with zero room to grow, is an obviously bad plan the moment it's written out as a number rather than left as a vague "it's a lot of data."

### Query throughput
```
100,000 peak queries/sec ÷ 5,000 queries/sec/node
  (conservative — in-memory tree traversal itself is microsecond-scale;
   this accounts for network and serialization overhead, not raw compute)
= 20 nodes
```
Trivial to satisfy — the same "throughput was never the hard part" finding as docs 05 and 06.

### The first real bottleneck: query-time variance driven by the same density skew
Raw QPS capacity isn't where this design actually strains. **The same density skew that motivated quadtree over geohash in Section 5 resurfaces at query time, in two different shapes:**
- **Dense-area queries** (a search centered on Times Square): the relevant cell can hold thousands of candidate POIs — the bottleneck here is **ranking/filtering cost**, CPU-bound on scoring a large candidate set, not on traversal.
- **Sparse-area queries** (a search in a rural area with almost no nearby cells populated): the bottleneck is the **opposite shape** — the tree traversal has to walk out across many parent/neighbor cells to gather enough results to return anything useful at all, an I/O-and-traversal-count problem, not a scoring problem.

**Both are real, and they're two faces of the same underlying data property** — worth stating explicitly rather than treating "handle the dense case" and "handle the sparse case" as unrelated engineering tasks.

### Iterate
- **Dense case**: bound the worst case by fetching a capped top-K from an oversaturated cell (ranked by a secondary relevance signal), rather than returning or even fully scoring every POI in the cell — the same "cap it and rank" instinct as doc 04's approximate-LRU sampling, applied to search results instead of eviction candidates.
- **Sparse case**: **dynamic radius expansion** — if the immediate cell (or its Section 2/3 neighbor set) returns too few results, progressively widen the search to parent cells or a larger cell radius until a useful result count is reached, rather than either failing the query or paying the traversal cost of an unnecessarily wide search on every request regardless of local density.

---

## 8. Further Reading

- **ByteByteGo's "Proximity Service" and "Quadtree" guides** (bytebytego.com/guides) — **free**, **essential** — the direct source for Sections 2 and 3's mechanics.
- **s2geometry.io** — Google's own S2 documentation — **free**, **essential** if asked any implementation-level S2 question beyond Section 4's overview.
- **Hilbert curve** (Wikipedia's article has a genuinely solid visual/mathematical treatment) — **free**, **supplementary** — worth a skim to actually see why the curve preserves locality rather than taking Section 4's claim on faith.
- **Redis `GEOHASH`/`GEOADD` command docs** — **free**, **supplementary** — a concrete, practical reference for how geohash gets implemented in a real, widely-used system rather than only in the abstract.

---

## 9. Summary

| Dimension | Value | Binding constraint? |
|---|---|---|
| In-memory index size (global) | ~50 GB | Fits one machine, but no headroom — the real trigger for regional sharding |
| Query throughput | 20 nodes for 100,000/sec peak | No — trivial, same as docs 05 and 06 |
| **Query-time variance (dense vs. sparse)** | **Two opposite failure shapes from one root cause** | **Yes — the actual first bottleneck, not raw capacity** |

**What ties this topic to the rest of the series**: the recurring hot-key/hot-partition pattern shows up a fourth time here, but in a new form — not a hot *key* or hot *host*, but a hot *geographic region*, and for the first time, the same skew that causes a storage-side problem (Section 2's uneven grid density) is shown to cause a **query-side** problem too (Section 7's dense/sparse variance), rather than those being two separate issues.

## 10. Follow-up Questions

- **"How do you handle a moving user (e.g., a rider tracking a driver)?"** — a different problem from static POI search: frequent location *writes* rather than reads, favoring geohash's O(1) write cost or a purpose-built structure like Uber's H3 over quadtree's O(log n) update cost — worth naming as a genuinely different sub-problem this design didn't target, per Section 1's scope (POI search, not live tracking).
- **"Why not just use PostGIS or a database's native geospatial index?"** — a legitimate, simpler alternative for smaller scale; this design's in-memory quadtree tier exists specifically because Section 1's latency and QPS targets favor avoiding a database round-trip on the hot query path entirely, the same reasoning doc 04's cache-aside pattern used to justify an in-memory layer in front of a durable store.
- **"What if a business has multiple locations spanning regions (e.g., a chain)?"** — each location is its own independent POI record in this model, naturally falling into whichever regional shard it geographically belongs to — no special handling needed, since the partition key (Section 6) is location, not business identity.
