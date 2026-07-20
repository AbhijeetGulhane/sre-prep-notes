# AI/ML Reliability Through an SRE Lens — One-Stop Reference

**Scope:** Week 6 AI-SRE/AIOps literacy block. Covers (A) applying SRE practice to ML systems — drift, model rollouts, silent failures — and (B) the inverse, AI applied to SRE operations, which is where Google's 2026 public material actually sits.

**Status:** Literacy block, not a study block. Read once, internalize the vocabulary, be able to hold a 5-minute conversation. Do not sink a weekend into this.

---

## 0. Calibration — read this before anything else

Two things need naming up front.

**First: the source you cited is not about what you think it's about.** `sre.google/resources` currently headlines "AI in SRE: How Google is Engineering the Future of Reliable Operations" (Papapanagiotou, Malesevic, Heiser, Meshenberg). That paper is about **AI agents operating production** — autonomy levels, safety guardrails, agentic mitigation. It is *not* about data drift or ML model canaries. The material for your stated subtopics lives elsewhere: the Google Cloud blog post *An SRE's guide to optimizing ML systems with MLOps pipelines* (Saltonstall & Virji, Feb 2025) and *A Brief Guide To Running ML Systems in Production*. If you cite "sre.google's AI reliability work" in an interview and then talk about PSI thresholds, you will look like you skimmed a headline. Know which is which.

**Second: expected interview yield is low.** This will not be a scored round for SRE-SysEng at Google. Where it pays:
- Behavioral/"how do you stay current" — one crisp paragraph, high signal, cheap.
- The reverse-questions slot at the end of a loop — asking about agentic ops maturity signals you read the 2026 paper.
- Meta PE and fintech (Two Sigma, Citadel) — model-serving reliability is closer to a real job function there.

Where it does not pay: NALSD, Linux troubleshooting, coding, the debugging round. **Budget: one evening. If you find yourself building a drift-detection demo, you've misallocated.** Your actual gaps are production-cloud experience and timed live-debugging reps — this is not one of them.

---

# PART A — SRE Practice Applied to ML Systems

## 1. The core reframe

An ML system is **a data pipeline with a stochastic function bolted on the end**. That's it. The reframe does most of the work in an interview: everything you already know about pipelines — freshness SLOs, completeness, idempotency, backfills, unattended operation — transfers directly. The Google blog makes exactly this argument: <cite index="10-1">ML relies on pipelines, so start with pipeline management, then look at training, model freshness, and efficiency</cite>.

What does *not* transfer is the part where correctness is a distribution rather than a boolean. A conventional service is either returning the right answer or an error. An ML service can return a well-formed, low-latency, HTTP-200 answer that is **wrong**, and no golden signal will move. That single property generates most of the discipline below.

### The three-layer model (use this to structure any answer)

| Layer | What breaks | How you detect it | Detection latency |
|---|---|---|---|
| **Infrastructure** | Serving pods, GPUs, network, autoscaling | Golden signals: latency, traffic, errors, saturation | Seconds |
| **Data** | Schema change, upstream join drops rows, staleness, skew | Freshness / completeness / distribution SLIs | Minutes → hours |
| **Model** | Accuracy decay, calibration drift, feedback poisoning | Proxy metrics now, ground-truth metrics later | Hours → weeks |

The failure mode of most orgs: they instrument layer 1 beautifully and layer 3 not at all. Detection latency grows by orders of magnitude as you descend, and **business impact grows with it**.

---

## 2. Data drift as a reliability signal

### 2.1 Taxonomy — get these terms right

Interviewers who know ML will catch you conflating these.

| Term | Formal | Plain English | Example |
|---|---|---|---|
| **Covariate / data drift** | P(X) changes, P(Y\|X) stable | Inputs look different; the relationship still holds | New geography onboarded; feature distributions shift |
| **Concept drift** | P(Y\|X) changes | The world changed meaning; same inputs now imply different outputs | Fraud tactics evolve; the same signature is now benign |
| **Label / prior drift** | P(Y) changes | Base rate moved | Fraud rate jumps 3× during a campaign |
| **Training–serving skew** | P_train(X) ≠ P_serve(X) *at the same instant* | Not drift at all — a bug | Feature computed in Spark for training, in Java for serving; rounding differs |
| **Upstream data drift** | Producer changed | Someone else's deploy broke you | Client app starts sending `country` as ISO-3 instead of ISO-2 |

**Skew is the one that matters most operationally**, and it is the one people mislabel as drift. Drift is the world moving. Skew is your own code being inconsistent with itself. Skew is a bug with a root cause and a fix; drift is a condition you manage. Getting this distinction out loud in an interview is a strong signal.

### 2.2 Why drift is an SLO problem, not an alert

Drift is a **slow burn**. It has no page-worthy edge. If you alert on "PSI > 0.2" you will get paged at 3am for a marketing campaign and you will start ignoring it within two weeks.

The correct framing is **error budget**, not threshold alerting: define a model-quality SLO, and treat drift metrics as *leading indicators* that burn the budget. Burn-rate alerting (multi-window, multi-burn-rate) is the same machinery as any other SLO — this is directly reusable from your planned SLO Error Budget Service project.

**Example SLO set for an ML service:**

| SLI | Definition | Example objective |
|---|---|---|
| Feature freshness | Age of newest data used to serve an inference | 99% of inferences use features < 15 min old |
| Feature completeness | Fraction of requests where all required features resolved (no default/imputed fill) | 99.5% over 28d |
| Prediction distribution stability | JS divergence of today's score distribution vs. a fixed reference window | JSD < 0.05 on 95% of hours |
| Model quality | AUC / precision@k on delayed ground truth | ≥ 0.87 on trailing 7d labeled sample |
| Serving parity | Fraction of shadow-replayed requests where offline and online features match bit-for-bit | 99.99% |

Freshness deserves special attention because it's the one Google explicitly calls out: <cite index="10-1">develop SLOs to measure pipeline health, especially data freshness — how recently the model got the data it's using to produce an inference — because stale data leads to lower-quality inferences, and requirements vary enormously by product (minutes for weather forecasting, days or longer for a spell-checker)</cite>. Freshness is part of the critical user journey, not an infra detail.

### 2.3 Detection methods — know the tradeoffs

**Distribution distance (per-feature, per-window):**

- **PSI (Population Stability Index)** — `Σ (p_i − q_i) · ln(p_i / q_i)` over binned values. Symmetric KL, essentially. Industry convention: < 0.1 stable, 0.1–0.25 moderate, > 0.25 significant. Cheap, interpretable, binning-sensitive. Dominant in credit risk — **know this one cold for fintech interviews**; Barclays model-risk teams live on it.
- **KL divergence** — asymmetric, blows up when the reference has zero mass in a bin. Needs smoothing.
- **Jensen–Shannon divergence** — symmetric, bounded [0,1] (log base 2), well-behaved. Better default than KL for alerting because it's bounded and therefore thresholdable.
- **Kolmogorov–Smirnov** — non-parametric, continuous features, no binning. Trap: at high volume, KS p-values go to zero on differences of no practical consequence. **Never alert on a p-value at scale.** Use effect size.
- **Wasserstein / earth-mover** — respects ordering and magnitude of shift. Better than KS for "the whole distribution slid right by 5%".
- **Chi-squared** — categorical features. Same large-n p-value trap.

**Cheap structural checks that catch more real incidents than any of the above:**
- Null rate per feature, per window
- Cardinality of categorical features (new unseen category = someone changed an enum)
- Min/max/range violations against a declared schema
- Row-count deltas vs. same-hour-last-week
- Type changes

This is the 80/20. Sophisticated divergence metrics catch subtle drift; **null-rate and cardinality checks catch the outage.** When a feature pipeline silently starts emitting `0.0` for a column, PSI will scream — but so will the null-rate monitor, and the null-rate monitor is explainable to an on-caller at 3am.

**Multivariate drift:** individual features can each look stable while the joint distribution shifts. Practical approaches: train a discriminator to distinguish reference-window from current-window samples — if it achieves meaningfully better than 0.5 AUC, the distributions differ (this is "domain classifier" drift detection). Or monitor drift in a learned embedding / PCA projection. Mention this only if pressed; it's rarely deployed.

### 2.4 The hard problem: no ground truth

The thing that separates people who've operated ML from people who've read about it.

You usually cannot compute accuracy in real time, because labels arrive late or never:
- Fraud: chargeback confirms the label 30–90 days later.
- Recommendations: no counterfactual — you never see what the user would have clicked on the other ranking.
- Credit default: 12–24 month horizon.
- Medical: possibly never.

So your online quality signals are all **proxies**:

| Proxy | What it catches | Failure mode |
|---|---|---|
| Prediction distribution shift | Model behaving differently | Could be legitimate input change |
| Confidence/entropy distribution | Model uncertain in new regions | Miscalibrated models lie confidently |
| Feature-importance drift | Model relying on different signals | Expensive to compute continuously |
| Business metric (CTR, approval rate) | Real impact | Confounded by everything else in the product |
| Human review sample | Ground truth | Expensive, slow, small-n |
| Fallback/default-path rate | Serving degradation | Only catches hard failures |

**The honest answer to "how do you know the model is still good?" is: you don't, in real time. You build layered proxies, you reserve a labeled holdout stream, and you accept a detection latency measured in days for the quality layer while keeping detection latency in seconds for the layers above it.** Saying this plainly is a much better interview answer than pretending drift detection solves it.

**Delayed-label reconciliation** is the concrete practice: keep a durable log of `(request_id, features, model_version, prediction, timestamp)`, and join it against labels as they land. That join is your real quality dashboard, permanently lagging. It also gives you exact per-model-version attribution when you need to answer "did the rollout cause this?" three weeks later. Design that log deliberately — it is the single highest-leverage piece of ML observability infrastructure, and it's a pure data-engineering problem you already know how to solve.

---

## 3. Model rollouts as canary deployments

### 3.1 Why a model rollout is *not* a binary rollout

The analogy is useful but breaks in five specific places. Naming the breaks is the interesting part of the answer.

1. **The code doesn't change.** Same binary, different weights file. Your entire release-engineering apparatus — build provenance, code review, diff inspection — has nothing to inspect. The artifact is opaque. Reviewability is near zero, so **validation must be empirical, not analytical**.
2. **Failure is graded, not binary.** A bad binary crashes. A bad model returns plausible, well-formed, slightly-worse answers. There's no crash loop to page on.
3. **The signal arrives late.** Latency regressions show in seconds. Quality regressions show in days. Your canary bake time is bounded by the *slowest* signal you actually care about, which means either long bakes or accepting risk.
4. **Feedback loops.** The model's predictions influence the data used to train the next model. A bad model in canary for 24h can poison the training set. Binary rollbacks don't have this property — **model rollbacks are not clean**.
5. **Statistical power.** Detecting a 0.5% conversion regression may need millions of impressions. At 1% canary traffic that could be a week. The canary percentage is bound by statistics, not by blast-radius appetite. This tension — small blast radius vs. adequate sample — is the crux, and interviewers who've done this will probe it.

### 3.2 The progression ladder

```
offline eval → shadow/dark → canary → progressive rollout → full → (retain rollback path)
```

**Offline eval.** Held-out test set, plus slice-level evaluation (per-geo, per-segment, per-device). Aggregate metrics hide segment collapse: overall AUC up 0.3%, but performance on a minority segment down 8%. **Gate on the worst slice, not the mean.** Also run the previous model on the same set as a control — you're comparing models, not scoring one.

**Shadow / dark launch.** New model receives 100% of real production traffic; its outputs are logged and discarded. Zero user risk. This validates:
- Latency and resource profile at real QPS and real payload distribution
- Feature availability in the serving path (catches train/serve skew immediately)
- Prediction distribution vs. incumbent, on identical inputs — this is the highest-signal comparison you'll get, because inputs are perfectly controlled
- Disagreement rate: fraction of requests where old and new models materially differ. Sample and human-review the disagreements. **This is the single best pre-canary check.**

Shadow is cheap in risk and expensive in compute. Do it anyway.

**Canary.** Small traffic slice to the new model. Now decide two things that people get wrong:

*What do you split on?* Random per-request splitting is wrong for anything with a session or a feedback loop — a user seeing recommendations from two models alternately gets an incoherent experience and pollutes your metrics. Split on a **stable hash of user ID**, so assignment is sticky.

*What do you monitor, and in what order?* Three tiers with wildly different detection latencies:

| Tier | Signals | Detects in | Use as |
|---|---|---|---|
| System | p50/p99 latency, error rate, CPU/GPU util, memory, timeout rate | Seconds–minutes | **Automated abort trigger** |
| Model | Score distribution (JSD vs. control), confidence entropy, fallback rate, output length/validity, disagreement rate vs. control | Minutes–hours | Automated abort with wider thresholds |
| Business | CTR, conversion, revenue/session, complaint rate, downstream error rate | Hours–days | **Human-reviewed promotion gate** |

Automate aborts on tiers 1 and 2. Do **not** automate promotion on tier 3 — the confounders will burn you.

**Progressive rollout.** 1% → 5% → 25% → 50% → 100%, with a bake at each step long enough to acquire statistical power for the metric you're gating on. Compute the required sample size in advance; if 1% for 2 hours can't detect the effect size you care about, the step is theatre.

**A/B test vs. canary — don't conflate them.** A canary asks "is this broken?" (safety, short, abort-oriented). An A/B test asks "is this better?" (efficacy, long, powered, statistically designed). Same infrastructure, different questions, different durations, different decision rules. Interviewers like this distinction.

### 3.3 Rollback is harder than it looks

- **Feature-store coupling.** The new model may have required new features whose computation was also newly deployed. Rolling back the model without rolling back the feature pipeline leaves the old model consuming features that didn't exist when it was trained.
- **Caches and precomputation.** Precomputed recommendations or embeddings from the bad model persist after rollback. You need cache invalidation as an explicit rollback step.
- **Training data poisoning.** Logged interactions during the bad model's exposure feed the next training run. Rollback must include **tainting or excluding that window** from the training corpus. Almost nobody builds this until after it bites them once.
- **Downstream consumers.** If another service consumed the bad predictions and stored derived state, model rollback doesn't undo that. Same class of problem as data-corruption incidents — see SRE book Ch. 26, Data Integrity.
- **The intervening-change problem.** Google's 2026 paper names this for code and it applies to models: <cite index="11-1">a simple binary rollback to the last known good version becomes risky when many changes have landed in rapid succession, since rolling back may remove critical fixes introduced in the interim</cite>. Their answer is dynamic configuration and feature flags to disable specific code paths, plus AI-assisted fix-forward, rather than unwinding everything. For models, the equivalent is a **serving-layer config flag that pins model version per traffic slice** — rollback becomes a config push, not a redeploy.

**Design implication:** model version must be a *runtime configuration parameter*, not a build-time constant. Rollback should be a config change taking effect in seconds. If rolling back a model requires a rebuild and redeploy, your MTTR for model incidents is your deploy pipeline latency, and that's usually 20+ minutes.

### 3.4 The freshness/stability tension

Fresher models track the world better but get less validation. Staler models are better-tested but drift further from reality. This is **exactly the SRE velocity-vs-reliability tradeoff**, and the resolution is the same: **error budget**. If the model-quality SLO has budget remaining, ship faster and retrain more aggressively. If it's exhausted, freeze retraining, lengthen bake times, invest in validation.

Say it that way in an interview. It converts an ML question into an SRE answer, which is what they want to hear from you.

---

## 4. Silent failures in ML pipelines

### 4.1 Definition

**A silent failure is one where every monitored signal is green and the output is wrong.** No error rate, no latency spike, no saturation, no crash. The golden signals are structurally blind to it. This is the single most important concept in this document.

### 4.2 Catalog — memorize four or five of these as concrete examples

| # | Failure | Mechanism | What you'd see | Detection |
|---|---|---|---|---|
| 1 | **Feature defaults to zero** | Upstream schema change; feature lookup fails; code path fills `0.0` instead of erroring | Nothing. 200s, normal latency | Null/default-fill rate SLI; **never silently impute in serving** |
| 2 | **Train/serve skew** | Training features computed in Spark/SQL, serving features in Go/Java; rounding, tokenization, or timezone differs | Model performs worse in prod than offline eval, permanently | Shadow-replay parity check: recompute serving features offline, assert bit-equality |
| 3 | **Upstream join drops rows** | A LEFT JOIN becomes an INNER JOIN; 8% of records vanish | Training set silently smaller; model biased toward the surviving population | Row-count assertions at every pipeline stage vs. expected ratio |
| 4 | **Stale features / frozen pipeline** | Feature-refresh job fails; serving keeps reading last-good values | Predictions still confident, increasingly wrong | Freshness SLI on max timestamp of the feature table — **the highest-value single ML monitor** |
| 5 | **Label leakage** | A feature encodes the target (e.g. `days_since_last_payment` when predicting default) | Offline AUC 0.99, production performance collapses | Suspiciously high offline metrics; adversarial feature review; temporal split validation |
| 6 | **Silent fallback model** | Primary model fails to load; serving falls back to a heuristic or an old version, logs at DEBUG | Latency actually *improves*. Error rate zero | Emit and alert on `model_version` as a first-class metric; alert on unexpected version distribution |
| 7 | **Truncation** | Input longer than max sequence length; silently truncated | Long inputs get systematically worse predictions | Distribution of input lengths vs. truncation rate as an SLI |
| 8 | **Timezone / DST** | Training in UTC, serving in local time | Cyclical features off by hours; degradation with daily periodicity | Time-of-day slice monitoring |
| 9 | **Feedback loop collapse** | Recommender only shows items it already ranks highly; training data narrows to its own outputs; diversity collapses | Engagement metrics may look *fine* or improve short-term | Catalog-coverage / diversity metrics; mandated exploration traffic |
| 10 | **Quantization / hardware drift** | Model quantized for serving; different GPU or kernel version changes numerics | Slight, uniform accuracy loss | Golden-set assertion: fixed inputs → expected outputs within tolerance, run in CI *and* in prod |
| 11 | **Retraining on polluted data** | Bad model's outputs became training labels | Progressive degradation over multiple retrain cycles | Model lineage tracking; training-data provenance |
| 12 | **Segment collapse** | Aggregate metric flat, one segment collapses | Nothing at aggregate level | Slice-level SLOs on protected/critical segments |

Items 1, 4, 6, and 12 are the ones to have ready as verbal examples. They're concrete, mechanically simple, and each maps to a monitor you can describe in one sentence.

### 4.3 The defense stack

**Data contracts / validation gates.** Every pipeline stage declares an expected schema: types, ranges, allowed categories, null tolerance, expected row-count ratio. Violation **fails the stage loudly** rather than degrading. Tooling: TFDV (TensorFlow Data Validation), Great Expectations, Deequ, Pandera. The principle is the one you already apply to config management — **declare intent, verify against it, fail closed**.

**Fail closed, never impute in serving.** If a feature is unavailable at serving time, return an error or an explicit degraded response. Do not silently fill a default. This converts silent failure #1 into a visible error-rate spike, which your existing golden-signal monitoring already catches. **This is the single highest-leverage design decision on the list**, because it moves an entire class of failure from layer 3 (days) to layer 1 (seconds).

**Golden-set canary queries in production.** A fixed set of inputs with known expected outputs, executed against the live serving path on a schedule. Assert outputs within tolerance. Catches quantization drift, wrong model loaded, feature-path breakage, silent fallback. This is a black-box prober, exactly like the HTTP probers you already run — the only difference is the assertion is a distance rather than a status code. **Cheapest high-value control on the list.**

**Model version as a first-class emitted metric.** `model_version` on every prediction, exported as a label. Alert on unexpected version distribution. Catches #6, and gives you exact attribution during rollout incidents.

**Shadow-replay parity.** Periodically recompute serving features through the offline training path and assert equality. The only reliable defense against train/serve skew.

**Slice-level SLOs.** Define the critical segments in advance (geography, customer tier, device class, protected classes) and hold objectives per slice, not just in aggregate.

**Layered evaluation, not a single number.** Aggregate metric → slice metrics → calibration (are predicted probabilities actually calibrated?) → business metric. A model can improve on aggregate AUC while degrading calibration, which breaks any downstream system that thresholds on the score.

### 4.4 Extending the golden signals

Useful table to have memorized — it's the most likely whiteboard artifact from this topic.

| Golden signal | Conventional service | ML service extension |
|---|---|---|
| **Latency** | Request duration | + feature-fetch latency, model-load time, batch queuing delay, **data freshness lag** |
| **Traffic** | QPS | + input distribution shape (not just volume), per-segment QPS mix |
| **Errors** | 5xx rate | + default-fill rate, fallback-model rate, prediction-validity rate, **and the acknowledgement that most ML errors never produce an error code** |
| **Saturation** | CPU/mem/disk | + GPU/TPU utilization and memory, batch queue depth, feature-store read capacity, **training-cluster contention** |
| *(new)* **Freshness** | — | Age of newest data reaching an inference |
| *(new)* **Completeness** | — | Fraction of features resolved without imputation |
| *(new)* **Quality** | — | Delayed-ground-truth metric, on the slowest loop |

The last three are the actual answer. Golden signals were designed for a world where a correct response and a fast response were the same thing.

---

# PART B — AI Applied to SRE (the sre.google 2026 paper)

This is where Google's public material actually sits right now, and it's the half more likely to come up in conversation with a Google interviewer, because it describes what their own org is doing.

## 5. What the paper argues

The framing premise: <cite index="11-1">AI coding assistants are accelerating code generation and deployment velocity, with organizations targeting up to 4x productivity increases, and human code review cannot scale linearly with machine-generated code volume</cite>. The response is not "use AI to automate existing toil" but to build a new control-plane foundation for reliability.

### 5.1 The autonomy ladder — memorize this

<cite index="11-1">Five levels defined by which of Monitor / Investigate / Mitigate / Actuate / Self-Direct are automated</cite>:

| Level | Monitor | Investigate | Mitigate | Actuate | Self-direct |
|---|---|---|---|---|---|
| L0 Manual | auto | human | human | human | human |
| L1 Assisted | auto | auto | human | human | human |
| L2 Partial | auto | auto | human | auto | human |
| L3 High | auto | auto | auto | auto | human |
| L4 Full | auto | auto | auto | auto | auto |

<cite index="11-1">The L2→L3 transition is the critical one, gated on establishing trust and robust safety controls for the system to act without real-time human approval</cite>. Note the collision with your own L4/L5 leveling vocabulary — don't get tangled in an interview.

### 5.2 The Safety Trifecta

<cite index="11-1">Transparency (agents log their chain of thought — signals used, hypotheses considered, reasons for the chosen action, confidence level); real-time risk evaluation (every proposed action is risk-assessed against current production context — ongoing deployments, error budget status, active incidents, time of day); and progressive authorization (agents start at low autonomy and scale up)</cite>. The risk-evaluation point is the sharp one: <cite index="11-1">draining a cell might be low-risk normally and high-risk during a regional peak</cite>. Risk is contextual, not a property of the action.

### 5.3 Architectural guardrails

<cite index="11-1">No ambient access and least privilege — agent identities distinct from human users, strongly authenticated, granted access on demand; agentic circuit breakers with agent-specific rate limits to prevent runaway loops, with every agent action highly interruptible; mandatory dry-run support so blast radius can be predicted before state is mutated; and zero-trust safe-by-default actuation, where the underlying tools are incapable of single-handedly taking down production regardless of who calls them</cite>.

That last principle is worth internalizing independent of AI: **the tool should be safe regardless of the caller's identity or intent.** It's the same argument for why `rm -rf` guards and confirmation prompts belong in the tool, not in the runbook — directly relevant to your Barclays change-management context.

### 5.4 Concrete systems and measured results

- **Detectr** — <cite index="11-1">a Gemini-powered platform that aggregates user feedback from social media, support, and product forums through a filter → cluster → de-noise → report pipeline, acting as a backstop to metric-based monitoring for novel issues</cite>. The insight worth stealing: <cite index="11-1">statistical anomalies in system metrics don't always equate to user impact, because these signals lack understanding of user intent — a feature launch or traffic shift can be misread as failure</cite>.
- **AI Alert** — read-only enrichment intercepting alerts before humans see them, <cite index="11-1">operating within roughly a two-minute budget, querying monitoring, logs, change logs, and dependency graphs in parallel, and presenting verifiable facts with links back to source rather than speculative conclusions</cite>.
- **Incident Hypothesis** — <cite index="11-1">measured via A/B testing at Google's scale to deliver a 10% reduction in Mean Time to Mitigate from informational assistance alone</cite>.
- **Investigation Dashboards** — <cite index="11-1">ML-based anomaly detection increased overall findings by 195% and delivered roughly a 44% reduction in MTTM for supported incidents</cite>.
- **AI Operator + Actuation Agent** — the reasoning engine is deliberately decoupled from the execution engine. <cite index="11-1">The actuation control plane runs pre-flight validations including mandatory dry-runs and justification verification, can automatically downgrade an L3 request to L2 when it detects elevated risk, and provides emergency "red button" endpoints to pause all in-flight agentic actions or globally revoke L3 permissions</cite>.
- **Evaluation data tiering** — <cite index="11-1">Bronze (heuristically autolabeled), Silver (programmatically generated but calibrated against Gold), Gold (human-verified), with stratified sampling to continuously produce Gold data and mathematically calibrate Silver, distinguishing true precision from observed precision</cite>. Golden labels are harvested inline: <cite index="11-1">when an oncaller declares an incident mitigated, the system proposes structured suggestions of the mitigations applied, and accepting or modifying them feeds Gold labels back without extra overhead</cite>.

### 5.5 The role shift

<cite index="11-1">As AI takes over low-level mitigation, SREs move up the abstraction ladder from incident responders to architects of AI safety — defining guardrails, curating golden evaluation data, and governing agent behavior</cite>. And on release: <cite index="11-1">traditional soak times and standard canarying become bottlenecks, requiring adaptive progressive rollouts with automated continuous production validation operating at machine speed — including for data-producing pipelines where anomalies must be caught before propagating downstream</cite>.

Note the loop closing: Part B's answer to high-velocity rollouts is Part A's problem — automated statistical validation of pipeline outputs.

---

## 6. Interview usage

### 6.1 Where to deploy this

- **"How do you stay current?"** — one paragraph, name the paper and one measured result (the 44% MTTM number, or the L2/L3 autonomy gate). Do not lecture.
- **Reverse questions at end of loop** — "Where is your team on the SRE autonomy ladder for your own mitigations?" This is a genuinely good question and signals you read the primary source.
- **NALSD, if an ML component appears in the design** — the freshness SLO and the fail-closed feature-serving decision are the two points to raise. Don't volunteer more.
- **Meta PE / fintech loops** — model-serving reliability is closer to a real job function; PSI in particular is table stakes vocabulary at Goldman/JPMC given model-risk governance.

### 6.2 Three likely questions and answer skeletons

**Q: "How would you monitor an ML service in production?"**
Three layers. Infra layer: golden signals, unchanged, seconds. Data layer: freshness, completeness, distribution stability — SLOs, minutes to hours. Model layer: delayed ground truth, days to weeks. Then the key point: **the golden signals are structurally blind to the failure mode that matters most, which is a fast, well-formed, wrong answer.** Then give one concrete silent failure — the feature-defaults-to-zero one — and the design fix, which is fail closed rather than impute.

**Q: "How would you safely roll out a new model?"**
Shadow first at 100% traffic with outputs discarded, measuring latency, feature-path parity, and disagreement rate against the incumbent on identical inputs. Then sticky-hash canary. Automate abort on system and model-behaviour signals; keep business-metric promotion human-gated because of confounders. Then name the two things that make it different from a binary rollout: statistical power bounds how small the canary can be, and rollback isn't clean because of caches and training-data pollution. Close on model version being a runtime config parameter so rollback is a config push.

**Q: "What's the difference between drift and skew?"**
Drift is the world changing — a condition to manage with SLOs and error budgets. Skew is your own training and serving paths computing features inconsistently — a bug with a root cause and a fix. Conflating them means you go looking for a retrain when what you needed was a diff. Detect skew with shadow-replay parity checks; manage drift with burn-rate alerting on a model-quality SLO.

### 6.3 Where this fits your narrative

You have 60,000+ nodes and a MIM background. The honest positioning is: *"I haven't operated ML serving in production. What I have operated is large-scale pipeline and platform reliability, and the ML-specific parts are a superset — freshness SLOs, data contracts, progressive rollout, blast-radius control are all things I've done; the delta is a quality signal with a multi-day feedback loop and no error code."*

**Do not overstate.** Claiming ML ops experience you don't have is a fast way to get taken apart in a follow-up. Position it as adjacent literacy plus transferable practice, which is exactly what it is.

---

## 7. Reading list — prioritized

### Essential (free, ~2 hours total, do this tonight)

| Resource | URL | Why |
|---|---|---|
| An SRE's guide to optimizing ML systems with MLOps pipelines | `cloud.google.com/blog/products/devops-sre/applying-sre-principles-to-your-mlops-pipelines` | The actual source for the ML half. Short. Read first. |
| AI in SRE: Engineering the Future of Reliable Operations | `sre.google/resources/practices-and-processes/ai-engineering-reliable-operations/` | The 2026 paper. Autonomy levels + Safety Trifecta. ~30 min. |
| SRE Workbook Ch. 13 — Data Processing Pipelines | `sre.google/workbook/data-processing/` | The foundation the ML material builds on. If you read one thing, this. |
| SRE Workbook Ch. 2 — Implementing SLOs | `sre.google/workbook/implementing-slos/` | You've likely read it; reread with freshness-as-SLI in mind. |
| SRE Book Ch. 26 — Data Integrity | `sre.google/sre-book/data-integrity/` | Directly applicable to silent-failure and rollback-pollution reasoning. |

### Supplementary (free, only if the topic keeps surfacing)

| Resource | URL | Why |
|---|---|---|
| Reliable Data Processing with Minimal Toil | `sre.google/resources/practices-and-processes/reliable-data-processing-with-minimal-toil/` | Automation and incident response for pipelines. |
| Efficient Machine Learning Inference (O'Reilly, 2021) | `oreilly.com/content/efficient-machine-learning-inference/` | Multi-model serving, cost/latency/utilization tradeoffs. Useful for the capacity-estimation angle. |
| Hidden Technical Debt in Machine Learning Systems (Sculley et al., NeurIPS 2015) | search `sculley hidden technical debt machine learning` | **The canonical paper.** CACE principle, entanglement, feedback loops, glue code. 9 pages. Most-cited ML-ops paper in existence. If you read one *paper*, this. |
| Artificial Intelligence: What Will It Cost You? (Underwood, SREcon EMEA 2023) | `usenix.org/conference/srecon23emea/presentation/underwood` | Specialized-hardware economics. Good for cost/capacity questions. |
| A Brief Guide To Running ML Systems in Production | `googlesre.page.link/reliable-ml` | Contextual metrics for freshness and throughput, input-data quality. |
| Google Rules of ML | `developers.google.com/machine-learning/guides/rules-of-ml` | 43 practical rules. Rules 29–32 are the train/serve skew section. Skim only those. |

### Paid — do not buy any of these before August

| Resource | Cost | Verdict |
|---|---|---|
| *Designing Machine Learning Systems* — Chip Huyen (O'Reilly) | ~$50 | The best single book on this topic. Ch. 8 (data distribution shifts) and Ch. 9 (continual learning, test in production) are exactly your subtopics. **Buy only if you pivot toward ML-infra roles.** |
| *Reliable Machine Learning* — Chen, Murphy, Underwood, Zinkevich, Hidalgo (O'Reilly) | ~$55 | Google SRE authors, explicitly SRE-lens on ML. Closest match to this topic. Still supplementary for your loop. |
| *Observability Engineering* — Majors et al. | ~$60 | Already on your Tier-1 project reading list. Not ML-specific. |

**Recommendation: read the five free essentials plus the Sculley paper. That's it. Skip every book on this list until after Week 10.**

---

## 8. What this doesn't cover, and shouldn't

Deliberately out of scope for a one-evening literacy block: LLM-specific reliability (eval harnesses, hallucination rates, prompt-injection defense, token-cost SLOs), GPU fleet management and scheduling, vector-database operations, federated learning, and model interpretability tooling. Each is a real specialization. None of them is going to be scored in an SRE-SysEng loop.

**Close the tab after this. Your next highest-value action is a timed live-debugging rep, not more reading.**

---

*Suggested location: `sre-prep-notes/concepts/ai-sre-reliability.md`. This is not a system-design topic, so it does not take an `NN.M-` prefix under `system-design/`.*
