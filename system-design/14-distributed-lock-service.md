# Distributed Lock Service (Chubby/ZooKeeper-style) — Comprehensive NALSD Reference

*Fourteenth in the system-design set. This is the doc that finally opens the box every prior doc left closed — [05-unique-id-generator](./05-unique-id-generator.md)'s machine-ID assignment, and the implied coordinators in [06-web-crawler](./06-web-crawler.md) and [07-email-system](./07-email-system.md), all referenced "Zookeeper" as a black box. This is what's inside it.*

*Also the doc that completes a three-way survey this series never assembled explicitly until now: [03.1-distributed-kv-store](./03.1-distributed-kv-store-nalsd.md)'s AP quorum-plus-vector-clocks, [09-video-platform](./09-video-platform.md)'s TrueTime-plus-Paxos, and this doc's majority-vote-consensus — three genuinely different mechanisms for getting multiple machines to agree on the truth.*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- **Acquire/release a distributed lock** — mutual exclusion across many independent processes/machines.
- **Leader election** — in practice, nearly the same primitive as a lock (Section 7).
- Secondary uses: configuration storage, service discovery.
- **Session-based liveness** — a crashed client's locks must release automatically, never hang forever waiting for an explicit unlock that will never come.
- **Notify waiters** when a lock becomes available, without requiring them to poll.

### Non-functional requirements — a genuinely different optimization target than every prior doc
- **Strong consistency is non-negotiable.** A lock service that could hand the same lock to two clients simultaneously is worse than no lock service at all — direct, deliberate contrast to doc 03.1's AP design, where tolerating and merging concurrent conflicting writes was the whole point. Here, "let's merge two conflicting lock grants" is meaningless.
- **High availability despite requiring strong consistency** — the classic CAP tension, resolved here through a third mechanism this series hasn't covered yet (Section 3).
- **This system is deliberately small and low-throughput by design**, not a scale target to optimize upward. Worth stating plainly before any capacity math: Chubby and Zookeeper are famous for being small, tightly-bounded coordination-plane services (a handful of nodes) sitting in front of much larger data-plane systems — not systems that themselves need to scale to millions of QPS.

### Scale target
- A **5-node ensemble** (odd-numbered, standard convention — Section 8 explains exactly why).
- **~10,000 ops/sec** (lock acquires, releases, watches, heartbeats combined) — deliberately modest, and, as Section 8 shows, this number turns out to barely matter to how the system is actually sized.

---

## 2. High-Level Architecture

```
Client library (session management, heartbeats, auto-reconnect)
        │
        ▼
   5-node consensus ensemble (Paxos-family — ZAB for Zookeeper, Paxos for Chubby)
        │
   One elected LEADER at any time
        │
   Every write: leader proposes → replicated to a MAJORITY of nodes → committed → acknowledged
   Reads: served by any node (with a staleness caveat) or the leader for strict consistency
```

---

## 3. Why Consensus, Not Doc 03.1's Quorum or Doc 09's TrueTime — the Third Mechanism

This series now has three genuinely different answers to "how do multiple machines agree on the truth," worth naming side by side explicitly rather than leaving them as three separate memories:

| | Mechanism | How disagreement is handled |
|---|---|---|
| Doc 03.1 (KV store) | Dynamo-style quorum (W+R>N) + vector clocks | **Tolerated and merged** — concurrent writes become siblings, resolved by the application |
| Doc 09 (video platform, metadata) | TrueTime (globally synchronized, bounded-uncertainty clocks) + Paxos | **Avoided via synchronized clocks** — real infrastructure investment (atomic/GPS clocks) buys a single global ordering |
| This doc (lock service) | Paxos/Raft/ZAB consensus, majority-vote leader election | **Avoided via a strict elected leader** — no synchronized clocks needed, no merge semantics; a majority vote makes only one decision possible |

**The actual split-brain-prevention mechanism, precisely**: in an ensemble of size N, any two majorities **must** overlap by at least one node — this is just arithmetic (two groups each larger than N/2 out of N total cannot be disjoint). That guaranteed overlap is what makes it structurally impossible for two different nodes to simultaneously believe they're the leader and both get a majority to agree with them: any node that would vote for a second, competing leader is necessarily also a member of the first leader's majority, and a correctly implemented consensus protocol ensures it can't do both. **Worth being precise that this is a different mechanism from "quorum" as doc 03.1 used the term** — there, a quorum tolerates multiple valid concurrent states; here, a quorum guarantees only one state transition can ever be agreed on at all.

**Fencing tokens (Section 5) are a separate, complementary defense layered on top of this — not a replacement for it.** Worth flagging that distinction now, since it's easy to conflate the two and treat fencing tokens as "the" split-brain fix when they solve a narrower, different problem.

---

## 4. Ephemeral Nodes and Session Leases

**Mechanism**: a client acquires a lock by creating an **ephemeral node** tied to its **session**, not to any single request or connection. The client (or its underlying connection) must periodically renew that session via heartbeats. If the session lease expires without renewal — client crash, network partition, a hung process — the service **automatically deletes all of that client's ephemeral nodes**, including its lock, with no explicit unlock call required.

**Why not a simple TTL on the lock key itself** (the doc 03.1/doc 04-style pattern)? A naive TTL has a specific, nasty failure mode worth stating precisely: if the client is merely **slow** (a long GC pause, a CPU stall) rather than actually dead, a TTL would release its lock **while the client is still alive and still believes it holds it** — exactly the split-brain scenario this whole design exists to prevent. The session-lease model narrows the risk window (bounded by the session timeout) but **does not eliminate it** — a session can be declared dead by the server while the client, mid-pause, has no way of knowing yet. This is precisely the gap Section 5 exists to close, and it's worth being honest that ephemeral nodes plus session leases alone are **not sufficient** to fully prevent split-brain, contrary to a common oversimplified telling of how this works.

---

## 5. Fencing Tokens — the Actual Defense Against "Slow, Not Dead"

### The scenario, worked through precisely (Martin Kleppmann's now-classic example)
```
1. Client 1 acquires the lock.
2. Client 1 experiences a long GC pause, exceeding the session lease timeout.
3. The lock service, seeing no heartbeat, expires Client 1's session and releases the lock.
4. Client 2 acquires the lock and begins writing to the shared protected resource.
5. Client 1 resumes from its pause — still believing it holds the lock, having received
   no notification otherwise (or the notification hasn't been processed yet) — and ALSO
   writes to the same protected resource.

Both clients are now writing concurrently, each believing it exclusively holds the lock —
genuine data corruption, despite the lock service having behaved entirely correctly
by its own session-timeout rules the whole time.
```

### The fix
Every successful lock grant is issued with a **monotonically increasing fencing token** (a simple integer, incremented on every grant). The client must attach this token to every subsequent write it makes **to the protected resource** — not to the lock service. **The protected resource itself, not the lock service, is responsible for rejecting any write bearing a token lower than the highest one it has already seen.**

```
Client 1 granted the lock: token = 33
Client 1 pauses; session expires
Client 2 granted the lock: token = 34   (strictly greater — tokens only increase)
Client 2 writes with token=34; the storage system records 34 as its high-water mark
Client 1 resumes, writes with token=33
The storage system sees 33 < 34 (already observed) and REJECTS the write
```

**A precise point worth stating explicitly, since it's commonly oversimplified**: fencing tokens don't prevent the race from *occurring* — Client 1's stale write attempt still happens. They prevent the race from causing **harm**, by pushing enforcement to the actual point of resource contention. **This means fencing tokens are not a transparent, drop-in fix** — every downstream system genuinely protected by the lock has to be modified to check and reject stale tokens. This is a real integration cost, not a free correctness upgrade, and it's the same honest shape as doc 12's "trace_id must be threaded through every layer of logging code" caveat: a theoretically sound mechanism that only actually works if every touchpoint disciplines itself to participate in it.

---

## 6. Watches — Avoiding Polling

Rather than clients polling to check whether a lock has been released, the service supports **watches**: a client registers interest in a specific node and receives a one-time asynchronous notification when it changes (e.g., when the ephemeral lock node is deleted on session expiry or explicit release). This lets waiting clients react promptly without wasting cycles on repeated polling — a real efficiency mechanism, worth naming even though it's not one of this doc's crux decisions.

---

## 7. Leader Election — the Real-World Use Case This Series Already Assumed

**"Distributed lock" and "leader election" are nearly the same primitive in practice**, worth stating explicitly: a set of candidate processes all attempt to create the *same* ephemeral node (e.g., `/leader`); the ensemble's own consensus guarantees exactly one succeeds, and that candidate becomes leader. If the leader's session expires, its ephemeral node is deleted automatically, and the remaining candidates race to create it again.

**This closes a loop across the whole notes repo**: doc 05's machine-ID assignment via Zookeeper ephemeral sequential nodes, and the implied coordination needs in docs 06 and 07, were all leaning on exactly this mechanism without this doc having existed yet to explain it. Worth reading this as the payoff for those earlier "a Zookeeper-like service handles this" hand-waves.

---

## 8. Capacity Math — Where "Add More Nodes" Stops Working

### The instinct from every prior doc doesn't apply here
A single Paxos/ZAB leader, given that every write must be synchronously replicated to a majority before acknowledgment, realistically sustains somewhere in the range of **10,000–50,000 ops/sec** for small-value writes on modern hardware — this is a well-known, deliberate characteristic of these systems: they're built for coordination-plane traffic, not data-plane volume. **Even at the low end of that range, a single 5-node ensemble already meets this doc's entire 10,000 ops/sec target.**

**Here's the genuinely important finding**: adding *more* nodes to this ensemble does **not** increase throughput — writes still funnel through one leader and must be replicated to a majority, and a **larger** ensemble means a **larger** majority is required per write, which is *more* synchronous replication work, not less. **This is the exact opposite relationship every prior doc's node-count math assumed** (more nodes = more capacity). Worth dwelling on this explicitly as a deliberate counter-example closing out the series' recurring "scale by adding nodes" instinct: here, adding nodes trades **availability** (tolerating more simultaneous failures) for **worse** write latency, not more capacity.

### The actual sizing formula for this system
```
An ensemble of odd size N tolerates (N-1)/2 simultaneous node failures
while still retaining a majority to keep operating:

N=3: tolerates 1 failure
N=5: tolerates 2 failures
N=7: tolerates 3 failures
```
This — fault tolerance, not throughput — is the real sizing question for this doc, replacing every prior doc's "nodes = load ÷ per-node capacity" formula entirely.

### The first real "bottleneck" — and it isn't a capacity problem at all
If the ensemble is spread across multiple data centers/regions for disaster tolerance (a real, common and sensible deployment choice — never put all consensus nodes in one building), **every write now requires a round trip to a majority of nodes**, bounding write latency by the round-trip time to the slowest node in whatever majority happens to respond:
```
Illustrative real-world round trips: NYC↔London ≈ 70ms, NYC↔Tokyo ≈ 150ms+
```
**This is a pure speed-of-light/network-topology constraint, not a resource-capacity one — no amount of additional machines, faster hardware, or engineering effort routes around it.** Worth naming this as a *different kind* of unfixable bottleneck than doc 06's per-host politeness ceiling: doc 06's was a policy limit; this one is physics. Both share the "no engineering workaround" property, but for entirely different underlying reasons — worth having that distinction precise if asked to compare bottleneck types across the series.

### Iterate
Keep the consensus ensemble's members within a single region/metro area (minimizing the latency the consensus protocol itself pays), while still spreading across availability zones within that region to tolerate a full AZ failure. If genuinely global, low-latency coordination is required, the honest answer is a **different architecture entirely** — regional lock services, each independently authoritative for its own region's resources, rather than one global ensemble every write must reach. This trades global coordination guarantees for local latency deliberately and explicitly — not a way around the physics, a different scope of problem being solved instead.

---

## 9. Further Reading

- **"The Chubby lock service for loosely-coupled distributed systems"** (Burrows, Google) — **free**, **essential** — the original source system this entire doc describes.
- **"In Search of an Understandable Consensus Algorithm"** (Ongaro & Ousterhout — the Raft paper) — **free**, **essential** — a more approachable modern alternative to Paxos, worth knowing even though Chubby/Zookeeper predate it and use Paxos-family protocols specifically.
- **Martin Kleppmann, "How to do distributed locking"** (blog post) — **free**, **essential** — the direct source of Section 5's GC-pause fencing-token scenario.
- **DDIA, Chapters 8–9** (already in your library) — **paid**, **essential** — the broader distributed-systems and consensus fundamentals underlying this entire doc.

---

## 10. Summary

| Dimension | Value | Binding constraint? |
|---|---|---|
| Ensemble size | 5 nodes | Fault tolerance (2 simultaneous failures), not throughput |
| Throughput | 10,000–50,000 ops/sec from a *single* leader | No — already exceeds the target; adding nodes doesn't help this dimension at all |
| **Cross-region write latency** | **70–150ms+, bounded by physics** | **Yes — and unlike every prior doc's bottleneck, no amount of engineering effort changes this number** |

**What this doc changes about the series' running "add nodes to scale" instinct**: every prior topic's math pass found a bottleneck that more resources, better algorithms, or a redesign could move. This is the first system in the series where **adding nodes makes the actual bottleneck (write latency) worse, not better** — a genuinely important exception worth having sharp, not smoothed over into "just scale it" muscle memory built up over the rest of this notes repo.

## 11. Follow-up Questions

- **"Why do Chubby/Zookeeper use an odd number of nodes specifically?"** — an even-sized ensemble gains no fault-tolerance benefit over the next-smaller odd size (a 4-node ensemble tolerates only 1 failure, the same as 3 nodes, while paying the replication cost of a 4th node for nothing) — odd sizes are the efficient choice, not a superstition.
- **"What happens during leader election itself — is the system unavailable?"** — briefly, yes: while the ensemble is between leaders (an existing leader's session expiring, or a new one being elected), write availability is paused until a new leader is established and reaches majority — a real, bounded availability gap, not a hidden flaw, and part of why keeping the ensemble in one region (Section 8) matters for minimizing how long that gap lasts.
- **"How does this compare to just using the doc 03.1 KV store with a 'first writer wins' convention for a lock key?"** — it fundamentally doesn't provide the same guarantee: doc 03.1's AP design can, under partition, let two different nodes both believe they accepted a write to the same key (resolved later via vector-clock siblings) — exactly the split-brain outcome a lock exists to prevent. A lock service's CP guarantee is the entire point; reusing an AP store for this purpose reintroduces the problem it's meant to solve.
