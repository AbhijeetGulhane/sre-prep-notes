# Raft: Understandable Distributed Consensus

> Week 7 material (DDIA Ch9 / Raft video), pulled forward. Leader election, log replication, commitment, and the leader-election state machine.

## Why Raft exists, and why not just Paxos

**The problem (consensus):** distributed nodes need to agree on a single, consistent sequence of operations (a replicated log) despite crashes, restarts, and network partitions — with no single point of failure and no disagreement about what happened. The actual mechanism underneath etcd, ZooKeeper, and any strongly-consistent distributed database's replication.

**Why Raft over Paxos:** Paxos is correct but notoriously hard to understand/implement — most "Paxos" systems in production actually run heavily modified variants because the base protocol is impractical to implement directly. Raft's explicit design goal was **understandability** — decomposing consensus into three separable subproblems (leader election, log replication, safety) reasoned about independently, vs Paxos's more monolithic approach. Not a stronger guarantee, a practical/pedagogical win — which is exactly why etcd, Consul, and CockroachDB chose it.

## The three node states — the core state machine

- **Follower** — passive, responds to Leader/Candidate requests, never initiates. Default/resting state, every node starts here.
- **Candidate** — actively campaigning to become Leader, mid-election.
- **Leader** — handles all client requests, replicates log entries to Followers. At most one Leader per term.

## Terms — Raft's logical clock

Monotonically increasing integers; at most one Leader per term (possibly zero, on split vote). Every message carries the sender's term. **Any node seeing a higher term than its own immediately updates its term and steps down to Follower** — even a current Leader, since a higher term means something moved on without it and continuing to act as Leader would be unsafe.

## Leader election — step by step

1. **Election timeout** — every Follower runs a randomized timer (typically 150-300ms). Randomization specifically reduces the chance of simultaneous elections/repeated split votes — not incidental.
2. **Follower → Candidate** — increments own term, votes for self, sends `RequestVote` RPCs to all other nodes in parallel.
3. **Voting** — a node grants its vote to the first Candidate it hears from in a term, IF: (a) hasn't already voted this term, AND (b) Candidate's log is at least as up-to-date as its own (the actual safety mechanism — see below). One vote per node per term — makes majority quorums meaningful.
4. **Winning** — Candidate becomes Leader on receiving votes from a **majority** (including self-vote). 5-node cluster = 3 votes. Guarantee: any two majorities in the same cluster overlap by ≥1 node, so **two Candidates can't both win a majority in the same term** — pigeonhole principle, not explicit coordination.
5. **Split votes** — votes divided across multiple Candidates → no majority → term expires without a Leader → new election, incremented term. Randomized timeout is what makes *repeated* split votes unlikely.
6. **Leader established → heartbeats** — Leader sends periodic empty `AppendEntries` RPCs (heartbeats), resetting Followers' election timeouts. **An election only starts when heartbeats stop** — Leader crash, network partition isolating the Leader, or Leader too overloaded to send heartbeats in time.

## Log replication

Only the Leader accepts client writes. Each write → a log entry (command + term created in), appended to Leader's log first, then replicated via `AppendEntries` RPCs (heartbeats = `AppendEntries` with empty entries list — same RPC type).

**Consistency check:** every `AppendEntries` includes the index+term of the entry immediately preceding the new ones. Follower rejects if its log doesn't have a matching entry there — detects/repairs inconsistencies (missed entries, conflicting entries from an old failed Leader) without a separate repair protocol: on rejection, Leader decrements the sync index and retries, walking backward until logs agree, then overwrites everything after that point on the Follower.

## Commitment — when a write is actually durable

An entry is **committed** once replicated to a **majority** — durable even if the Leader immediately crashes (a majority holds it, and any future Leader must also win a majority vote, which necessarily overlaps with ≥1 node holding this entry). Leader tracks highest committed index, includes it in subsequent `AppendEntries`, telling Followers "safe to apply to your state machine now."

**Answer to "when is a write durable in Raft":** not at client-return time (too early, no replication yet), not on every node (too strict, sacrifices availability for one slow/down node) — specifically majority replication, the same quorum tradeoff throughout distributed systems.

## Election safety guarantee — the unified mechanism

**"Can a Candidate with stale data become Leader?"** — No. During voting, a node compares the Candidate's last log entry (term + index) against its own; grants a vote only if the Candidate's log is at least as up-to-date (higher last-log term, or equal term with ≥ as many entries). Since winning requires a majority, and a majority necessarily includes ≥1 node holding the most recently committed entry (same overlap guarantee as leader uniqueness), **a Candidate missing a committed entry cannot win a majority** — the node holding that entry refuses to vote for it. Leader election and log safety are one unified guarantee, not two separate mechanisms that happen to cooperate.

## The leader-election state machine

```
                    ┌─────────────────────────────────────┐
                    │                                      │
                    ▼                                      │
     start ──▶ [FOLLOWER] ──election timeout──▶ [CANDIDATE]│
                    ▲                               │  │   │
                    │                               │  │   │
        discovers current Leader                    │  │   │
        OR higher term                               │  │   │
                    │                               │  │   │
                    └───────────────────────────────┘  │   │
                    (via any state, on higher term seen)│   │
                                                          │   │
                            wins majority vote ───────────┘   │
                                    │                          │
                                    ▼                          │
                               [LEADER] ──sends heartbeats─────┘
                                    │            (resets Followers'
                                    │             election timers)
                            steps down on
                            discovering higher term
                                    │
                                    ▼
                               [FOLLOWER]
```

Three states, five transitions:

1. **Follower → Candidate**: election timeout expires. Increments term, votes for self, sends `RequestVote` to everyone.
2. **Candidate → Leader**: receives majority votes. Starts heartbeats immediately.
3. **Candidate → Follower**: discovers another node already won this term (valid heartbeat from new Leader), OR discovers a higher term than its own from any message.
4. **Candidate → Candidate** (self-loop, often forgotten): election timeout expires again mid-election with no winner (split vote) — increments term again, restarts.
5. **Leader → Follower**: discovers a higher term than its own — steps down immediately regardless of confidence.

**Detail that trips people up:** no direct `Follower → Leader` arrow, no direct `Leader → Candidate` arrow. Every path to Leader goes through Candidate; every path away from Leader goes through Follower first (a demoted Leader doesn't skip straight to campaigning — it becomes Follower, and only becomes Candidate again if it independently times out later).

## Interview one-liners, compiled

- "Raft's actual innovation over Paxos isn't stronger guarantees — it's decomposing consensus into leader election, log replication, and safety as separately understandable pieces, which is why it won in practice: etcd, Consul, and CockroachDB all chose it specifically for implementability."
- "Randomized election timeouts exist specifically to prevent repeated split votes — different random timers mean one Follower usually gets a head start collecting votes before a competing candidacy can even begin."
- "At most one Leader per term is guaranteed by the pigeonhole principle, not explicit coordination — any two majorities in the same cluster must overlap by at least one node, so two Candidates can't both win a majority in the same term."
- "A log entry is committed once replicated to a majority, not once it's on every node — the same availability/durability quorum tradeoff that shows up everywhere in distributed systems, and it's what makes a committed write survive even an immediate Leader crash."
- "A Candidate with stale or incomplete log data literally cannot win an election — voters refuse a Candidate whose log isn't at least as up-to-date as their own, and since winning requires a majority, at least one voter is guaranteed to be holding the most recent committed entry."

---
*Pulled forward from Week 7 (DDIA Ch9 + Raft video slot). DDIA Ch5-8 (Replication, Partitioning, Transactions, Trouble with Distributed Systems) and the SE Daily consensus podcast remain in their originally scheduled Week 7 slot.*
