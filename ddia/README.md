# DDIA Notes (2nd Edition)

Interview-calibrated notes on *Designing Data-Intensive Applications* (Kleppmann, 2nd ed). Written for the **SRE / SysEng interview level** — tradeoffs, failure modes, and the "when do I need this?" recognition — not implementation depth. Each file flags what to **read** vs **skim**.

Chapter numbers follow the **2nd edition** (Encoding = Ch 5, so everything is +1 vs the 1st edition).

| File | Chapter | Depth | One-line focus |
|---|---|---|---|
| `ch06-replication.md` | 6 — Replication | Read tradeoffs / skim mechanics | sync vs async, lag anomalies, failover risks |
| `ch07-partitioning.md` | 7 — Partitioning | Read tradeoffs / skim index detail | range vs hash, hot keys, rebalancing |
| `ch08-transactions.md` | 8 — Transactions | **Read fully** | isolation ladder, write skew vs lost update |
| `ch09-distributed-systems-trouble.md` | 9 — Trouble w/ Distributed Systems | **Read fully — highest yield** | timeouts, clocks, pauses, fencing tokens |
| `ch10-consistency-consensus.md` | 10 — Consistency & Consensus | Read concepts / skim protocol | linearizability, CAP, consensus, etcd/ZK |

## The arc across chapters

The five chapters tell one connected story, worth holding as a single narrative in an interview:

1. **Replication (6)** puts copies of data on many nodes → introduces the sync/async + lag tradeoffs, and needs a leader.
2. **Partitioning (7)** splits data across nodes to scale → needs routing metadata stored somewhere reliable.
3. **Transactions (8)** give correctness guarantees on top → the isolation ladder.
4. **Trouble (9)** explains why all of the above is hard: you can't tell dead from slow, clocks lie, processes pause.
5. **Consensus (10)** is the answer to (9): agreement-despite-failure, provided by etcd/ZooKeeper — which is what elects the leader in (1) and holds the routing metadata in (2).

**The loop closes:** failover needs a leader → electing one needs consensus → consensus lives in etcd/ZK → which also stores partition routing.

## Highest-leverage for SRE loops

If time is short, **Ch 9 and Ch 8** are the two that most directly reward you. Ch 9 because it's your on-call reality in precise vocabulary; Ch 8 because isolation anomalies are a common data-corruption interview probe.
