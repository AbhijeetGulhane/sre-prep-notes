# DDIA Ch 8 — Transactions

> 2nd edition numbering. **Read this one properly** — isolation levels come up in interviews. Skim only the SSI implementation mechanics.
> Interview level: *"What isolation level does this need, and what anomaly are you preventing?"*

---

## What a transaction actually gives you

A transaction groups reads/writes so they succeed or fail as a unit. The point is to **collapse a huge space of partial-failure and concurrency edge cases into two outcomes**: commit or abort. Without it, you hand-handle every crash-mid-operation and every race.

**ACID** — know what each letter really means:
- **Atomicity** — all-or-nothing on *abort* (better word: *abortability*). Not about concurrency.
- **Consistency** — app-level invariants hold. This one is the app's job, not really the database's — DDIA is blunt that C is the odd one out.
- **Isolation** — concurrent transactions don't step on each other. **This is the meat of the chapter.**
- **Durability** — committed data survives crashes.

---

## Isolation levels — the core of the chapter

Isolation is a **ladder**: each level prevents more anomalies but costs more performance. Learn it as *anomaly prevented → level required*.

### The anomalies (know these by name)

| Anomaly | What happens |
|---|---|
| **Dirty read** | You read another transaction's uncommitted write |
| **Dirty write** | You overwrite another transaction's uncommitted write |
| **Read skew (non-repeatable read)** | Same row read twice in one txn gives different values |
| **Lost update** | Two txns read-modify-write the same value; one is lost |
| **Write skew** | Two txns read overlapping data, write disjoint rows, together break an invariant |
| **Phantom** | A write in one txn changes the result set of a search in another |

### The levels (what each prevents)

**Read Committed** — prevents dirty reads + dirty writes. The common default (Postgres default). Does *not* prevent read skew.

**Snapshot Isolation (Repeatable Read)** — each txn reads from a consistent snapshot taken at start. Prevents read skew. Implemented via **MVCC** (multi-version concurrency control — keeps old versions around). Postgres, MySQL, Oracle. Does *not* prevent write skew / phantoms.

**Serializable** — the strongest. Behaves as if transactions ran one at a time. Prevents *everything* including write skew. Three ways to implement:
- **Actual serial execution** — literally one at a time (VoltDB, Redis). Only works if txns are fast and fit in memory.
- **Two-phase locking (2PL)** — locks on read and write; strong but slow, deadlock-prone.
- **Serializable Snapshot Isolation (SSI)** — optimistic; detect conflicts at commit, abort losers. Postgres's serializable mode. **Skim the mechanics** — know it's optimistic and named SSI.

---

## Lost update vs write skew — the pair interviewers love

**Lost update:** two people increment the same counter; one increment vanishes. Fixes: atomic operations (`UPDATE ... SET x = x + 1`), explicit locks (`SELECT FOR UPDATE`), or compare-and-set.

**Write skew:** the subtle one. Two doctors both on-call, each checks "is at least one other doctor on-call?" (yes), each books off — now zero doctors on-call. Both reads were valid; the *combination* of writes broke the invariant. Only **serializable** isolation prevents it. This is the canonical write-skew example — memorise it.

**Interview trap:** snapshot isolation does *not* prevent write skew or lost updates by itself. If someone reaches for "repeatable read" to solve the on-call doctor problem, that's the tell they don't know the ladder.

---

## Why this matters for SRE specifically

You're rarely *choosing* an isolation level in an SRE loop, but you'll be asked to **reason about a data-corruption incident**: "the balance went negative even though we checked it first." That's write skew or a lost update, and naming it precisely — plus the fix (atomic op / serializable / explicit lock) — is exactly the depth signal the interview wants.

---

## Interview-ready summary

- ACID: atomicity = abortability, isolation is the real content, consistency is the app's job.
- Ladder: Read Committed → Snapshot Isolation (MVCC) → Serializable.
- Anomalies by name: dirty read/write, read skew, lost update, write skew, phantom.
- Snapshot isolation does NOT stop lost updates or write skew — only serializable does.
- On-call doctors = the write-skew example. Counter increment = the lost-update example.

---

## Reading resources

- **DDIA 2nd ed, Ch 8** — primary, read fully except SSI internals. (paid)
- **"A Critique of ANSI SQL Isolation Levels"** (Berenson et al.) — free PDF — the foundational paper; skim, but it's where the anomaly definitions come from. (free, supplementary)
- **PostgreSQL docs — Transaction Isolation** (postgresql.org/docs) — free — concrete, precise, maps directly to the ladder above. Best free reinforcement. (free)
- **Martin Kleppmann's talk "Transactions: myths, surprises and opportunities"** (YouTube) — free — the author explaining the chapter; excellent. (free)
