# Googleyness Grid — STAR Stories × Behavioral Dimensions

Maps STAR stories against Google's core behavioral dimensions. ✅ = strong hit, ◐ = partial/can be angled, blank = doesn't cover.

**Status:** rows 1–6 still mapped from *headlines* — confirm each ✅ holds when you re-read the written version. Row 7 is now mapped from the full written story. Row 8 is in progress.

## The grid

| Story | Ambiguity | Conflict Resolution | Ownership | Bias to Action | Humility / Learn | Collaboration | Lifting Others | User-First |
|---|---|---|---|---|---|---|---|---|
| 1. FD-exhaustion outage | ✅ | | ◐ | ✅ | | | | ◐ |
| 2. Patching automation (60k) | ◐ | | ✅ | ✅ | | ◐ | | |
| 3. AD migration (4k) | | ◐ | ✅ | | | ✅ | | |
| 4. Honest failure | ◐ | | ◐ | | ✅ | | | |
| 5. Technical disagreement | | ✅ | | | ◐ | ◐ | | |
| 6. Capsule project | ✅ | | ✅ | ◐ | | | | |
| 7. Mentorship / grad rotation | | | ✅ | ◐ | ◐ | ◐ | ✅ | |
| 8. Network misattribution *(in progress)* | | ◐ | ◐ | | ✅ | ◐ | | |

## Row 7 — corrections after reading the full story

Downgrades and upgrades against the headline mapping:

- **Ownership ◐ → ✅.** Not just assigned a mentee. Proposed to the manager that the grad be taken into real patching work beyond the program's stated bar, and owned the outcome.
- **Collaboration ✅ → ◐.** The headline implied collaboration; the content is mentorship. Working *with* peers is not what this story shows. **This downgrade matters — see verdict.**
- **Humility blank → ◐.** The Complication section is a genuine self-critical beat: taught persistence, failed to teach the escalation boundary, cost the application teams their test window.
- **Bias to action blank → ◐.** Wrote the Confluence pages as the gaps surfaced rather than waiting for a documentation initiative.
- **Lifting Others ✅ confirmed.** Zero Linux to handling post-patch service issues across 200–300 servers/day in eight months. Docs still in use two years on.

## Coverage read

**Well covered:**
- **Ownership** — 4 strong (2, 3, 6, 7). Strongest dimension.
- **Humility / learning** — 2 strong (4, 8) plus partials. Now adequate; was thin.
- **Ambiguity** — 2 strong (1, 6).
- **Bias to action** — 2 strong (1, 2).

**Thin:**
- **Conflict resolution** — still only ONE strong hit (#5), with partials at #3 and #8. Unchanged gap.
- **Collaboration** — **now down to one strong hit (#3)** after the row 7 correction. This got worse, not better.
- **Lifting Others** — one hit (#7). Fine; it's a real one and this dimension rarely gets probed twice.
- **User-First** — **uncovered.** No story leads with the person on the other end. New column, deliberately left near-empty rather than blank-by-omission.

## Verdict + the fix

**Two gaps, one action each.**

**1. Collaboration and conflict are both single-threaded, and #3 is the only story serving both.** If the interviewer probes either twice, or probes both, story #3 carries too much. Re-angle #3 (AD migration, 4k servers) to surface its conflict thread explicitly — pushback from an app team, a downtime standoff, a priority clash — so it reads as a conflict story, and find collaboration elsewhere. #2 (patching automation) is the likely candidate: a 60k-node rollout needed other teams to adopt it, and adoption is collaboration.

**2. User-First is genuinely uncovered.** Production support is the obvious source — a time you went past the ticket for the trader or user on the other end, or pushed a fix because of who was affected rather than what the severity said. One story, doesn't need to be dramatic.

## Watch: self-critical story overlap

Three stories now carry a self-critical beat — #4 (honest failure), #7 (taught the wrong thing), #8 (misattributed to another team). That's fine *only if the flaw types differ*. #7 and #8 are cleanly different: a development misjudgment vs. a technical misdiagnosis. **Check #4** — if its flaw is also "misjudged what someone needed," you have one blind spot told three ways, and an interviewer hearing two of them in a 45-minute round will notice. Re-read #4 and confirm.

## Do not

Add more new stories. Eight is already more than a 45-minute round can hold. Re-angle what exists.
