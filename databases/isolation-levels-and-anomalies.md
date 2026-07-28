# Transaction Isolation Anomalies — Worked Examples

*Supporting reference for [03.1-distributed-kv-store-nalsd.md](../system-design/03.1-distributed-kv-store-nalsd.md) (the AP/consistency trade-off) and [09-video-platform.md](../system-design/09-video-platform.md) (Spanner's external consistency). This is database transaction theory, not NALSD system design — no capacity math here, just the concurrency-control foundation several designs in this series built on top of.*

**The honest caveat up front, worth internalizing before anything else**: the ANSI SQL standard's isolation-level definitions are weaker and more ambiguous than most people assume, and real databases diverge from them under the same names. Postgres's "Repeatable Read" is actually Snapshot Isolation, not the lock-based mechanism the ANSI standard describes. This is a well-documented point of confusion (DDIA Ch. 7 covers it directly) — every answer below distinguishes "what the standard technically requires" from "what real databases actually do."

---

## 1. Dirty Read

**What goes wrong**: a transaction reads data written by another transaction that hasn't committed yet — and might never commit.

```
T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- not yet committed
T2: SELECT balance FROM accounts WHERE id = 1;                   -- reads the uncommitted -100
T1: ROLLBACK;                                                     -- the -100 never actually happened
```
T2 made a decision based on a value that, in the end, was never real.

**Prevented by**: **Read Committed and above.** Only **Read Uncommitted** — the weakest standard isolation level — permits this. In practice, almost no production database defaults to Read Uncommitted; this anomaly is largely a textbook baseline rather than a real operational risk on modern systems.

---

## 2. Phantom Read

**What goes wrong**: a transaction re-runs the same range/predicate query twice and gets a **different set of rows** the second time, because another transaction inserted or deleted rows matching that predicate in between — new rows "phantom" into existence within the same logical transaction.

```
T1: SELECT * FROM orders WHERE amount > 1000;   -- returns 5 rows
T2: INSERT INTO orders (amount) VALUES (2000); COMMIT;
T1: SELECT * FROM orders WHERE amount > 1000;   -- now returns 6 rows
```

**Prevented by**: **Serializable**, per the ANSI standard, is the only level that formally guarantees no phantoms. **Repeatable Read**, in the standard's own technical definition, does **not** guarantee phantom prevention — it only guarantees that rows already read won't change value, not that new matching rows can't appear.

**Where real-world behavior diverges**: MySQL's InnoDB "Repeatable Read" uses next-key locking, which *does* prevent phantoms in practice — a real strengthening beyond what the standard requires under that name. Postgres's "Repeatable Read" is Snapshot Isolation, which also tends to prevent phantoms as a side effect of reading from a single consistent snapshot, though for a completely different mechanism (MVCC snapshot, not locking) than InnoDB's approach. **The practical lesson**: never assume "Repeatable Read" behaves identically across databases — check the specific engine's actual guarantee, not the standard's name.

---

## 3. Lost Update

**What goes wrong**: two transactions concurrently read the same value, each computes a new value based on that read, and one write silently clobbers the other — the second writer never saw the first writer's update at all.

```
T1: reads balance = 100
T2: reads balance = 100        (concurrently, before T1 writes)
T1: writes balance = 100 + 50 = 150; commits
T2: writes balance = 100 + 30 = 130; commits

Final balance: 130
Correct balance: 100 + 50 + 30 = 180
```
T1's +50 update is completely lost — not merged, not conflicted, just silently gone.

**Prevented by**: **not** Read Committed — this is Read Committed's classic, well-known weakness, since each individual read and write is committed cleanly, but the read-then-write sequence as a whole still races. **Snapshot Isolation prevents this only if the database adds explicit write-write conflict detection** ("first committer wins," aborting the second transaction rather than letting it silently overwrite) — this is a real, common point of confusion worth stating precisely: Snapshot Isolation as a pure theoretical concept does *not* automatically prevent lost updates; most real production implementations (Postgres, Oracle) add this detection specifically to close the gap, but it's an addition to SI, not an inherent property of it. **Serializable** prevents it unconditionally.

**Practical mitigations that sidestep the problem entirely**, worth knowing independent of isolation level:
- **Atomic compound operations**: `UPDATE accounts SET balance = balance + 50 WHERE id = 1` — the read-modify-write happens as one atomic database operation, never exposing a read-then-write race window at all. Directly the same idea as doc 02's Redis `INCR` — one atomic command beats a check-then-act pair regardless of which system you're in.
- **Explicit locking reads**: `SELECT ... FOR UPDATE`, forcing T2 to wait for T1's transaction to complete before it can even read the value.
- **Application-level compare-and-swap** via a version column, rejecting a write if the row's version doesn't match what was read.

---

## 4. Read Skew

**What goes wrong**: a transaction reads two (or more) values that are supposed to be consistent with each other, but reads them at different points in time relative to a concurrent transaction's update — ending up with a combination of values that never actually existed as a valid state at any single moment.

```
Before: account A = 500, account B = 500   (total = 1000, always true when consistent)

T1: reads account A's balance = 500

T2 (concurrent): transfers 100 from A to B
                 A: 500 → 400, B: 500 → 600, commits

T1: reads account B's balance = 600

T1's view: A=500 (stale, pre-transfer) + B=600 (fresh, post-transfer) = 1100
```
The true total is always 1000 — either the before-transfer state (500+500) or the after-transfer state (400+600) — but T1 saw a combination that mixes both, a state that was never real.

**Prevented by**: **Snapshot Isolation** — a transaction sees one consistent point-in-time view for its *entire* duration, so it would see either both pre-transfer values or both post-transfer values, never a mix. In databases where "Repeatable Read" is actually implemented as Snapshot Isolation (Postgres), that name prevents this too — but that's again a real-world implementation detail, not a guarantee of the ANSI standard's Repeatable Read definition itself. **Serializable** prevents it as well, being at least as strong. **Not prevented by Read Committed** — Read Committed only guarantees each individual statement sees committed data at the moment it runs, not that the whole multi-statement transaction sees one consistent snapshot throughout.

---

## 5. Summary Matrix

| Isolation level (ANSI standard definition) | Dirty read | Phantom read | Lost update | Read skew |
|---|---|---|---|---|
| Read Uncommitted | ✗ possible | ✗ possible | ✗ possible | ✗ possible |
| Read Committed | ✓ prevented | ✗ possible | ✗ possible | ✗ possible |
| Repeatable Read (standard/locking definition) | ✓ prevented | ✗ possible (standard permits it) | ✓ prevented (locking holds reads) | ✓ prevented (same rows won't change) |
| Snapshot Isolation (not in ANSI standard, common real implementation) | ✓ prevented | Implementation-dependent, often prevented in practice | ✓ only with added write-write conflict detection | ✓ prevented |
| Serializable | ✓ prevented | ✓ prevented | ✓ prevented | ✓ prevented |

**The one thing worth remembering above all the individual cells**: don't trust a database's isolation-level *name* without checking its actual documented behavior — "Repeatable Read" alone means meaningfully different things in MySQL, Postgres, and the ANSI standard's own text.

---

## 6. Bonus, adjacent concept worth knowing even though not asked: Write Skew

Distinct from lost update — **two transactions read overlapping data, each makes a decision based on what they read, and each writes to a *different* object, so no direct read-write or write-write conflict is ever detected**, yet the combined result violates an invariant neither transaction individually violated. DDIA's classic example: two doctors on call both check "is at least one other doctor on call?", both see yes, both remove themselves from on-call duty, and now zero doctors are on call — a real invariant violation that Snapshot Isolation's write-write conflict detection (Section 3) doesn't catch, because there's no overlapping *write*, only an overlapping *read* followed by disjoint writes. **Only true Serializable isolation reliably prevents write skew** — worth having this distinction ready specifically because it's the natural follow-up question after lost update, and the two are easy to conflate.

---

## 7. Further Reading

- **DDIA, Chapter 7 ("Transactions")** — **paid** (already in your library), **essential** — the direct source for this doc's treatment of the ANSI-standard-vs-real-world divergence, and the original source of the write-skew doctors example referenced above.

## 8. Follow-up Questions

- **"Which of these matters for doc 03.1's AP-style KV store?"** — none of them directly, in the traditional sense — that design doesn't offer transactional isolation across multiple keys at all; its conflict-resolution machinery (vector clocks, doc 03.2) solves a related but distinct problem: concurrent writes to the *same* key, not multi-statement transaction anomalies across *different* keys.
- **"Why does Spanner (doc 09) get to claim strong consistency without these trade-offs?"** — it doesn't skip the trade-off, it pays for it differently: TrueTime's bounded-uncertainty synchronized clocks plus Paxos-based replication is what lets Spanner offer serializable-equivalent guarantees at global scale, at real infrastructure cost (globally synchronized atomic/GPS clocks) rather than by avoiding the underlying hard problem.
- **"Is Serializable always the right choice if it prevents everything?"** — no — it's the most expensive isolation level in terms of throughput (typically requiring either full locking or optimistic conflict detection with real abort/retry rates under contention), so the right level is a genuine trade-off between correctness guarantees needed and throughput cost, not a default-to-strongest decision.
