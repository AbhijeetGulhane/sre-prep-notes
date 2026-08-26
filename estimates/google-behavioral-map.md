# Google Behavioral Map — Six STAR Stories → Google's Rubric

> Companion to `star-stories.md`. Maps the six locked stories against Google's actual scored attributes, gives a question-stem → story lookup, and names the coverage gaps specific to Google. Google-only — HRT/other targets mapped separately.

---

## Google's scored attributes (current framework)

Google scores four named attributes in every loop:

1. **GCA — General Cognitive Ability.** Structured problem-solving; making forward progress with incomplete information; how you reason, not just what you concluded.
2. **RRK — Role-Related Knowledge.** Technical/domain depth for the role. For SysEng-SRE: Linux internals, debugging, systems, reliability.
3. **Leadership — "emergent leadership."** Stepping up to lead *when your skills are needed*, without relying on formal authority. Influencing through data, persuasion, and collaboration — not title.
4. **Googleyness.** A behavioral rubric (reframed Dec 2024), scoring four pillars:
   - **Comfort with ambiguity** — make progress when the problem/goal/path is undefined, instead of stalling for instructions.
   - **Receive and act on feedback** — intellectual humility; take critique (from people or from failure) and change.
   - **Challenge the status quo** — fix broken systems / push improvement when evidence supports it; bias to action.
   - **User-first** — default to the user over internal convenience.

**Delivery rules Google specifically rewards:**
- **STAR-L** — add a *Learnings* close. Interviewers probe the **Action** hardest; every claim in it needs specifics.
- **"I" for actions, "we" for outcomes.** Vague ownership is a top reason technically strong candidates score poorly.
- Prepare **6–8 deep, reusable stories**, each mapped to **more than one** pillar, so you can adapt to the exact stem asked. (This doc is that mapping.)

---

## Story → attribute coverage

| # | Story | Primary attribute | Also hits | SRE theme |
|---|-------|-------------------|-----------|-----------|
| 1 | **Ownership** — patching automation (60K estate) | Googleyness: *challenge status quo / bias to action* — nobody owned the toil; taken on beyond BAU | Leadership (emergent); GCA (scoping the problem) | **Toil** |
| 2 | **Technical Excellence** — FD exhaustion | **RRK** — deep systems debugging | GCA (distrusted the symptom → reasoned to root cause) | **On-call / incident** |
| 3 | **Conflict** — AD/Hadoop migration | **Leadership** — influence without authority; moved a resistant team with evidence, not power | Googleyness: *receive/act on feedback* (treated their fear as legitimate, adapted the plan); GCA (SQL diagnosis) | — |
| 4 | **Failure** — Gap network misconfig | Googleyness: *receive/act on feedback* + intellectual humility | Googleyness: *challenge status quo* (built the verification tool); strong Learnings beat | — |
| 5 | **Cross-team** — DCR events | **Leadership** — emergent, on a live bridge | RRK (cross-boundary fault isolation); Googleyness collaboration | **On-call / incident** |
| 6 | **Ambiguity** — holiday data bug | Googleyness: *comfort with ambiguity* + **GCA** (top-down decomposition) | Googleyness: *user-first* (traders' data correctness) | — |

**Attribute totals across the set:**
- **Googleyness** — well covered on 3 of 4 pillars (ambiguity ✅ Story 6; feedback ✅ Stories 3, 4; challenge-status-quo ✅ Stories 1, 4). **User-first = thin** (only Story 6).
- **Leadership (emergent)** — strong: Stories 1, 3, 5.
- **GCA** — strong: Stories 2, 6 (plus reasoning in 1, 3).
- **RRK** — Stories 2, 5 carry the technical depth.

---

## Question-stem → which story to pull

| Question stem (paraphrased) | Pull |
|---|---|
| "Led a project start-to-finish / took ownership without being told" | **Story 1** |
| "Influenced others / aligned stakeholders **without formal authority**" (Google favorite) | **Story 3** |
| "Led a cross-functional team / coordinated under pressure" | **Story 5** |
| "A time you failed / a mistake / feedback you acted on" | **Story 4** |
| "Hardest thing you debugged / a tricky technical problem" (never say "hardest") | **Story 2** |
| "Made progress when requirements/problem were **unclear**" (most-probed pillar) | **Story 6** |
| "Challenged the status quo / fixed something broken" | **Story 1** or **4** |
| "Took a calculated risk / drove change / process improvement" | **Story 1** (automation) or **4** (verification tooling) |
| "Disagreed with someone / handled pushback" | **Story 3** |
| "Handled a high-pressure incident / on-call situation" | **Story 5** or **2** |

---

## Coverage gaps vs. the Google rubric

**1. User-first is the thinnest pillar (fix in delivery, not new stories).**
Only Story 6 leads with user impact. Make the latent user beat explicit in the others — one sentence of *who was affected and why it mattered to them*:
- **Story 2 (FD):** a live production platform was denying all service — real users/consumers blocked.
- **Story 5 (DCR):** the DR work protects **business continuity for trading** — the user is the desk/business that must keep operating through a failover.
- **Story 6 (holiday bug):** traders couldn't trust their cumulative P&L — the user impact *is* the story; state it up front.
Cheap, high-return: adds a Googleyness pillar with no new material.

**2. "Receiving feedback from a person and acting on it" — partial gap.**
Story 4 covers learning from a *failure*, but Google's feedback pillar also probes taking *critical feedback from a human* (manager/peer) and changing. No dedicated story exists.
- Closest stand-in: **Story 3** (adapted the plan after the Hadoop team's pushback = acting on stakeholder feedback).
- **To-do:** hold one real "someone told me my approach was wrong and I adjusted" example in reserve, even unpolished.

**3. SLOs / error budgets — no behavioral story (RRK/design gap, not G&L).**
None of the six touch reliability trade-offs / "how reliable is reliable enough." This is design territory, so not a G&L blocker — but if a behavioral question is framed around reliability trade-offs, there's no story to pull.
- **Fix in progress:** the **SLO Error Budget Service** project (starting September) is meant to fill exactly this. Build it before the interview window so it's a usable story, not a plan.

---

## Pre-loop to-do (Google-specific)

- [x] Add an explicit **Learnings** line to Stories **2, 3, 5** (1, 4, 6 already have one). ✅ done
- [x] Add a one-sentence **user-impact** beat to Stories **2, 5** (Story 6 already leads with it). ✅ done — note Story 2 is internal/not customer-facing, framed honestly as "every consumer of the service blocked."
- [ ] Prepare a reserve **"acted on critical feedback from a person"** story (or rehearse Story 3 as the stand-in). ← needs a real memory; not yet built
- [ ] Build the **SLO Error Budget Service** to cover the reliability-trade-off gap.
- [ ] Rehearse each Action out loud with specifics — Google probes Action hardest; "I" for actions, "we" for outcomes.
