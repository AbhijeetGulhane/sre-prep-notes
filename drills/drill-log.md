# Drill Log — Recurring Error Patterns

Running record of weak spots across rapid-fire recall sessions, so they get **reinforced deliberately** rather than rediscovered each day. Not a per-session transcript — only patterns worth drilling.

**How this gets updated:** after a drill session with Claude, any recurring miss (something wrong 2+ times, or a consistent imprecision) gets logged here. Claude reads this at the start of a session to weight questions toward known weak spots.

---

## Active weak spots (drill these)

_(none logged yet — will populate after the first few sessions)_

<!-- Format:
- **[Pattern]** short description — first seen <date>, seen <N>x. Drill: <what to do>.
  e.g. **[Arithmetic — div/mult]** Divides when should multiply on bandwidth Qs — seen 3x. Drill: force "per-unit × count = total" out loud.
  e.g. **[Technique — imprecise]** Says "versioning" for vector clocks — seen 2x. Drill: the concurrent-vs-causal distinction.
-->

## Resolved (kept for reference)

_(none yet)_

---

## Session tally

| Date | Technique ✅/partial/✗ | Fermi ✅/✗ | Flagged for next |
|---|---|---|---|
| _(first session pending)_ | | | |
---

## Coding Diagnostic Day 1 — Aug 27 2026 (3 mediums: hashmap / sliding-window / graph)

All three solved with correct, optimal algorithms. Every problem broke on execution, not thinking. Patterns:

**P0 — Identifier typos break runnability. [TOP PRIORITY — fired 3/3]**
- `prefx` (P1), `chat` (P2), `dequq` (P3). The `dequq` typo survived an *explicit* in-session correction.
- At an SRE loop, code that fails on an untested NameError reads as "doesn't verify their own work" — disproportionately damaging for this role.
- **Drill:** before declaring done, one slow pass reading ONLY identifiers against their declarations. Every var, every function name. This pass kills all three above.

**P1 — Technique-before-constraint-check.**
- P1: grabbed "sliding window" on surface features before testing against negatives (it doesn't hold — no monotonicity). Needed redirect.
- Improved visibly by P2 (did the monotonicity check partly unprompted, connected it back to P1). Keep reinforcing.
- **Drill:** the instant you say "this looks like [technique]," ask out loud "what constraint would break it?" — BEFORE building.

**P2 — Verify on the adversarial example, not the comfortable one.**
- P1: traced all-positive Example 1 instead of the negatives+seed Example 2 — i.e. tested where already confident, skipped the case the problem was actually about.
- **Drill:** pick the test case with the feature you're LEAST sure of (negatives, empty, the seed, the boundary).

**P3 — BFS assembly under time pressure.**
- P3: collapsed worklist + visited-set into one structure; traversal couldn't advance. Algorithm known abstractly; canonical two-structure shape not yet reflex.
- **Drill:** multi-source BFS skeleton to 30-sec cold — `q=deque(seeds); seen=set(seeds); while q: pop; for nbr: if unseen: mark+enqueue`.

**Ongoing — weld-the-why (existing prescription).**
- Still gave right answers as bare values (`{0:1}`, count-not-presence, max_freq-no-decrement), needed pulling for the "because." Trending better — heavier in P1 than P3.

---

## Round 2 — Coding Medium (Aug 30 2026, topological sort / Kahn's)

**Verdict: LEANING NO HIRE (as delivered).** [Corrected from "No Hire": on Google's 6-point scale the correct rung is Leaning No Hire — algorithm off the No-Hire floor, execution below the hire line.] Algorithm was L5 — Kahn's chosen cleanly, edge direction `b→a` reasoned correctly out loud, cycle-detection-by-count (`len(result)==n`) identified unprompted. But **4 rounds to runnable**, all four breaks were identifier/name defects, and a NEW one (`i2`) was introduced *while fixing others* after two explicit warnings. Contrast: Round 1 same day was clean-on-delivery, no typos → HIRE. Same brain, opposite verdicts. The gap IS the identifier read-pass, and it did not hold once code got longer (2 structures + build loop).

### 🔴 IDENTIFIER-DEFECT TALLY: 7
Every one would die in a 20-second read-pass. None got caught before "done."
1. `prefx` (Day 1, P1) — typo
2. `chat` for `char` (Day 1, P2) — typo
3. `dequq` for `deque` (Day 1, P3) — survived an explicit correction
4. `adj[node]` — `node` undefined (Day 2, R2) — wrong name
5. `indegree[n]`/`append(n)` — loop var, should be `i` (Day 2, R2)
6. `adj[s2].append[s1]` — brackets on a method call (Day 2, R2)
7. `q.append(i2)` — `i2` undefined, INTRODUCED during a fix (Day 2, R2)

**Rule: increment this every session. The count stopping = the habit took.**

### The habit, named
The defect is not "makes typos." It's: **declares code "done" without verifying every name resolves.** The missing checkpoint sits between "finished writing" and "claiming done." It held in Round 1 (short code, fresh) and collapsed in Round 2 (long code) → it's an effortful act, not yet a habit. A habit survives load; this didn't.

### The read-pass protocol (run EVERY time, before "done", ~30s)
1. **Last line written ≠ done.** "Done" is only sayable after the pass.
2. **Read BOTTOM-TO-TOP** — defeats forward-reading autocomplete (why `i2`/`node` survive a normal read).
3. **Name scan:** every variable — exists? right one?
4. **Declaration trace:** every used variable → glance up to its definition. No declaration above = bug.
5. **Call-vs-subscript:** every `.method` followed by `(`, not `[`.
6. **Loop-var match:** every `for X in…:` → body uses `X`.
7. **Boundary/operator glance:** `<` vs `<=`, `>=` vs `>`, `>3` vs `>=3`.

### Making it stick
- **Narrate the pass out loud** — "`i2`, not declared, fix" — forces it to actually run.
- **Drill it on LONG problems** (25–40 lines, 2 data structures + build loop) — that's where it fails and matters, not short easy functions.
- **Metric:** success = zero identifier defects survive to "done" (NOT zero typos while writing). Typo freely; catch all in the pass.

**Monitor tape:** *Last line written ≠ done. Bottom-to-top name scan, then say "done."*


---

## Sliding Window Set — 2 timed mediums (Aug 31 2026)

**Result: 2 of 2 CLEAN. Both Hire. Read-pass held BOTH times, unprompted, on real timed code. Tally stayed at 9 — zero new identifier defects.**

- **P1 — longest substring with ≤ k distinct** (dict + counts). Clean on delivery, O(n)/O(min(k,26)), k=0 handled, adversarial-verified (`"aabbcc",k=1`). One verbal correction: the "run of same chars" framing for why the shrink is a `while` was slightly off — real reason: a left-removal only drops distinct-count if that char's freq hits 0, so you loop until back within k. Code was right.
- **P2 — shortest window with sum ≥ limit** (running sum, no dict). Clean on delivery, O(n)/O(1), impossible-case → 0 handled, record-before-shrink order correct (the shortest-window trap), non-negative monotonicity argument crisp and unprompted.

**Ran the read-pass himself on both, before declaring done — confirmed when asked.** This is the Round-2 lapse NOT recurring.

### Week trend (the thing that matters)
- Day 1 (Aug 27): 0/3 clean, execution broke all three.
- Day 2 R1: clean (Hire). Day 2 R2 (Kahn's): lapsed, 4 rounds to runnable (Leaning No Hire) — read-pass failed on longer code.
- Aug 31: 2/2 clean (Hire, Hire). Read-pass holding as code length grows.
- **Variable that changed: not the algorithms (L5 throughout) — the read-pass now runs BEFORE "done" and holds. Tally frozen at 9 for a full session.**

### ⚠️ Caveat — do NOT read as "fixed"
Both clean problems were sliding-window, a pattern drilled cold the same day. The pass held under *favorable* conditions (warm pattern, low algorithmic load). The real test is a COLD pattern under time pressure — where working memory is spent on the algorithm and the pass is most likely to lapse (exactly what happened with Kahn's in R2). Conclusion: "holding under favorable conditions, untested under unfavorable." Next session = cold, un-pre-drilled pattern (graph/DP) to test whether the pass survives divided attention.

### Identifier-defect tally: 9 (unchanged this session)

---

## Interview-Sim Set — 2 hard-mediums, phone-screen mode (Aug 31 2026)

First session in full interview-simulation mode: interviewer silent, no mid-problem help, candidate drives, grade after. This mode is now the default for all coding.

**Result: P1 Hire (border Strong Hire), P2 Leaning Hire. Approach was Strong-Hire caliber on BOTH — delivery dragged them down.**

- **P1 — alert dedup, unsorted arrival** (dict-of-sorted-lists + bisect). CORRECT, including the hard part: diagnosed that naive sliding window fails on unsorted timestamps, reached for sorted-insert + binary search unprompted. Clean code, no identifier defect. **Held at Hire (not Strong) purely on delivery:** went fully silent while coding, never stated final complexity, never verified out loud. Strong-Hire *work*, Hire *grade*.
- **P2 — min node capacity, contiguous chunks** (binary-search-on-answer). Approach correct and well-narrated (bounds [max,sum], "binary search on capacity not values"), complexity O(n log S)/O(1) stated and correct. BUT code did not run — two bugs, neither caught:
  - `curr_um = r` (typo for `curr_sum`) → accumulator never resets → **identifier defect #10**
  - `if nodes > cap` should be `if nodes > num_nodes` → wrong-variable logic bug → feasibility check broken, wrong answers on the examples
  Both die if either example is traced by hand. Leaning Hire = right idea, right complexity, delivered artifact has uncaught correctness bugs.

### The finding (now demonstrated, not predicted)
**The read-pass holds when warm, lapses when the algorithm is cold/hard.** Two clean sliding-window problems earlier same day (warm, pre-drilled) → pass held. Two hard-mediums (cold, not drilled) → P2 lapsed, defect #10. This is the SAME failure mode as Kahn's in R2. The pass is still effortful, not reflex — and effort is the first thing dropped when the algorithm eats working memory. Warm-pattern cleanliness is NOT proof the habit is built.

### Recurring completeness gap
Said "done" without tracing an example on BOTH problems. Both problems had issues a single out-loud trace would surface (P2's two bugs; P1's correctness left unproven). **Highest-leverage single fix: before "done," trace the ADVERSARIAL example out loud.** It catches `curr_um` (sum visibly not resetting), catches `nodes > cap` (never fires on the trace), and demonstrates correctness instead of leaving the interviewer to verify. One habit fixes both grades.

### Identifier-defect tally: 10
10. `curr_um` for `curr_sum` — binary-search feasibility check; read-pass lapsed on cold/hard problem (Aug 31)

### Standing gaps to close for Strong Hire (from this session)
1. Narrate continuously while coding — no silent stretches (P1).
2. State final complexity, volunteered, at approach time (P1 missed it entirely).
3. Run read-pass AUDIBLY, especially on cold/hard problems where it lapses (P2, #10).
4. **Trace the adversarial example out loud before "done" — both problems, both sessions.**



---

## PROMOTION-GATE RE-TEST — Aug-27 diagnostic re-run (Sep 4 2026)

**Result: 3 of 3 STRICT CLEAN. Tally held flat at 10 across all three — including the graph problem. PROMOTION EARNED.**
(Note: timeline since collapsed to fast-track — interview Sep 15 AM, ~10 prep days. Promotion is moot as a track-selector but the re-test result stands as the readiness signal.)

Strict-clean bar met on all three: solved in 35min, correct, optimal, runnable as delivered, zero identifier defects.

| | Problem | Verdict | Tally |
|---|---|---|---|
| P1 | hashmap — longest consecutive sequence | Strict Clean / Hire | 10 |
| P2 | sliding-window — longest single-char run w/ k replaces | Strict Clean / Hire (near Strong) | 10 |
| P3 | **graph — cycle detect / Kahn's** | **Strict Clean / Hire** | **10** |

### Why this matters (vs Aug 27: same 3 categories, was 0/3 clean)
- **The read-pass HELD on the graph problem** — the exact spot it broke 4× this week (defects 4,5,6,7,9). The `.append()` construct that was defect #6 AND #9 came out with parens this time. Not luck — the habit held under the cold/hard condition where it used to fail.
- **Out-loud adversarial trace happened UNPROMPTED on P1 and P2** (the #1 open item from the progress report). On P3 he traced the happy path first, needed a nudge to trace the cycle case — then nailed it.
- **Self-correction working:** caught his own edge-direction confusion on P3 (`b:[a]` vs `a:[b]`) and resolved it before coding; correctly reasoned the `visited` set was redundant in Kahn's and didn't force it in.

### Tally FROZEN at 10 for the whole session
First full multi-problem session (incl. a graph) with zero new identifier defects. The habit is holding.

### Still shy of Strong Hire (the remaining ceiling, now the fast-track target)
Two of three landed Hire not Strong, same reason both times:
- **Complexity not stated out loud** (explained mechanism, didn't put the O() number on it) — P2, P3.
- **Adversarial case traced SECOND, not first** — P3 (happy path first, cycle case after a nudge). Strong Hire attacks the breaking case first.
These are the only coding items left worth pushing in the 10 days. Correctness + read-pass are holding — do NOT grind those.

### Identifier-defect tally: 10 (unchanged — held flat through a full 3-problem re-test)
