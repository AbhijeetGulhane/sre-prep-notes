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
