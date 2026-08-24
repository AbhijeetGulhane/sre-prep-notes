# Design Decisions — SLO Error Budget Service

This service reads request metrics from Prometheus (currently `search-api`'s),
computes multi-window error budgets and burn rates against declared SLOs, exposes
them as derived Prometheus metrics plus a JSON API, and drives multiwindow
multi-burn-rate alerts.

**Core stance, stated up front:** Prometheus stays the source of truth for raw
event counts. This service is a *compute + control plane* over Prometheus, not a
parallel data store. Every design decision below follows from that.

This document is the deep-dive (level 2 of the three-level docs discipline). The
one-paragraph README summary is at the end; inline comments (level 3) land when
the ingest/compute code is written.

---

## 1. SLI/SLO chosen

**Decision:** Two request-based SLIs on `search-api` — availability and latency —
each expressed as a `good events / valid events` ratio. SLOs: **99.9% availability**
and **99% of requests ≤ 300 ms**, both over a **30-day rolling window**.

### Why request-based, not windows-based

The SRE Workbook (Ch. 2) splits SLIs into request-based (count good vs valid
*events*) and windows-based (count good vs bad *time slices*). Request-based
composes directly with the counters `search-api` already emits
(`search_api_requests_total`, `search_api_request_latency_seconds_bucket`) and
makes the error-budget math a plain ratio. Windows-based fits things measured
per-time-slice ("was the batch job healthy this minute") — not this service.

### Availability SLI

- **good** = requests with HTTP status `< 500`
- **valid** = all requests
- **SLI** = good / valid

```promql
sum(rate(search_api_requests_total{status!~"5..", namespace="search-sre"}[{window}]))
/
sum(rate(search_api_requests_total{namespace="search-sre"}[{window}]))
```

**4xx decision:** counted as *good* — the service correctly rejected a malformed
request, which is a success of the service. Alternative considered: exclude 4xx
from `valid` entirely (neither success nor failure). Rejected for v1 because it
complicates the denominator and would hide overload-driven 429s, which *are* our
problem. **Gap:** revisit if ordinary 4xx noise starts distorting the budget.

**SLO: 99.9% over 30 days.** Chosen to match the existing `search-api` chaos-test
baseline — that test measured 0.08% budget consumed for a single-pod recovery,
i.e. comfortably inside a 0.1% (= 99.9%) budget. Keeping the same number means the
new service validates against evidence you already have.

### Latency SLI

- **good** = requests completing in ≤ 300 ms
- **valid** = all requests
- **SLI** = good / valid

```promql
sum(rate(search_api_request_latency_seconds_bucket{le="0.3", namespace="search-sre"}[{window}]))
/
sum(rate(search_api_request_latency_seconds_count{namespace="search-sre"}[{window}]))
```

The 300 ms boundary is a real histogram bucket edge (buckets include 0.2 and 0.3),
so `le="0.3"` counts exactly with no quantile interpolation — this is why those
bucket boundaries were chosen back in search-api Day 4.

**Instrumentation gap, named explicitly:** `search_api_request_latency_seconds` is
labeled by `endpoint` only — there is **no `status` label on the histogram**. So a
status-filtered latency SLI ("fast *and* successful") is *not computable from what
search-api emits today*. Two ways out:

1. **v1 decision (chosen):** measure latency over *all* requests, 5xx included.
   Bias: a 500 that fails fast counts as "fast," slightly inflating the SLI. Small
   in practice because availability is already 99.9%, so ≤ 0.1% of events are 5xx.
2. **Fix in search-api (deferred):** add a `status` label to the histogram. Cost:
   histogram cardinality multiplies by the number of status classes. Do this only
   if the bias above shows up in the data.

**Latency SLO: 99% (not 99.9%).** Tail latency is noisier than availability — GC
pauses and cold-cache blips would burn a 99.9% latency budget on non-incidents.
99% is the honest target for a single-model CPU inference service.

### SLIs deliberately dropped

No correctness SLI (are search results *right*?) and no freshness SLI (the corpus
is static, so freshness is trivially met). Named here so the interview answer to
"what SLIs did you consider and reject?" is ready.

---

## 2. Budget window

**Decision:** 30-day **rolling** window. Error budget = `(1 − SLO) × valid events`
over that window. Alerting is multiwindow multi-burn-rate per SRE Workbook Ch. 5.

### Rolling vs calendar month

Rolling 30-day, not calendar-month. A calendar reset creates a cliff: burn the
whole budget on the 30th and the 1st wipes the slate, hiding a chronic problem;
"budget remaining" also jumps discontinuously at midnight on the 1st. Rolling gives
a continuous, always-fair 30-day view.

**Trade-off, stated:** rolling requires range queries over 30 days of data (heavier
than a since-midnight counter), and budget *returns* as old errors age out of the
window. Returning budget is expected behavior, not a bug — this is documented so
it doesn't get "fixed" later.

### Error-budget math (99.9% / 30 days)

```
allowed error ratio      = 1 − 0.999 = 0.001            (0.1%)
budget (events)          = 0.001 × valid_events(30d)
budget remaining (frac)  = 1 − bad_events(30d) / budget_events
```

### Burn rate

```
burn rate = observed error ratio / allowed error ratio
```

Burn rate 1 exhausts the entire 30-day budget exactly at day 30. Burn rate *N*
exhausts it in `30/N` days.

### Multiwindow multi-burn-rate thresholds (99.9%, 30-day)

| Severity | Budget consumed | Long window | Short window | Burn rate | Error rate to trip |
|----------|-----------------|-------------|--------------|-----------|--------------------|
| Page     | 2%              | 1 h         | 5 m          | 14.4      | 1.44%              |
| Page     | 5%              | 6 h         | 30 m         | 6         | 0.6%               |
| Ticket   | 10%             | 3 d         | 6 h          | 1         | 0.1%               |

Derivation — `burn rate = budget_fraction / (long_window / 720h)`:

```
2%  / (1h  / 720h) = 0.02 / 0.001389 = 14.4
5%  / (6h  / 720h) = 0.05 / 0.008333 = 6
10% / (72h / 720h) = 0.10 / 0.1      = 1
error rate to trip = burn_rate × 0.001
```

### Why long window AND short window

The **long window** gives the alert its meaning (sustained burn). The **short
window** is a "is it still happening right now?" gate that lets the alert clear
quickly once the burn stops — it cuts reset time and reduces flapping. An alert
fires only when *both* windows exceed the threshold. (SRE Workbook Ch. 5, "6.
Multiwindow, Multi-Burn-Rate Alerts.")

### Why page vs ticket

Fast burn (14.4) pages — at that rate you'd exhaust the month in ~2 days, so
someone needs to wake up. Medium burn (6) pages — ~5-day-to-exhaust pace. Slow
burn (1) is a *ticket*, not a page — a real problem, but not a wake-someone-up
problem. This mapping is the answer to "how did you decide what pages?"

### Latency SLO (99%) — same table, different base

Allowed error ratio there is 1% (0.01), so the same burn-rate multipliers apply to
a 10× larger base: trip error rates become 14.4%, 6%, 1%. **Policy decision:** v1
pages only on **availability** burn; latency burn is **ticket-only**. Paging on a
noisy latency tail at 3 a.m. is not worth it until the SLI proves stable.

---

## 3. Data model

**Decision:** three entities, one persistence stance.

- **Raw event counts** → Prometheus. Not duplicated.
- **SLO definitions** → version-controlled YAML (config-as-code).
- **Derived budget / burn-rate series** → re-exposed as this service's own
  Prometheus metrics, so Prometheus is the history store. No bespoke TSDB.
- **Alert firing state** → Prometheus alerting rules + Alertmanager. The service
  generates the rules; it does not run its own alert state machine in v1.

The biggest data-model mistake available here is re-storing the raw good/valid
counts in a second database. Prometheus is already a TSDB with retention and
Grafana integration. This service computes over it. This is how Pyrra and Sloth
work in production (they generate recording/alerting rules rather than hoarding
data) — useful prior art to cite in interview.

### (a) SLO definition — YAML, in git

```yaml
slos:
  - name: search-api-availability
    service: search-api
    sli:
      type: availability
      good:  'sum(rate(search_api_requests_total{status!~"5..",namespace="search-sre"}[{window}]))'
      valid: 'sum(rate(search_api_requests_total{namespace="search-sre"}[{window}]))'
    objective: 0.999
    window: 30d
    alerting:
      - {severity: page,   long: 1h, short: 5m,  burn_rate: 14.4}
      - {severity: page,   long: 6h, short: 30m, burn_rate: 6}
      - {severity: ticket, long: 3d, short: 6h,  burn_rate: 1}
```

**Why config-as-code:** SLOs are declarations that belong in git — reviewable,
diffable, GitOps-ready — not rows mutated at runtime. Alternative considered: a
`slo_definitions` table with a CRUD API. Deferred; it adds a database and an API
surface for something that changes rarely. **Gap:** revisit if non-engineers need
a UI to edit SLOs.

### (b) Derived budget snapshot — exposed to Prometheus, not stored

Computed each evaluation interval and emitted on the service's own `/metrics`:

```
slo_objective{slo,service}                     0.999
slo_sli_ratio{slo,service,window}              0.9993   # good/valid over window
slo_error_budget_remaining_ratio{slo,service}  0.42     # 1 − used/budget
slo_burn_rate{slo,service,window}              2.1      # per configured window
```

**Why re-expose instead of a private TSDB:** Prometheus already handles retention,
downsampling, and Grafana. Emitting derived metrics means the dashboard and history
come for free and stay aligned with the raw data's timeline. Alternative — write
snapshots to SQLite/Postgres — is only justified to keep SLO history *beyond*
Prometheus retention (e.g. quarterly SLA reporting). **Gap:** add that store if/when
long-horizon reporting is a real requirement.

### Internal compute state (transient, per eval loop)

The evaluator holds this tuple, derived fresh each loop and never persisted:

```
BudgetState(
    slo_name, window,
    good_events, valid_events,                      # from Prometheus queries
    sli_value             = good/valid,
    bad_events            = valid − good,
    budget_events         = (1 − objective) × valid_events,
    budget_remaining_frac = 1 − bad_events / budget_events,
    burn_rate_by_window   = {'5m':…, '30m':…, '1h':…, '6h':…, '3d':…},
)
```

Because it's recomputed from Prometheus every loop, it's always consistent with
source and needs **no state recovery on restart**. **Trade-off:** each eval fires
several range queries at Prometheus. As the SLO count grows, precompute the
good/valid rates as **Prometheus recording rules** (push the aggregation into
Prometheus). **Next step:** generate those recording rules from the same YAML.

### (c) Alert rule / state — generated, delegated to Alertmanager

The multiwindow condition compiles from the YAML into a Prometheus rule:

```yaml
alert: SLOFastBurn_search_api_availability
expr: |
  slo_burn_rate{slo="search-api-availability", window="1h"} > 14.4
  and
  slo_burn_rate{slo="search-api-availability", window="5m"} > 14.4
for: 2m
labels: {severity: page}
```

**Why delegate:** re-implementing `for`/hold, dedup, silencing, and routing that
Alertmanager already does is wasted and error-prone. Alternative — in-process
evaluation with direct paging — deferred; only worth it as a self-contained demo
mode that runs without Alertmanager. **Gap:** build that mode only if a
zero-dependency demo is needed.

### Data flow

```
Prometheus (raw counters)
        │  range/instant queries
        ▼
  eval loop  ──►  compute budget + burn rates
        │
        ▼
  derived metrics on /metrics
        │  scraped
        ▼
   Prometheus  ──►  Grafana (dashboard)
        │
        └──►  Alertmanager (generated rules)  ──►  page / ticket
```

---

## Open decisions / gaps (recap)

- **Latency histogram has no `status` label** → v1 measures latency over all
  requests, accepting a ≤ 0.1% "fail-fast counts as fast" bias. Fix in search-api
  only if the bias shows.
- **4xx counted as good** for availability → revisit if 4xx noise distorts budget.
- **SLO definitions in YAML, no CRUD API** → add a DB-backed API only if a UI is
  needed.
- **No long-horizon snapshot store** → add SQLite/Postgres only for SLA reporting
  beyond Prometheus retention.
- **Alerting delegated to Alertmanager** → in-process paging mode is optional.
- **Recompute-every-loop** → move good/valid aggregation into Prometheus recording
  rules as SLO count scales.

---

## Reading

- **SRE Workbook, Ch. 2 — Implementing SLOs** — https://sre.google/workbook/implementing-slos/ — free — **essential** (SLI menu, good/valid formulation, request- vs windows-based).
- **SRE Workbook, Ch. 5 — Alerting on SLOs** — https://sre.google/workbook/alerting-on-slos/ — free — **essential** (the six-step path to multiwindow multi-burn-rate; the exact 14.4 / 6 / 1 table).
- **SRE Book, Ch. 4 — Service Level Objectives** — https://sre.google/sre-book/service-level-objectives/ — free — supplementary (SLI/SLO/SLA foundations).
- **Pyrra** — https://github.com/pyrra-dev/pyrra — free / OSS — supplementary (reference implementation of the rule-generation pattern; prior art to name in interview).
- **Sloth** — https://sloth.dev — free / OSS — supplementary (Prometheus SLO generator; emits multiwindow rules out of the box).

## README summary (level 1 — drop into README.md)

> **SLO Error Budget Service.** Computes real-time error budgets and burn rates for
> `search-api` against declared SLOs (99.9% availability, 99% of requests ≤ 300 ms,
> 30-day rolling window), and drives multiwindow multi-burn-rate alerts. Prometheus
> is the source of truth for raw events; this service reads it, computes budgets,
> re-exposes derived metrics, and generates Prometheus/Alertmanager rules. Design
> rationale — SLI/SLO choices, budget-window math, and data model — in
> [`docs/DESIGN.md`](docs/DESIGN.md).
