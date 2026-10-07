# Scorecard template (copy this file to drills/scorecard.md and keep appending)

Use one section per type. Fill a row immediately after each problem or mock, while you still remember it.

## 1. Coding problem (every problem, timed or not)

Protocol: clarify → examples and edge cases → approach → complexity WITH the reason, stated BEFORE coding → narrate while coding → test by hand. Python, plain doc, no IDE, no autocomplete. Replace hedging with "let me think for a moment."

| Date | Problem | Type (timed / learning / mock) | Time (min) | Hints needed | Complexity justified (Y/N) | Silences >20 s | Bugs found by me | Bugs found by interviewer | Redo? (🔁) |
|---|---|---|---|---|---|---|---|---|---|
| | | | | | | | | | |

Counting rules
- Time: from "read the problem" to "tested by hand". Over 35 minutes counts as a miss.
- Hint: any moment you looked at the approach, a tag, or an interviewer nudge.
- Complexity justified: Y only if you said the reason ("O(n) because each element enters the window once"), before coding.
- Silence: any gap over 20 seconds without narration or a clean "let me think for a moment."

## 2. Human or self mock (one block per mock)

- Date / mock number (of 10) / type (coding + behavioral, NALSD, troubleshooting, full loop) / free or PAID / interviewer
- Verdict on the scale: Strong No Hire / No Hire / Lean Hire / Hire / Strong Hire
- Top 2 fixes the interviewer gave:
  1.
  2.
- Rubric (score each 1–4, where 3 = hire)

| Area | Score | Evidence (one line) |
|---|---|---|
| Communication and narration | | |
| Problem solving / approach | | |
| Coding or design quality | | |
| Testing / failure modes | | |
| Complexity or capacity math | | |
| Fundamentals depth (Linux, networking, storage) | | |
| Behavioral: Googleyness (user-first, humility) | | |
| Behavioral: leadership / influence without authority | | |
| Behavioral: ambiguity, failure, conflict | | |

Use only the rows that apply to the mock type.

## 3. NALSD design (every timed design)

| Date | Prompt | Requirements first (Y/N) | Capacity chain with units (Y/N) | Bottlenecks named | Failure modes (count) | Monitoring / SLO plan (Y/N) | Finished in 45 min (Y/N) | Hedges / silences |
|---|---|---|---|---|---|---|---|---|
| | | | | | | | | |

## 4. Gate table (update after every mock and every gate Sunday)

| Gate | Criterion | Evidence | Met? |
|---|---|---|---|
| A | 8 of 10 timed unseen mediums in ≤35 min, narrated, complexity justified | Week 5 (Wed, Sun) + Week 6 (Mon, Wed, Sun) rows | |
| A | 3 consecutive human mocks at Hire or better | Mocks in Weeks 4, 5, 6 | |
| B | 2 live NALSD mocks in 45 min with full capacity math | Weeks 8, 9 | |
| B | 2 troubleshooting mocks where someone else broke the system | Weeks 10, 11 | |
| B | "Why" 3 levels deep on every résumé line and project component | Week 12 audit | |
| C | 2 full mock loops at Hire | Weeks 14, 16 | |
