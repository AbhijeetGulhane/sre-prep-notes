# Coding Prep — Progress Report

*Aug 27–31 2026 · Diagnostic Day 1 through Interview-Sim Set*
*Companion to `drills/drill-log.md` (per-session patterns). This is the week-level synthesis. Re-read before the Sept 3 re-test and grade that re-test against the two open habits below.*

---

## Headline

**The algorithm ceiling is not the problem and never has been. Every point lost across five days has been delivery.** That is the entire report in one sentence — and it is the most fixable diagnosis possible: the expensive thing (algorithmic judgment) is already at L5; the cheap, mechanical things (read-pass, out-loud verification) are what drag the grades down.

---

## The arc, by session

| Session | Problems | Verdicts (6-pt scale) | What it showed |
|---|---|---|---|
| Day 1 (Aug 27) | 3 mediums | ~No Hire ×3 | Correct algorithms, none runnable as delivered. 3 typos broke all 3. |
| Day 2 R1 | 1 medium | **Hire** | Clean on delivery, no typos. Read-pass held. |
| Day 2 R2 (Kahn's) | 1 medium | **Leaning No Hire** | Correct algo, 4 rounds to runnable. Read-pass lapsed on longer code. |
| Sliding-Window set | 2 mediums | **Hire, Hire** | Both clean, read-pass held unprompted. Warm pattern. |
| Interview-Sim set | 2 hard-mediums | **Hire (border Strong), Leaning Hire** | Strong-Hire *approaches*; delivery dragged both down. Read-pass lapsed again on the cold/hard one. |

**Trajectory:** 0/3 clean → 1 clean → lapse → 2 clean → 1 clean of 2 (harder band). Genuinely upward, with a clear conditional: **cleanliness holds on warm/easy, breaks on cold/hard.**

---

## What's good — real strengths, not encouragement

**Algorithmic thinking is consistently L5.** Proven, not asserted:
- Diagnosed that naive sliding window fails on unsorted timestamps and reached for sorted-insert + bisect *unprompted* (Interview-Sim P1).
- Reverse-graph insight on the dependency problem — failure flows against edge direction — got it in the restate (Day 1).
- Binary-search-on-the-answer for min-capacity, correct bounds `[max, sum]`, cleanly reasoned aloud (Interview-Sim P2).
- Killed sliding window on the monotonicity argument, then *transferred that same check* to the next problem in his own words (Day 1 → Day 2).

**The technique-check reflex is now working.** Day 1 he pattern-matched labels before validating them. By the Sliding-Window set he was stating "this works *because* values are non-negative → sum is monotonic" as part of the plan. Fixed gap.

**When the read-pass runs, it works.** Every session where it ran audibly/deliberately, the code was clean. The tool works; the problem is consistency of *using* it.

---

## What's bad — named explicitly, ranked by cost

### 1. The read-pass lapses under load. #1 issue, now demonstrated not theorized.

Identifier-defect tally went 0 → 10 across the week:

| # | Defect | Session |
|---|---|---|
| 1 | `prefx` | Day 1 |
| 2 | `chat` for `char` | Day 1 |
| 3 | `dequq` for `deque` (survived explicit correction) | Day 1 |
| 4 | `adj[node]` undefined | Day 2 R2 |
| 5 | `indegree[n]` loop-var | Day 2 R2 |
| 6 | `.append[s1]` bracket-on-method | Day 2 R2 |
| 7 | `append(i2)` (introduced *during a fix*) | Day 2 R2 |
| 8 | `for r` vs `right` mismatch | skeleton drill |
| 9 | `.append[follow]` bracket (repeat of #6, same algo) | Kahn's cold rep |
| 10 | `curr_um` for `curr_sum` | Interview-Sim P2 |

Pattern in the data: defects cluster on **cold, longer, harder** problems (Day 1's unfamiliar set, Kahn's, the binary-search feasibility check) and **vanish on warm, drilled** ones (the two sliding-window mediums). Conclusion: **the pass is still effortful, not reflex — and effort is the first thing dropped when the algorithm itself is consuming attention.** Two clean sliding-window problems were *holding under favorable conditions*, not proof the habit is built.

### 2. No out-loud verification before "done." Both Interview-Sim problems; cost grades directly.

P2's two bugs (`curr_um`, `nodes > cap`) both die if one example is traced by hand. P1 was correct but left the interviewer to prove it. Highest-leverage unfixed habit: **one out-loud trace of the adversarial example catches identifier bugs, logic bugs, AND demonstrates correctness** — it would have lifted both of today's grades.

### 3. Narration drops when heads-down.

In silent interview mode (P1), produced a correct, non-trivial solution with *zero spoken reasoning* and never stated complexity. Correct-but-silent is Hire; the missing narration is exactly the Hire→Strong-Hire gap.

---

## Fixed vs still-open

**Fixed / trending fixed:**
- Technique-before-constraint-check → now checks first. ✅
- Welding the "why" onto complexity claims → mostly there, occasional pull needed. 🟡→✅
- Algorithm selection under pressure → consistently strong. ✅

**Still open:**
- Read-pass as a *reflex under load* (not just when warm). ❌ — the core issue.
- Out-loud verification before "done." ❌ — highest leverage, cheapest fix.
- Continuous narration + volunteered complexity in silent mode. ❌ — the Strong-Hire gap.

---

## Standing against the promotion gate

Gate: **3 clean on a re-test → compress to the 2-week track.** On current evidence: **not there yet** — not from ability, but because clean delivery isn't reliable on cold/hard problems, which is what a real loop throws.

Realistic band right now: **Hire on warm/medium; Leaning Hire-to-Hire on cold/hard; risk of Leaning No Hire if a typo survives to "done" on a hard one** (happened as recently as Aug 31, defect #10).

Distance to consistent Hire (then Strong Hire) is **two habits, no new knowledge:**
1. **Trace the adversarial example out loud before "done."** Fixes the most, costs the least.
2. **Run the read-pass audibly, specifically on cold/hard problems** — the exact condition where it currently lapses.

---

## One-paragraph verdict

An L5-caliber problem-solver with junior-level delivery discipline, and the gap between those two is the entire story of the prep. One major gap already closed this week (technique-checking), and a hard algorithm (Kahn's) went from four-rounds-to-runnable to a clean cold rep. Remaining work is not more LeetCode — it's making two mechanical habits reflexive under pressure: **verify out loud, and run the read-pass when the problem is hard, not just when it's easy.** Do that, and the delivered verdict converges with the algorithm verdict — Hire becomes the floor and Strong Hire comes into range. On the numbers that convergence is close: one hand-trace away from two higher grades on Aug 31.

---

## For the Sept 3 re-test

Grade the re-test against this file specifically:
- Does the **tally stay flat** through a cold/hard problem? (Defects vanishing only on warm patterns = habit not built.)
- Is the **adversarial example traced out loud before "done,"** unprompted?
- Is **complexity volunteered** at approach time, and narration continuous?

Flat tally + unprompted trace on a cold/hard problem = the promotion signal. This report is the baseline to measure against.
