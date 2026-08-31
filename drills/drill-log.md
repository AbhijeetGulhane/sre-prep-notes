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

**Verdict: NO HIRE (as delivered).** Algorithm was L5 — Kahn's chosen cleanly, edge direction `b→a` reasoned correctly out loud, cycle-detection-by-count (`len(result)==n`) identified unprompted. But **4 rounds to runnable**, all four breaks were identifier/name defects, and a NEW one (`i2`) was introduced *while fixing others* after two explicit warnings. Contrast: Round 1 same day was clean-on-delivery, no typos → HIRE. Same brain, opposite verdicts. The gap IS the identifier read-pass, and it did not hold once code got longer (2 structures + build loop).

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
