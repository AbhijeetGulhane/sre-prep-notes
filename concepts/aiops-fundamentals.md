# AIOps — One-Stop Reference

**Scope:** Week 6 AI-SRE/AIOps literacy block, part 2. Companion to `ai-sre-reliability.md`. Where that doc covered *SRE applied to ML systems*, this one covers *ML applied to ops data* — alert noise reduction, event correlation, anomaly detection, and the honest state of the field in mid-2026.

**Status:** Literacy + narrative-mining. This one has more direct payoff than part 1, because you have real ELK/Splunk operational experience at 60k-node scale, and this vocabulary lets you *reframe work you've already done* rather than claim work you haven't.

---

## 0. Calibration

Three things to name before anything else.

**One: the term is compromised.** "AIOps" was coined by Gartner in 2017 and then stretched past usefulness by a decade of vendor marketing. <cite index="22-1">Gartner renamed the category to "Event Intelligence Solutions" in 2025 to narrow the market definition back to its intended domain and use cases</cite>. That single fact is the highest-signal thing in this document. Saying *"AIOps is basically an analyst-and-marketing category — Gartner actually renamed it Event Intelligence Solutions in 2025, which is a fair description of what the working subset really is: event correlation and noise reduction"* immediately separates you from candidates who read a vendor page.

**Two: capability maturity is wildly uneven, and vendors do not distinguish.** <cite index="22-1">Alert correlation and knowledge retrieval are the most production-validated capabilities, while fully autonomous remediation remains aspirational</cite>. Know which claims are load-bearing and which are brochure. A candidate who says "AIOps reduces MTTR" is repeating marketing; one who says "correlation and dedup are real and measurable, predictive prevention is mostly forecasting with a good PR team, and autonomous remediation is nearly always a runbook with an ML-shaped trigger" is credible.

**Three — and this is the one that matters for you: your Barclays background is the asset, the AIOps vocabulary is just the label.** You have ELK/Splunk at 60,000+ nodes and a MIM (major incident management) track record. That is *actual event-intelligence operating experience*. The mistake would be spending this evening learning vendor taxonomy instead of mining your own history for the two or three concrete stories that this vocabulary lets you retell more legibly to a Google or Meta interviewer. **Section 6 is the part of this document you should spend the most time on.** Everything before it is context.

Budget: one evening, and most of it on §5, §6, and §7.

---

## 1. What AIOps actually means

The standard definition, effectively unchanged since 2017: <cite index="19-1">Gartner's framing is that AIOps combines big data and machine learning to automate IT operations processes, including event correlation, anomaly detection and causality determination</cite>. Dynatrace's version is broader: <cite index="14-1">the practice of using AI to identify issues, reveal answers, and automate remediation by ingesting data from every layer of the stack and continuously analyzing it</cite>.

Strip the marketing and you get:

> **AIOps = statistical/ML techniques applied to operational telemetry (logs, metrics, traces, events, change records, tickets) to compress a high-volume, low-signal stream into a small number of actionable items.**

That's it. It is a **signal-compression problem**, not an intelligence problem. Framing it that way is both more accurate and more useful.

### 1.1 The problem it exists to solve

The structural driver is real, not invented: <cite index="22-1">microservices architectures generate correlated failure signals across dozens of observability surfaces at once — a single service degradation produces CPU, latency, and error-rate alerts across multiple tools, each appearing as an independent incident</cite>. Scale figures commonly cited: <cite index="21-1">over 40% of IT organizations receive more than a million event alerts daily, and 11% receive over ten million</cite>. Treat those numbers as directionally true and vendor-sourced; don't quote them as fact in an interview.

You've lived the underlying problem. One storage array degrades, and you get 400 host-level alerts, 60 application alerts, a synthetic-monitor failure, and three ticket auto-creations — all describing one event.

### 1.2 AIOps vs observability — get this distinction right

<cite index="25-1">Observability is the ability to ask arbitrary questions of your system using telemetry. AIOps is the AI layer on top of observability data that detects, correlates, and sometimes suggests or fires remediation. You need observability before you can have useful AIOps; AIOps without good observability inputs produces noisy, unreliable output.</cite>

**Garbage in, confidently-labeled garbage out.** This is the single most important operational truth about the category and the reason most enterprise deployments underperform: teams buy AIOps to compensate for bad instrumentation, inconsistent tagging, and no service topology. It doesn't work. Correlation requires knowing that host A serves service B which depends on service C — and if your CMDB is stale, no model recovers that.

---

## 2. The data substrate — why ops data is hard

Worth understanding because it explains why the "just throw ML at it" instinct fails.

| Property | Consequence |
|---|---|
| **Extreme class imbalance** | Real incidents are maybe 1 in 10⁶ events. Any classifier can hit 99.9999% accuracy by always predicting "fine." Accuracy is a useless metric here; precision/recall on the minority class is the only thing that matters. |
| **Almost no labels** | Nobody labels "this alert was actionable." Incident tickets are the closest thing, and they're inconsistent, late, and written under pressure. Supervised learning is starved. |
| **Non-stationary** | Deploys change system behavior constantly. Yesterday's baseline is wrong today. Models decay fast — this is exactly the concept drift problem from part 1, applied to the ops domain. |
| **Strong seasonality, multi-period** | Daily, weekly, monthly (batch close), quarterly. Naive anomaly detection fires every Monday 9am and every month-end. Seasonality handling is table stakes, not a feature. |
| **Heavy-tailed, bursty** | One incident generates a burst that looks statistically like an anomaly by volume alone. |
| **Semi-structured text** | Logs aren't natural language and aren't structured records. They're templates with variable slots — which is exactly why log parsing gets its own algorithms (§4.1). |
| **Topology is the missing key** | The single most valuable feature for correlation is the dependency graph, and it usually lives in a CMDB that's 30% wrong. |

**Interview-grade line:** *"The hard part of AIOps isn't the modeling, it's that you have a million-to-one class imbalance, essentially no labels, a non-stationary target, and the most predictive feature — service topology — is the one nobody maintains accurately."*

---

## 3. Capability map with honest maturity ratings

| # | Capability | What it does | Maturity | Notes |
|---|---|---|---|---|
| 1 | **Event correlation / dedup** | Collapse many alerts into one incident | **Production-real** | The proven core. Mostly rules + clustering + topology, less ML than advertised. |
| 2 | **Alert noise reduction** | Suppress non-actionable alerts | **Production-real** | Vendors cite up to ~95% noise removal. Achievable — but see §5, much of that is fixable without ML. |
| 3 | **Anomaly detection** | Flag unusual metric behavior without static thresholds | **Real but noisy** | Works well on smooth, seasonal, high-volume metrics. Poor on spiky, low-volume ones. Tuning cost is high. |
| 4 | **Root cause analysis** | Point at the causal component | **Partial** | Good at "what changed and when" (change correlation). Weak at genuine causality. Topology-based fault-tree approaches outperform pure statistical ones. |
| 5 | **Knowledge retrieval / alert enrichment** | Surface past similar incidents, relevant runbooks, recent changes | **Production-real, underrated** | RAG over incident history. Highest ROI-to-risk ratio in the whole category. Read-only, so failure is cheap. |
| 6 | **Predictive / forecasting** | Warn before saturation | **Real but oversold** | Capacity forecasting works (it's linear regression on disk fill). "Predict the outage before it happens" mostly doesn't. |
| 7 | **Automated remediation** | Fix it without a human | **Aspirational at the general case** | Works for a narrow catalog of well-understood, low-blast-radius actions. See the Google autonomy ladder in part 1 — nearly all deployed systems sit at L1–L2. |

If you remember one row: **#5 is the sleeper.** Alert enrichment is read-only, cheap, low-risk, and Google measured <cite index="11-1">a 10% MTTM reduction from informational assistance alone</cite> — with their Investigation Dashboards delivering <cite index="11-1">roughly 44% MTTM reduction for supported incidents</cite>. No autonomy required.

---

## 4. Techniques — enough depth to hold a conversation

### 4.1 Log parsing and templating

The foundational step nobody outside the field knows about. Raw logs are unusable as ML features; you first extract **templates** (the static skeleton) from **parameters** (the variable parts).

```
Input:  "Connection to db-prod-17 failed after 3021ms"
        "Connection to db-prod-04 failed after 1897ms"
Template: "Connection to <*> failed after <*>ms"
```

Once logs are templatized, everything downstream becomes tractable: you can count template frequencies, detect *new* templates (a template never seen before is a strong incident signal), and cluster.

- **Drain** — fixed-depth parse tree, streaming, O(1)-ish per log line. The de-facto standard. Know this name.
- **Spell** — longest-common-subsequence based, streaming.
- **IPLoM, LogSig, LenMa** — older, mostly academic now.
- **Elastic's `categorize_text` / ML categorization jobs** — this is Drain-family logic productized. **You may already have used this in ELK without knowing the algorithm name.** Worth checking.
- **Splunk `cluster` command and Log Event Clustering in ITSI** — same idea, different implementation.

**The killer primitive: new-template detection.** A log template that has never appeared before, appearing at rate, is one of the highest-precision incident signals available and requires no labels. Cheap, explainable, effective.

### 4.2 Event correlation — three families

**(a) Rule-based / deterministic.** Grouping keys, inhibition rules, dependency suppression. Prometheus Alertmanager's `group_by`, `inhibit_rules`, and silences are exactly this. Boring, explainable, and handles the majority of real noise. **Start here always.**

**(b) Statistical / temporal.** Alerts co-occurring within a time window, historically, get grouped. Techniques: co-occurrence matrices, association rule mining (FP-growth, Apriori), sequential pattern mining. Learns from history without labels. Failure mode: correlates things that always happen together for unrelated reasons (two jobs that both run at 02:00).

**(c) Topology-based.** Use the service dependency graph to walk from symptom to probable cause. This is what Dynatrace's Davis does — <cite index="18-1">using application topology and service flow maps together with high-fidelity metrics to perform fault tree analysis</cite>, and Dynatrace explicitly markets it as <cite index="20-1">deterministic and causation-based, with no model training</cite>, positioning against pure-ML competitors.

**That positioning is worth understanding, because it's the real technical argument in the space:** correlation-based ML tells you things moved together; topology-based fault-tree analysis tells you the direction of causation, because the dependency graph encodes it. **Causality doesn't come from data, it comes from structure.** If you can say that sentence in an interview you sound like you've thought about it rather than read about it.

Practical answer: (c) works if your topology is accurate. If it isn't — and at a bank, it usually isn't — (a) plus (b) is what you actually get.

### 4.3 Anomaly detection on time series

Know the ladder, roughly in order of how often it's the right answer:

1. **Static thresholds** — still correct for most things. Don't be embarrassed by them.
2. **Seasonal decomposition (STL) + residual thresholding** — decompose into trend/seasonal/residual, alert on residual. Handles the Monday-9am problem. Usually sufficient.
3. **Holt-Winters / exponential smoothing** — forecast the next value, alert on prediction error. Cheap, multi-seasonal variants exist. This is what most "AI-powered baselining" actually is.
4. **Prophet / ARIMA-family** — better on trend + multiple seasonalities + holidays. Heavier.
5. **Changepoint detection (CUSUM, Bayesian online changepoint)** — detects level shifts rather than spikes. Excellent for post-deploy regression detection; badly underused.
6. **Isolation Forest / robust covariance** — multivariate, unsupervised, no seasonality awareness on its own.
7. **Autoencoder / LSTM reconstruction error** — multivariate, learns normal, flags what it can't reconstruct. Expensive, opaque, hard to debug at 3am. Usually not worth it.

**The distinction that matters more than the algorithm: anomaly ≠ problem.** Google states this plainly in the 2026 SRE paper: <cite index="11-1">statistical anomalies in system metrics don't always equate to user impact, largely because these signals lack a deep understanding of user intent — a new feature launch or standard traffic shift can easily be misread as a failure</cite>.

This is the single strongest argument for **symptom-based, SLO-driven alerting over anomaly detection**. An SLO burn-rate alert is by construction user-impact-anchored. An anomaly detector is by construction only statistically anchored. When an interviewer asks about anomaly detection, this is the pivot that shows SRE judgment.

### 4.4 The LLM layer (2024–2026)

What changed recently is a shift from classification to **synthesis**. <cite index="25-1">The 2017 definition mostly shipped noise reduction; the 2026 version ships incident drafts — a model reads the alert storm, correlates it against recent deploys and topology, writes a probable root-cause paragraph, suggests a runbook step, and opens a channel with on-call tagged</cite>. Google's equivalents are AI Alert (read-only enrichment within a ~2 minute budget) and Incident Hypothesis (see part 1, §5.4).

The architectural pattern is consistent everywhere: **RAG over incident history + runbooks + change logs + topology, producing evidence-linked hypotheses rather than conclusions**. Google is explicit that <cite index="11-1">AI Alert focuses on verifiable facts and evidence-based insights rather than speculative conclusions, with all findings linked back to source data</cite>.

Also genuinely new: **unstructured user feedback as a detection signal.** Google's Detectr <cite index="11-1">aggregates signals from social media, customer support, and product forums through a filter → cluster → de-noise → report pipeline, acting as a backstop to metric-based monitoring for novel issues</cite>. The clever part is the noise filter is economic: <cite index="11-1">the volume of related feedback indicates severity, while the effort required for a user to submit feedback acts as a filter against noise</cite>. At a bank, the equivalent input is call-center ticket volume and branch/relationship-manager escalations — and I'd bet Barclays doesn't feed that into detection.

---

## 5. The non-ML baseline that beats most AIOps deployments

**Read this section twice. It is the most SRE-flavored content in the document and the highest-scoring thing you can say on this topic.**

The uncomfortable truth: **most alert noise is a design defect, not a data science problem.** Before any ML, the following typically removes 70–90% of noise, deterministically, explainably, at zero licensing cost:

1. **Alert on symptoms, not causes.** SRE Book Ch. 6. Page on "users are getting errors," not on "CPU is 90%." High CPU with a healthy SLO is not an incident. This alone deletes entire alert categories.
2. **Replace threshold alerts with SLO burn-rate alerts.** Multi-window multi-burn-rate (SRE Workbook Ch. 5). Converts hundreds of component thresholds into a handful of user-impact alerts with tunable precision/recall. **This is your planned SLO Error Budget Service project — which means that project is directly a noise-reduction project, and you should describe it that way.**
3. **Every page must be actionable.** If the runbook says "wait and see," it's a ticket or a dashboard, not a page. Enforce ruthlessly.
4. **Grouping and inhibition at the alerting layer.** Alertmanager `group_by` on cluster/service, `inhibit_rules` so a node-down suppresses all its pod alerts. Deterministic dependency suppression buys most of what "AI correlation" claims.
5. **Deduplicate at the source.** One alert per condition per service, not per instance.
6. **Delete alerts nobody acts on.** Audit 90 days of alerts against actions taken. The distribution is always brutal: a small handful of alert rules produce most of the volume and nearly none of the actions.
7. **Fix the top offender.** The noisiest alert usually reflects a real reliability defect. Fix the system, not the alerting.

**The interview line:** *"Before I'd buy an AIOps platform, I'd want to know what the alert-to-actionable-page ratio is and whether alerting is symptom-based. If you're paging on causes with static thresholds, ML on top of that is compressing noise you should have deleted. AIOps is worth it for the residual correlation problem you still have after doing the SRE hygiene, not as a substitute for it."*

That is a senior answer. It's also a defensible position rather than an opinion about vendors, and it maps directly onto the L4-vs-L5 signal you're trying to send.

---

## 6. Mapping to your Barclays ELK/Splunk background

**This is the section that pays. Spend your time here.**

### 6.1 What you likely already have that counts

Go through your actual work and identify which of these you can speak to concretely:

| AIOps concept | Likely Barclays equivalent | Mine this if... |
|---|---|---|
| Log templating / categorization | Splunk `cluster`/`transaction`; Elastic ML categorization jobs | You built or tuned any log-pattern-based detection |
| Event correlation | Splunk ITSI notable events + correlation searches; ELK Watcher rules | You defined correlation searches or notable-event aggregation policies |
| Noise reduction | Any alert-tuning or suppression work | You reduced ticket/page volume — **get the number** |
| Anomaly detection | Elastic ML anomaly jobs; Splunk ITSI adaptive thresholding / KPI baselining | You configured baselines rather than static thresholds |
| Service topology | ITSI service trees / entity definitions; CMDB integration | You modeled service dependencies for alerting |
| Incident correlation at scale | MIM role | **Always. This is your strongest card.** |
| Knowledge retrieval | Runbook/knowledge-base work, postmortem process | You built or maintained runbooks tied to alerts |

### 6.2 How to talk about it

**Do:** *"At Barclays we run Splunk and ELK across a 60,000-node estate. The practical problem isn't detection, it's correlation and suppression — a single infrastructure event fans out into hundreds of downstream alerts. I've worked on the event-intelligence side of that: [specific thing]. The lever that mattered most was [suppression rules / symptom-based alerting / service-tree modeling], not the ML features."*

**Don't:** claim you built ML models on ops data if you configured vendor features. An interviewer will ask which algorithm and what your false-positive rate was, and the answer "the vendor handles that" is fine *if* you framed it as configuration up front, and fatal if you framed it as modeling.

**The honest and strong framing is operator, not builder:** you've run event intelligence at a scale most candidates haven't touched, you know where it breaks, and you know what the non-ML baseline gets you. That's a more credible and more senior position than "I trained a model."

### 6.3 The number you still don't have

You have an open `[X]` placeholder for a real MTTR figure feeding your PE resume bullet, the LinkedIn PE section, and your incident STAR story. **This topic is the reason to close it this week, not later.** Every question in this space — "how did you reduce alert noise," "how did you improve incident response," "what was the impact" — terminates in a number. Candidates without one sound like they observed the work rather than owned it.

Get at least one of: MTTR before/after, alert volume before/after, page-to-incident ratio, or MIM incidents handled per quarter. Any of them. Directional and defensible beats precise and absent.

---

## 7. How to evaluate an AIOps deployment (metrics)

If asked "how would you know if it's working" — and this is the most likely probing question — do **not** answer "MTTR went down." Answer with a measurement design.

**Detection quality (treat alerting as a classifier):**
- **Precision** = actionable pages / total pages. The number that governs on-call sanity.
- **Recall** = incidents caught by alerting / total incidents (including customer-reported). The number that governs whether you're blind.
- **Alert-to-incident compression ratio** = raw events / correlated incidents. The direct measure of correlation value.
- **False-negative source mix** — what fraction of incidents were first reported by a human? That's your recall gap, and it's the metric Google's Detectr exists to attack.

**Response quality:**
- MTTD, MTTA, MTTM/MTTR — but only as *paired before/after with a control*, because they're confounded by everything.
- Time-to-first-correct-hypothesis — closer to what enrichment actually improves.

**Human cost:**
- Pages per on-call shift, and pages outside business hours.
- Toil hours per week on alert triage.
- On-call handover length.

**The methodological point that scores:** these metrics are confounded by traffic, deploy rate, seasonality, and org changes. Google has the scale to A/B test operational practice — <cite index="11-1">they A/B tested Incident Hypothesis and attributed a 10% MTTM reduction to it</cite>. Most orgs can't. So the honest approach is a **staged rollout by service or team with a holdout**, not a big-bang cutover followed by a before/after chart. Saying "I'd want a holdout set of services" is a strong signal.

---

## 8. Failure modes

| Failure | Mechanism |
|---|---|
| **Garbage-in** | Inconsistent tagging, no topology, no service ownership. Models can't recover missing structure. |
| **Automating around a defect** | Suppressing a noisy alert that reflects a genuine reliability problem. The noise was the message. |
| **Black-box distrust** | Unexplainable grouping → on-callers ignore it → shadow dashboards → net negative. **Explainability is an adoption requirement, not a nice-to-have.** |
| **Model decay** | Non-stationary environment; baselines learned pre-migration are wrong post-migration. Nobody owns retraining. |
| **Learning the outage as normal** | Adaptive baselines trained on a period containing an incident absorb it into "normal." Needs incident-window exclusion. |
| **Correlation feedback loop** | Suppressed alerts stop generating incident records, so the correlation model sees less evidence of that failure class and suppresses harder. |
| **Cost** | <cite index="25-1">Expect a 20–40% uplift over base observability spend once AIOps features are enabled</cite>. Often more than the toil it removes. |
| **Autonomy overreach** | Auto-remediation without dry-run, rate limits, or circuit breakers. See the Google guardrails in part 1 §5.3 — those exist because this failure is expensive. |

---

## 9. Interview usage

### 9.1 Answer skeletons

**Q: "What do you think of AIOps?"**
Lead with the honest framing: it's an analyst-and-vendor category, Gartner itself renamed it to Event Intelligence Solutions in 2025 to narrow it back to what works. The proven subset is event correlation, dedup, and alert enrichment. Anomaly detection is real but noisy and expensive to tune. Autonomous remediation is mostly aspirational outside a narrow action catalog. Then the SRE point: most alert noise is a design defect — symptom-based alerting and SLO burn-rate alerting delete far more noise than ML compresses, and you should do that first.

**Q: "How would you reduce alert noise on a system generating 10,000 alerts a day?"**
Don't jump to ML. Audit first: 90 days of alerts joined against actions taken, ranked by volume. Expect a handful of rules to dominate. Then in order: delete non-actionable alerts, convert cause-based to symptom-based, move to SLO burn-rate alerting for user-facing services, add grouping and dependency inhibition at the alerting layer, dedupe per-service instead of per-instance. Measure precision (actionable pages / total pages) throughout. *Then*, for the residual correlation problem — genuinely independent alerts that share a root cause across domains — that's where clustering and topology-based correlation earn their keep. Close with: I'd fix the noisiest alert's underlying defect rather than suppress it.

**Q: "How does anomaly detection differ from threshold alerting, and when would you use each?"**
Thresholds encode a known bad state; anomaly detection encodes a deviation from learned normal. Thresholds are right whenever you actually know the bad value — which is most of the time, and there's no shame in it. Anomaly detection earns its place on metrics with strong multi-period seasonality where a static threshold is either too loose off-peak or too tight at peak. But the critical caveat is that an anomaly is not a problem — a launch or traffic shift is a statistical anomaly with zero user impact. So I'd anchor paging on SLO burn rate, which is user-impact-anchored by construction, and use anomaly detection as an investigation aid rather than a paging source.

**Q: "Have you worked with AIOps tooling?"** → §6.2. Operator framing, honest about configuration vs. modeling, and land on a number.

### 9.2 Reverse questions (genuinely good ones)

- "What's your alert-to-actionable-page ratio, and how do you track it?"
- "Where does your team sit on the autonomy ladder for automated mitigation — is anything actuating without human approval?"
- "How much of your incident detection comes from monitoring versus user reports?"

The third one is the sharpest — it's a recall question, and most teams have never measured it.

---

## 10. Reading list

### Essential — free, ~90 minutes total

| Resource | Where | Why |
|---|---|---|
| **SRE Book Ch. 6 — Monitoring Distributed Systems** | `sre.google/sre-book/monitoring-distributed-systems/` | Symptom-vs-cause alerting. **The actual answer to alert noise.** Non-negotiable. |
| **SRE Workbook Ch. 5 — Alerting on SLOs** | `sre.google/workbook/alerting-on-slos/` | Multi-window multi-burn-rate. Already on your project reading list — this is the second reason to read it. |
| **SRE Book Ch. 10 — Practical Alerting** | `sre.google/sre-book/practical-alerting/` | Borgmon-era but the principles hold. |
| **AI in SRE (Google, 2026)** | `sre.google/resources/practices-and-processes/ai-engineering-reliable-operations/` | Already read for part 1. The Detectr and AI Alert sections are the AIOps-relevant ones. |
| **What is AIOps? (Dynatrace)** | `dynatrace.com/news/blog/what-is-aiops-2/` | Vendor overview. Read it *as a vendor overview* — useful for the causal-vs-correlational positioning argument, not for neutral definitions. |

### Supplementary — free, only if it keeps coming up

| Resource | Where | Why |
|---|---|---|
| **Prometheus Alertmanager docs — grouping & inhibition** | `prometheus.io/docs/alerting/latest/configuration/` | The deterministic baseline, concretely. 20 minutes, immediately usable. |
| **Drain: An Online Log Parsing Approach with Fixed Depth Tree** (He et al., ICWS 2017) | search the title | The log-parsing algorithm behind most productized log categorization. 8 pages. |
| **Loghub / LogPAI** | `github.com/logpai` | Public log datasets + benchmark implementations of the parsers. Useful if you ever want to actually build something here. |
| **Gartner Market Guide for Event Intelligence Solutions** | `gartner.com/reviews/market/event-intelligence-solutions` | Vendor landscape (BigPanda, BMC Helix, Moogsoft, etc.). Skim the vendor names only — useful for fintech interviews where these are actually deployed. |
| **Elastic ML anomaly detection docs** | `elastic.co/guide` → Machine learning | Directly maps to your ELK background. Read the categorization job section specifically. |
| **Splunk ITSI event analytics docs** | `docs.splunk.com` → ITSI | Same reasoning. Notable events, correlation searches, adaptive thresholding. |

### Paid — buy none of these

| Resource | Cost | Verdict |
|---|---|---|
| *Observability Engineering* — Majors, Fong-Jones, Miranda | ~$60 | Already on your Tier-1 project list. Ch. on alerting is relevant. Not AIOps-specific. **After Week 10.** |
| *Practical Monitoring* — Mike Julian | ~$30 | Good, dated, and everything useful in it is also in SRE Book Ch. 6. **Skip.** |
| Any vendor-authored AIOps ebook | free but costly | Marketing. Skip. |

---

## 11. Out of scope

Deliberately not covering: specific vendor product comparisons and pricing, ITSM/ServiceNow integration patterns, security-side AIOps (UEBA, SIEM correlation), network-specific AIOps (NMS), and capacity-optimization/FinOps. Each is a real specialization; none will be scored in an SRE-SysEng loop.

**Action item out of this doc that isn't reading: close the MTTR `[X]` placeholder.** That's worth more to your August application than every link above combined.

---

*Suggested location: `sre-prep-notes/concepts/aiops-fundamentals.md`, alongside `ai-sre-reliability.md`. Not a system-design topic, so no `NN.M-` prefix.*
