# DDIA Ch 10 — Consistency and Consensus

> 2nd edition numbering (was Ch 9 in 1st ed). Read for **concepts and recognition**; skim step-by-step Raft/Paxos protocol traces.
> Interview level: *"Does this system need consensus, and what does linearizability cost you?"*

---

## The one distinction to nail: linearizability vs eventual consistency

**Linearizability** (strong consistency): the system behaves as if there's **one single copy** of the data and every operation is atomic and instantaneous. Once a write completes, every subsequent read — on any node — sees it. "Recency guarantee."

**Eventual consistency:** replicas converge *eventually* if writes stop, but reads can be stale in the meantime. This is what most replicated/leaderless systems give you by default.

**The tradeoff:** linearizability is expensive — it requires coordination, hurts latency, and (per CAP) becomes unavailable during network partitions. Eventual consistency is cheap, fast, available under partition, but exposes staleness to the application.

**Interview-ready:** *"Linearizability means every read sees the latest write as if there's one copy. It costs coordination, latency, and availability under partition — so you only pay for it where correctness demands it (locks, leader election, uniqueness), not for everything."*

---

## CAP — state it correctly or not at all

CAP is widely mangled. The precise version:

> When a network **partition** (P) happens, you must choose between **consistency** (linearizability) and **availability**. When there's *no* partition, you can have both.

So CAP is really "**during a partition**, C or A." It says nothing useful when the network is healthy, and "pick 2 of 3" is a misleading framing. Knowing this precisely is a signal; parroting "pick 2 of 3" is a mild anti-signal.

---

## Ordering and causality

- **Total order:** every operation is comparable (linearizable systems give this).
- **Causal order:** only causally-related operations are ordered; concurrent ones aren't. Weaker but **much cheaper** and often sufficient.
- **Lamport timestamps / version vectors:** mechanisms to capture causal ordering. Know they exist and what they're for; skim the construction.

Causal consistency is the strongest model that stays available under partition — a useful "middle ground" answer when asked what to use when linearizability is too expensive.

---

## Consensus — what it actually is

Consensus = getting several nodes to **agree on a single value** even with failures. It sounds narrow but a huge number of problems reduce to it:

- **Leader election** (who's the single leader? avoid split brain)
- **Atomic commit** (do all nodes commit this distributed transaction, or none?)
- **Uniqueness constraints** (only one user gets this username)
- **Lock/lease granting**

**The key insight for interviews:** these are all "the same problem" — they all need the cluster to agree despite failures, and that's consensus. Recognising when a design *needs* consensus is far more valuable than reciting the protocol.

---

## The consensus algorithms (SKIM the traces)

- **Paxos** — the original, famously hard to understand.
- **Raft** — designed to be understandable; leader-based; the one to reference. Used by etcd, Consul.
- **Zab** — ZooKeeper's protocol.

You should be able to say: *"Raft elects a leader, replicates a log to a majority, and a write commits once a quorum acks it; if the leader dies, a new election runs."* That's the right depth. **Do not** memorise the message-by-message flow, term-increment rules, or log-matching proofs for an SRE loop — that's implementer depth.

---

## Coordination services — where this shows up in real systems

**ZooKeeper** and **etcd** are consensus-as-a-service. Systems outsource their hard coordination to them:
- Kubernetes stores all cluster state in **etcd** (Raft).
- Kafka historically used **ZooKeeper** for controller election and metadata.
- Partition-routing metadata (Ch 7) and leader election (Ch 6 failover) live here.

**Interview-ready:** *"Most systems don't implement consensus themselves — they push it to ZooKeeper or etcd for leader election, config, and locks. Kubernetes is etcd; that's Raft underneath."*

This closes the loop across the whole distributed-systems arc: Ch 6 failover needs a leader → electing one needs consensus → consensus is provided by etcd/ZooKeeper → which also holds the Ch 7 routing metadata.

---

## Interview-ready summary

- Linearizability = one-copy illusion, recency guarantee; costs coordination, latency, availability-under-partition.
- Eventual consistency = cheap, fast, stale reads possible.
- CAP precisely: **during a partition**, choose C or A. Not "pick 2 of 3."
- Causal consistency = strongest model available under partition.
- Consensus = agreement despite failure; leader election / atomic commit / uniqueness / locks all reduce to it.
- Raft is the reference algorithm; etcd & ZooKeeper are consensus-as-a-service. K8s = etcd.

---

## Reading resources

- **DDIA 2nd ed, Ch 10** — primary. Read linearizability, CAP, and the "what needs consensus" framing; skim protocol internals. (paid)
- **"The Raft paper" (raft.github.io)** + the **secret lives of data Raft visualisation** (thesecretlivesofdata.com/raft) — free — the visualisation gives you the right intuition in 10 minutes without the proof depth. (free, essential)
- **Kleppmann "Please stop calling databases CP or AP"** blog — free — sharpens the CAP framing so you don't parrot the wrong version. (free)
- **etcd / ZooKeeper docs — overview pages only** — free — concrete grounding for the coordination-service answer. (free)
- Skip for now: Paxos Made Simple, full Raft proofs (implementer depth, not needed for SRE).
