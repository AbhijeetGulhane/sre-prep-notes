# Grafana Concepts, PromQL Golden Signals, Dashboard JSON Model

> Week 4 Thu carry-over, closed out in Week 5. Includes live Helm-installed Prometheus+Grafana lab on minikube, plus a real Docker Desktop backend recovery.

## Grafana core mental model — query-and-render layer, not a data store

**Grafana stores no metrics itself.** It's a visualization/querying frontend over other systems that hold time-series data (Prometheus, InfluxDB, Elasticsearch, CloudWatch, Loki for logs, Tempo for traces). Job: connect to a datasource, run a query, render as chart/table/gauge. Delete Grafana, metrics data is untouched — it lives in Prometheus, Grafana was just the window.

**Troubleshooting implication:** "dashboard shows no data" is almost never a Grafana bug — it's a datasource connectivity problem, a wrong query, or the metric genuinely not existing yet. Triage always starts with "Grafana problem or data problem" — rarely the former.

## Datasource — the connection to where data lives

Configured connection: type (Prometheus, MySQL, etc.), URL, auth. Configured once per backend; every panel/dashboard references it by name.

```
Settings → Data Sources → Add data source → Prometheus
URL: http://prometheus:9090
Access: Server (default)
```

**Server vs Browser access mode:**
- **Server** — Grafana's *backend* calls the datasource; browser only talks to Grafana. Sane default for anything behind network segmentation — one network path to secure (Grafana → Prometheus), not N browsers needing direct access to internal metrics infra.
- **Browser** (legacy) — user's browser calls the datasource URL directly; requires that URL reachable from wherever the browser sits, a real problem in segmented networks.

**Datasource URL in K8s context:** often just another Service (e.g. `prometheus-server.monitoring.svc.cluster.local:9090`) — literally the CoreDNS → ClusterIP → kube-proxy DNAT mechanism from earlier topics; Grafana's datasource is just another consumer of that same path.

## Panel — one query, one visualization

A **panel** = a single chart/table/gauge on a dashboard, with one (or a small handful of) **target** (a query against a chosen datasource) plus viz settings (chart type, axis labels, thresholds, colors, units).

**Core mental model:** one panel = one PromQL query = one rendered visualization over the selected time range. Debugging a bad panel = debugging that one specific query's output in isolation.

**Panel types worth knowing:**
- **Time series** (default/most common) — line/area graph over time
- **Stat** — single big number, sparkline, threshold-based coloring — good for "current error rate" at-a-glance
- **Gauge** — dial-style, good for value against a bounded range (disk usage %)
- **Table** — raw tabular results ("top 10 slowest endpoints right now")
- **Heatmap** — good for histogram/bucket data (latency distributions) as color-intensity grid over time

## Dashboard — collection of panels, shared controls

Named collection of panels in a grid layout, sharing a common **time range** picker (top right, applied to every panel's query simultaneously) and optionally **template variables** (dropdown selectors like "choose a namespace/service" substituted into every panel's query) — makes one dashboard reusable across many services/environments instead of copy-pasting per-service dashboards.

## Auth — anonymous admin (dev) vs production

Local dev: anonymous access + auto-granted Admin role is fine — zero security concern on your own laptop/minikube.
```ini
[auth.anonymous]
enabled = true
org_role = Admin
```

**Production: genuinely dangerous.** Dashboards often expose business-sensitive data (request volumes = business scale, error rates = operational health, sometimes customer-identifying labels). Anonymous Admin = anyone reaching the URL can both view *and* modify/delete dashboards, alerts, datasource configs. Production uses real auth — OAuth/SSO (Google, GitHub, SAML, LDAP) tied to org identity provider, role-restricted (most users Viewer, few Editor/Admin).

## PromQL mental model for the 3 Golden Signal panels

**Rate (traffic):**
```promql
sum(rate(requests_total[1m]))
```
`rate()` on a **counter** — counters only go up (resets on restart), `rate()` computes per-second average increase over the window, correctly handling counter resets.

**Latency p99:**
```promql
histogram_quantile(0.99, rate(latency_bucket[1m]))
```
`histogram_quantile()` on **bucket** metrics (`_bucket` suffix, from a Prometheus histogram type) — approximates a percentile from the bucketed distribution. Never use `rate()` alone for latency — raw latency values aren't meaningfully summed/averaged; percentiles need the bucket structure.

**Errors:**
```promql
sum(rate(requests_total{status=~"5.."}[1m]))
```
Same `rate()` on counter pattern as traffic, filtered by a label regex (`status=~"5.."` matches any 5xx). Often expressed as a ratio against total traffic for a true error *rate* rather than raw count:
```promql
sum(rate(requests_total{status=~"5.."}[1m])) / sum(rate(requests_total[1m]))
```

**The pattern to memorize:** `rate()` on counters, `histogram_quantile()` on buckets. Everything else follows from combining these two with label filters and `sum()`/aggregation.

## Dashboard JSON model

A dashboard is, underneath the UI, just a JSON document. Key structural pieces worth knowing when debugging "why does this panel show No Data":
- **`panels[]`** — array of panel definitions, each with its own `targets[]` (the actual queries), `type` (time series/stat/gauge/etc.), and layout position (`gridPos`)
- **`targets[]`** (inside each panel) — the actual PromQL/query strings sent to the datasource; if a panel shows No Data, this is the first thing to inspect — often the query references a metric/label that doesn't exist or doesn't match anything in the current time range
- **`templating.list[]`** — template variable definitions (the dropdown selectors); a panel query referencing `$variable_name` that isn't defined here, or where the variable's current value doesn't match any real label value, silently returns nothing
- **`time`** — the dashboard's default time range

**Why this matters practically:** exporting/importing dashboards as JSON is the standard way to version-control or share dashboards (checking dashboard JSON into git alongside the app code, e.g. in `search-api`), and understanding the structure is what lets you debug a broken imported dashboard by reading the raw JSON instead of clicking through UI panels one at a time.

## Interview one-liners, compiled

- "Grafana doesn't store data — it's a query-and-render layer over datasources like Prometheus. 'No data' in a panel is almost always a datasource or query problem, not a Grafana bug."
- "Server access mode means Grafana's backend calls the datasource, not the user's browser — the sane default for anything behind network segmentation."
- "One panel is one query rendered one way — debugging a bad panel means debugging that specific PromQL query in isolation."
- "Anonymous admin mode is fine for local dev because you're the only user — in production it's a real security hole, since dashboards often expose business-sensitive operational data."
- "rate() on counters, histogram_quantile() on buckets — that's the entire PromQL pattern behind the three Golden Signal panels."
- "A dashboard is just JSON — panels[] with targets[] holding the actual queries, and templating.list[] for variables. Debugging 'No Data' starts with reading the targets[] query directly."

## Lab: real Prometheus + Grafana on minikube via Helm, live `up` query panel

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Datasource came pre-wired automatically by the Helm chart (no manual config needed) — confirmed Prometheus already listed under Connections → Data sources on first login.

**Built the `up` panel:** Dashboards → New → Add visualization → Prometheus → Code mode → query `up` → Run. Result: one line per scrape target (`alertmanager`, `config-reloader` ×2 instances, `coredns`, `etcd`, `grafana`, etc.), all flat at `1` (healthy). Confirms the full loop live: datasource → PromQL query → panel → dashboard, using real cluster data, not a canned example.

**Note on the graph's start point:** data only appeared partway through the visible time window — not a bug, just because Prometheus's scrape history only exists from the moment the Helm chart was installed; nothing to show before that.

## Real-world infra troubleshooting encountered en route (worth remembering as patterns, not just this session's noise)

**Docker Desktop's WSL2 backend distro (`docker-desktop-data`) went fully missing**, not just stopped — confirmed via `wsl --list --all` showing only `Ubuntu-22.04`, no `docker-desktop`/`docker-desktop-data` at all. Fix: `wsl --unregister docker-desktop` (only the one that still existed), relaunch Docker Desktop, let it rebuild both backend distros from scratch. Data loss caveat: any local images/containers were already gone the moment the data distro vanished — the rebuild doesn't recover them, it just restores a working (empty) backend.

**Newer Docker Desktop versions don't show a `docker-desktop` WSL distro the same way** — this version uses `docker-desktop-user-distro` integrated directly into `Ubuntu-22.04`, so `wsl -l -v` never showing a separate `docker-desktop` entry was a red herring, not a sign of failure. The actual signal that Docker was healthy was in `com.docker.backend.exe.log`: `"Ubuntu-22.04 distro is ready"` and successful real API calls.

**A WSL2 terminal open before Docker's CLI symlinks were created won't see the `docker` command** — closing and reopening the terminal (not just `exec bash`) picks up the newly-created `/usr/bin/docker` symlink. Stale shell PATH/filesystem cache, not a real missing binary.

**minikube's `--memory` flag has a hard floor (1800MB) and a recommended minimum (1900MB)** — going right at the floor caused the API server to crash-loop (`connection reset by peer`, not `connection refused` — the tell that a process is actually dying repeatedly rather than just being slow to start). Fixed by using `--memory=2200mb`, comfortably above the recommended minimum.

---
*Week 4 fully closed out. Currently in Week 5 (NALSD/system design phase).*
