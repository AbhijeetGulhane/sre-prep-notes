# slo-budget inventory (fleetguard Week 0)

Snapshot: 2026-10-12. Repo: github.com/AbhijeetGulhane/slo-budget (public), 5 commits, last 2026-08-26.
Language: Python (FastAPI). fleetguard is Go, so "merge" = PORT the math to Go, not a code merge.
Tests: 34 passing (pytest). Code: 915 lines total. Built with AI assistance (Claude).

## 1. What exists

| File | Lines | Role |
|---|---|---|
| app/config.py | 153 | parse_duration; SLO (objective validation, error_budget, window); BurnRatePolicy (long/short windows, threshold, min_request_rate, self-check that budget_consumed_at_threshold matches the SRE Workbook: 14.4x/1h = 2%, 6x/6h = 5%) |
| app/burn_rate.py | 146 | burn_rate, budget_consumed, hours_to_exhaustion; WindowEval, AlertEval (+ reason()); evaluate_policy: both-windows-hot AND rule, 1e-9 relative epsilon at the threshold, low-traffic guard |
| app/engine.py | 127 | SLOEngine: dedupes windows, queries Prometheus, builds SLOReport/BudgetStatus (budget floored at zero) |
| app/prometheus.py | 89 | PrometheusClient: PromQL error-ratio and request-rate queries over HTTP (httpx) |
| app/api.py | 89 | FastAPI: /healthz, /slo/status (JSON), /metrics (hand-written exposition) |
| rules/*.yml | - | Prometheus recording + alerting rules mirroring the service logic |
| tests/test_burn_rate.py | 193 | 26 tests: math, Workbook constants, firing logic, boundaries, low-traffic guard |
| tests/test_engine.py | 107 | 8 tests: report building, fake-transport Prometheus queries |

Gaps found: docs/ empty (DESIGN.md only in sre-prep-notes); two commits share one message (852f694, 57b71a5); no latency SLI, CI, Dockerfile or dashboard; requirements unpinned with pytest in runtime deps.

## 2. Ported to Go (fleetguard gating module, Week 13)

- config.py -> duration parsing, SLO + BurnRatePolicy types with the same validation and Workbook self-check.
- burn_rate.py -> burn_rate, budget_consumed, evaluate_policy (AND rule, epsilon, low-traffic guard), WindowEval/AlertEval incl. human-readable reason (feeds rollout pause/rollback messages).
- tests/test_burn_rate.py -> Go table-driven tests with identical inputs/expected outputs: the Python results are the reference values that prove the port is faithful.
- Optional: hours_to_exhaustion (not needed to gate a rollout; port only if the controller reports budget ETA).

Estimate: ~300 Python lines of logic + 26 test cases.

## 3. Stays behind

- prometheus.py, engine.py orchestration, api.py, rules/*.yml, FastAPI/httpx/uvicorn deps.
- Engine ideas worth re-implementing (not porting): window dedupe; budget floored at zero.

## 4. Open questions for the fleetguard design doc

1. Timescale mismatch: Workbook windows (1h/5m, 6h/30m over 30d) are for paging. A canary wave bakes for minutes, so a 1h long window can't produce a verdict in one wave. Options: rescaled windows; burn-rate over the bake window against an allowed budget fraction; canary-vs-control comparison (SRE Workbook Ch 16).
2. Data source: slo-budget asks a Prometheus server via PromQL. If fleetguard's controller scrapes agents directly (decision 3), it must compute ratios from raw counter deltas itself, including counter resets on agent restart.
3. Which SLI the gate evaluates (decision 4).
4. Low-traffic guard maps to minimum sample size: a 1-node canary yields few events, which is the same statistical problem the guard solves. Define a minimum-events rule for a gate verdict.
5. Epsilon carries over unchanged (Go float64 = IEEE 754 double, same rounding).
6. After the port: archive slo-budget with a README pointer to fleetguard, or keep it as a separate public artifact?

## Defense-bank items (must answer cold)
- Why 14.4x over 1h = 2% of a 30-day budget (14.4 x 1/720).
- Why both windows must be hot (long = significance, short = still happening, gives fast reset).
- Why the 1e-9 epsilon exists (14.4 * 0.001 / 0.001 = 14.3999...).
- Why the low-traffic guard exists and how its floor (~69 requests) was derived.
