# Talking Points — AI's Role in SRE (Week 8–9)

**Purpose:** Rehearsal script for behavioral rounds, mock interviews, and the "how do you stay current" slot. Two talking points, each in a 30-second and a 90-second version.

**Rule: deploy one per loop, not both.** Two AI talking points in one conversation reads as a hobby horse. TP1 for Google and Meta PE. TP2 for fintech, and for anywhere the conversation turns to on-call.

**Blocker: TP2 does not work without a real number.** See §3.

---

## TP1 — ML-serving reliability is a different failure class

### 30-second version

> "The thing that changed how I think about reliability recently is that ML serving fails differently from a normal service. A normal service fails loudly — it 500s, it times out, it crashes, and the golden signals catch it. An ML service returns a fast, well-formed, HTTP-200 answer that's just *wrong*. Latency's fine, error rate's zero, saturation's fine. Every signal is green and the output is bad. So the entire monitoring model has to change — you end up treating data freshness and feature completeness as first-class SLIs, because model quality itself only shows up days later when the labels arrive."

*(~95 words, ~35 seconds spoken. Stop here. Let them pull.)*

### 90-second version — if they engage

Add, in this order:

**The concrete example.** "The one that made it click for me: a feature pipeline changes schema upstream, the lookup fails, and the serving code fills a default of zero instead of erroring. Nothing pages. The model just quietly gets worse for everyone. And the fix isn't a better model — it's a design decision. Fail closed. If a feature isn't available at serving time, return an error instead of imputing. That converts a silent multi-day quality failure into a visible error-rate spike, which your existing monitoring already catches."

**The rollout point.** "It changes deploys too. A model rollout is the same binary with different weights, so there's nothing to code-review — validation has to be empirical. Shadow first at full traffic with the outputs discarded, so you can compare the new model against the incumbent on identical inputs. Then a sticky-hash canary. And you find the canary size is bounded by statistics, not blast radius — detecting a half-percent regression might need more traffic than you want to expose."

**The honest close — do not skip this.** "I should be clear that I haven't operated ML serving in production. What I've operated is large-scale pipeline and platform reliability, and most of it transfers directly — freshness SLOs, data contracts, progressive rollout, blast-radius control. The genuinely new part is a correctness signal with a multi-day feedback loop and no error code attached to it."

### Delivery notes

- **The load-bearing sentence is "every signal is green and the output is wrong."** Everything else is support. If you only land one line, land that one.
- **Say "fail closed" out loud.** It's a design opinion, not a fact, and design opinions are what separate L5 signal from L4.
- **The disclaimer at the end is not weakness, it's calibration.** Interviewers weight self-assessment accuracy heavily. A candidate who volunteers the boundary of their experience gets trusted on everything inside it.
- Pace: this is a topic where you'll want to speed up. Don't. The silent-failure point needs a beat after it.

### Likely follow-ups

| They ask | You say |
|---|---|
| "How would you actually detect that?" | Three layers with different detection latencies. Infra: golden signals, seconds. Data: freshness, completeness, distribution stability as SLOs — minutes to hours. Model quality: delayed ground truth — days. Name freshness as the single highest-value monitor. |
| "What's drift?" | Distinguish it from skew immediately. Drift is the world changing — a condition you manage with burn-rate alerting on a model-quality SLO. Skew is your training and serving paths computing features inconsistently — that's a bug with a root cause, detected by shadow-replay parity checks. Conflating them means you retrain when you needed a diff. |
| "How do you know the model is still good, in real time?" | You don't. That's the honest answer. Labels are late or absent — fraud labels arrive with the chargeback, credit default is a 12-month horizon. So you build layered proxies: prediction distribution, confidence entropy, fallback rate, business metrics, plus a reserved human-labeled sample. And you accept days of detection latency at the quality layer while keeping seconds at the layers above. |
| "Have you built any of this?" | No — and say so cleanly, then pivot to the transferable half. Do not stretch. |

### Do not say

- Anything implying you've owned a model in production.
- "PSI" or "Jensen-Shannon" unprompted. If they raise it, engage. If you introduce it, you've moved from operator to name-dropper and invited a question you can't answer from experience.
- "AI is changing everything." Vague enthusiasm is the default answer; specificity is the differentiator.

---

## TP2 — AIOps and alert noise: the design-defect argument

### 30-second version

> "On-call at Barclays across a 60,000-node estate, the problem was never detection — it was that one infrastructure event fans out into hundreds of downstream alerts. So AIOps is interesting to me, but I've landed somewhere slightly against the grain on it: most alert noise is a design defect, not a data science problem. Symptom-based alerting and SLO burn-rate alerting delete far more noise than ML compresses. I'd want to do that hygiene first, and use correlation for the residual — the genuinely independent alerts across domains that share a root cause."

*(~95 words. Same rule — stop and let them pull.)*

### 90-second version — if they engage

**The credibility fact.** "It's worth knowing that Gartner coined the term in 2017 and then renamed the category to Event Intelligence Solutions in 2025 — which is a fair description of what the working subset actually is. Correlation, dedup, and alert enrichment are production-real. Anomaly detection is real but noisy and expensive to tune. Autonomous remediation is mostly still aspirational outside a narrow catalog of low-blast-radius actions."

**The method — this is the part that scores.** "If someone handed me a system generating ten thousand alerts a day, I wouldn't start with ML. I'd start with an audit: ninety days of alerts joined against actions actually taken, ranked by volume. The distribution is always brutal — a handful of rules produce most of the volume and almost none of the actions. Then in order: delete the non-actionable ones, convert cause-based alerts to symptom-based, move user-facing services onto SLO burn-rate alerting, add grouping and dependency inhibition at the alerting layer. And I'd fix the noisiest alert's underlying defect rather than suppress it — usually the noise is the message."

**The measurement point.** "The metric I'd track is precision — actionable pages over total pages — and separately recall, meaning what fraction of incidents were first reported by a human rather than by monitoring. Most teams have never measured the second one, and it's the one that tells you whether you're blind."

**Your evidence.** [ONE concrete thing you did — see §3.]

### Delivery notes

- **Lead with the operator credential, not the concept.** "On-call across 60,000 nodes" earns you the right to have an opinion. Reversing that order makes it sound theoretical.
- **The Gartner rename is the credibility spike.** It's specific, checkable, slightly contrarian, and almost nobody knows it.
- **"Most alert noise is a design defect" is a position, and positions get remembered.** Deliver it as a considered conclusion, not a hedge.
- **This is a stronger point than TP1 for you**, because it's grounded in work you actually did. If you can only rehearse one, rehearse this.

### Likely follow-ups

| They ask | You say |
|---|---|
| "What tooling did you use?" | Splunk and ELK. **Be precise about configuration vs. building.** "I configured correlation searches and suppression rules" is credible and fine. "I built ML models on log data" invites a question about your false-positive rate that you can't answer. |
| "So is AIOps worth buying?" | Conditionally. It compounds on good observability and can't substitute for it — garbage in, confidently-labeled garbage out. And expect a meaningful uplift on your observability spend, often more than the toil it removes. The read-only half — alert enrichment, surfacing similar past incidents and recent changes — is the best risk-adjusted piece, because failure is cheap. |
| "What about anomaly detection?" | Useful on metrics with strong seasonality where a static threshold is too loose off-peak and too tight at peak. But an anomaly isn't a problem — a launch or a traffic shift is statistically anomalous with zero user impact. So anchor paging on SLO burn rate, which is user-impact-anchored by construction, and use anomaly detection as an investigation aid. |
| "How would you measure whether it worked?" | Staged rollout with a holdout set of services, not a big-bang cutover and a before/after chart. MTTR is confounded by traffic, deploy rate, and seasonality. Precision and compression ratio are cleaner. |
| "What's Google doing here?" | Only if asked. One sentence: they published an autonomy-level framework this year, agents sit at partial autonomy with human approval for anything meaningful, and their measured wins came from read-only enrichment rather than autonomous action. Don't recite the paper. |

### Do not say

- Any vendor's marketing numbers as if they were yours.
- "We reduced alert noise significantly." Significantly is not a number.
- Anything that sounds like you're dismissing AIOps wholesale. The position is *sequencing* — hygiene first, correlation for the residual — not rejection.

---

## 3. The blocker

**TP2 is 80% complete and 100% blocked on one thing: you still have an `[X]` placeholder where a real number goes.**

Every one of these follow-ups terminates in "and what was the impact?" A talking point that lands the method and then trails off into "it improved things" actively hurts you — it signals you observed the work rather than owned it, which is precisely the L5 down-level risk you're already carrying.

You need **one** of the following, and any of them is sufficient:

- MTTR before/after on a class of incidents
- Alert or ticket volume before/after
- Page-to-actionable-page ratio
- MIM incidents handled per quarter

Directional and defensible beats precise and absent. "Roughly a third fewer pages per shift after we consolidated the storage alerts" is a fine answer. Silence is not.

This same number feeds resume bullet E2, the LinkedIn PE section, and your incident STAR story. **Four artifacts, one number, and it's been open since July 16.** Close it before the first mock.

---

## 4. Rehearsal protocol

1. Say the 30-second version out loud, timed. If it runs past 40 seconds, cut a clause — you're padding.
2. Have someone (or a mock) ask two follow-ups cold. The follow-ups are where these actually get tested; the opener just buys you the floor.
3. Record yourself once on TP2. The specific failure mode to listen for is your voice speeding up on the method list — four items delivered fast sounds rehearsed, delivered with beats sounds considered.
4. Practice the TP1 disclaimer until it sounds neutral rather than apologetic. "I haven't operated ML serving in production" should land like a fact, not a confession.

---

*Suggested location: `sre-prep-notes/behavioral/talking-points-ai-sre.md`. Source material: `concepts/ai-sre-reliability.md` and `concepts/aiops-fundamentals.md`.*
