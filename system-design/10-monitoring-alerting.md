# Monitoring & Alerting System — Comprehensive NALSD Reference

*Tenth in the system-design set. Directly relevant to your own MIM/on-call background and your planned post-prep Tier 1 project (SLO Error Budget Service) — this doc is worth reading with that project in mind, not just as an abstract exercise.*

*Callbacks throughout: [07-email-system](./07-email-system.md) and [09-video-platform](./09-video-platform.md) (bytes-vs-count as the ingestion constraint — checked here, not assumed), [05-unique-id-generator](./05-unique-id-generator.md) (cardinality explosion turns out to share its exact bottleneck shape), every storage-heavy doc in this series (range-partition-by-natural-key, recurring a fifth time).*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- **Collect metrics** (counters, gauges, histograms) from every instrumented host/service, continuously.
- **Store as time series**, queryable over arbitrary time ranges with aggregation (sum, average, percentile).
- **Evaluate alert rules** against that data and fire notifications when a condition breaches.
- **Serve dashboards** (Grafana-style) via the same query interface.

### Non-functional requirements
- **Extremely high write throughput** — every monitored target emits continuously, not on-demand.
- **Alerting must be timely** — a monitoring system slow to notice a breach has failed at its actual job, independent of how well it stores data.
- **The monitoring system must be more reliable than what it monitors** — worth stating as a first-class requirement, not an afterthought: if the monitoring stack shares a failure domain with the systems it watches, an outage takes out visibility at exactly the moment it's needed most. This shapes Section 6 directly.

### Scale target
- **100,000 hosts/services**, each exposing **~1,000 distinct time series** (metric × label combinations), scraped/pushed at a **10-second interval**.
```
Active time series: 100,000 × 1,000 = 100,000,000 (100M)
Ingestion rate: 100,000,000 series ÷ 10 sec = 10,000,000 (10M) data points/sec
```

---

## 2. High-Level Architecture

```
Instrumented services ──(pull or push, Section 3)──▶ Ingestion layer ──▶ Time-series DB (Section 4)
                                                                                │
                                              ┌─────────────────────────────────┼─────────────────────────────────┐
                                       Rule evaluator                    Query engine                    Downsampling/retention job
                                     (Section 5)                    (dashboards, ad-hoc queries)              (Section 4)
                                             │
                                       Alertmanager-style pipeline
                                    (dedup, group, silence, route)
                                             │
                                    Notification channels (Slack, PagerDuty, email)
```

---

## 3. Pull vs. Push — The Crux Decision

### Pull (Prometheus model)
A central server maintains a target list (often via service discovery — Kubernetes, Consul) and periodically `GET`s each target's `/metrics` endpoint on a fixed interval, parsing a simple `metric_name{labels} value` exposition format.

**The property worth stating precisely, not just as a vague advantage**: a failed scrape is an unambiguous, immediate signal — the server *knows*, definitively, which targets it did and didn't successfully reach last round. **This is structurally impossible for a pure push system to replicate without an extra mechanism** (a heartbeat metric), because push is inherently silence-tolerant — a push-based client simply not sending anything looks identical to "client is down" and "client has nothing to report right now."

**Natural backpressure, for free**: the collector controls its own scrape pace — it can't be overwhelmed by a client suddenly emitting 100x its normal volume, because it's the one initiating each request. Worth contrasting directly with doc 02's rate limiter: **pull gets this protection structurally, with no explicit rate-limiting component required**, unlike push (below), which genuinely needs one.

**Real limitations**: doesn't trivially centralize at 100K-host scale (needs its own sharding — Prometheus federation, multiple independent shard servers each owning a subset of targets); requires the collector to have network reachability *to* every target, which is awkward across NAT/firewalled/serverless boundaries; and **structurally misses short-lived jobs** — a batch job that runs and exits in 5 seconds can complete before any scrape interval catches it. Prometheus's own ecosystem addresses this gap with a **Pushgateway** — a push-based escape hatch bolted onto an otherwise pull-based system, for exactly the case pull can't handle natively. Worth naming this honestly as a hybrid necessity, not a clean pull-only story.

### Push (StatsD model)
Each host/service actively sends metric updates to a central collector as they occur — commonly over UDP, for low overhead and fire-and-forget semantics.

**Pros**: naturally handles ephemeral/short-lived workloads (no scrape window to miss — the metric is sent the instant it happens); clients need only outbound connectivity, friendlier to NAT'd or serverless environments; can achieve lower latency for time-critical metrics than waiting for the next scrape.

**Cons, stated precisely**:
- **The collector genuinely needs its own explicit protective rate-limiting** — a direct, real instance of doc 02's rate limiter design, unlike pull's structural immunity above.
- **Silence is ambiguous**, as already noted — a real, structural weakness relative to pull, not fixable without adding a heartbeat metric on top.
- **UDP means real, expected data loss under congestion** — a deliberate trade, not an oversight: StatsD's design philosophy explicitly accepts occasional loss in exchange for instrumentation that never blocks or slows the calling application code. Worth stating as a first-class design principle (emitting a metric should never be on the critical path of the thing being measured), not a limitation to apologize for.

### Decision
**Pull is the better default** for this design's scale and Kubernetes-native context, specifically for the "absence is a signal" and free-backpressure properties. **Push remains genuinely better for ephemeral/serverless jobs** and low-overhead fire-and-forget instrumentation where occasional loss is acceptable. Production systems commonly run **both** — pull as primary, Pushgateway-style bridging for the ephemeral-job gap — the same "combine techniques rather than force a single winner" conclusion doc 08 reached for geospatial indexing.

---

## 4. Time-Series Database Design

### Why not reuse doc 03.1's KV store design as-is
Time-series data has structural properties a general-purpose store leaves real efficiency on the table for:
- **Append-mostly, roughly time-ordered** — very different from doc 03.1's arbitrary read/write access pattern.
- **Values within a series usually change slowly and predictably** relative to the prior point (a CPU gauge drifts, it doesn't teleport) — this is what makes aggressive compression possible.

### Compression — delta-of-delta encoding
The widely-cited technique here (from Facebook's Gorilla paper) stores each point as the **difference from the previous value**, and even the **difference of that difference**, rather than a raw timestamp+float64 pair — exploiting exactly the slow-and-predictable-change property above. The commonly cited result: **~1.37 bytes/data point on average**, versus roughly 16 bytes for naive raw storage — an order-of-magnitude reduction that's the entire reason time-series ingestion at this volume is tractable at all (Section 7 quantifies this directly).

### Row/series key design — the fifth recurrence of the same partitioning lesson
Series are keyed by `(metric_name + labels)` as the series identifier, with a time-ordered run of `(timestamp, value)` pairs chunked contiguously for a given series over a time window (e.g., one chunk per 2-hour window). **This is structurally the same range-partition-by-natural-key choice as doc 07's `user_id + reverse_timestamp` row key** — collocating one series' data for efficient range-scan retrieval, the same reasoning that's now shown up for doc 06 (partition by host), doc 07 (partition by user), doc 08 (partition by region), and doc 09 implicitly (partition by video_id for the transcode queue). Worth stating plainly at this point in the series: **this is not a coincidence per topic, it's the dominant partitioning strategy any time the read pattern is "give me one entity's data over a time range," and only consistent hashing (doc 03.1, doc 06's dedup filter) fits when the read pattern is instead "give me one specific record by exact key."**

### Downsampling and retention tiers
Raw 10-second-resolution data is expensive to keep forever. Standard practice: keep full resolution for a short window (24–48 hours, useful for immediate debugging), then progressively downsample into coarser rollups (1-minute averages for 30 days, 1-hour averages for a year) — trading precision for storage cost as data ages. Directly parallel to doc 07's hot/cold storage tiering for email, and conceptually close to doc 04's TTL-driven eviction: data less likely to need fine-grained precision gets compressed rather than deleted outright, since aggregate trend data retains value long after per-second precision stops mattering.

---

## 5. Alerting Pipeline

- **Rule evaluation**: alert rules are queries against the TSDB (e.g., "avg CPU > 90% over 5 minutes"), evaluated periodically (every 30s–1min) by a rule evaluator. A streaming/real-time alternative — evaluating directly against the ingestion stream rather than re-querying the TSDB each cycle — trades implementation complexity for lower alerting latency, the same batch-vs-real-time trade-off doc 07's search-freshness discussion made for a different problem.
- **Deduplication and grouping**: a single root cause can trip the same alert on dozens of hosts simultaneously. Grouping alerts by shared labels (cluster, service) collapses that into **one** notification instead of dozens of pages for one incident — a genuine, important operational UX requirement, not a nice-to-have. Ungrouped alerting is one of the most common real causes of on-call alert fatigue.
- **Silencing and inhibition**: silences suppress known, expected alerts (planned maintenance); inhibition rules suppress a whole class of lower-level alerts (individual "instance down" pages) when a broader, causally related alert (a "cluster down") is already firing — the lower-level alerts are a known, redundant consequence, not independently actionable information.
- **Routing**: a routing tree, not a single flat destination — alerts route to different channels/on-call schedules by label (severity, team, service).
- **Flapping and hysteresis**: a metric oscillating right around a threshold can fire and resolve repeatedly, generating its own alert fatigue. The standard fix — requiring a condition to hold continuously for a minimum duration (a `for: 5m` clause) before actually firing — is the same principle behind burn-rate alerting on SLOs (directly relevant to your own planned Tier 1 SLO Error Budget Service project, and the SRE Workbook's "Alerting on SLOs" chapter already in your reading list): a brief, noisy blip shouldn't page anyone; a sustained breach should.

---

## 6. The Monitoring System's Own Reliability — A Different Kind of Requirement

This is worth its own section because it's a genuinely different *shape* of requirement than anything else in this doc: not a capacity trade-off or an algorithm choice, but a **failure-domain independence requirement**. If metrics ingestion and alerting share infrastructure (region, power, network path) with the primary systems being watched, an outage that takes down the application also blinds the monitoring stack — at precisely the moment visibility matters most. The mitigation is architectural, not algorithmic: run the monitoring/alerting stack in a genuinely separate failure domain, and **monitor the monitoring system itself** ("who watches the watchmen") — a real, sometimes-overlooked meta-requirement that Section 7's cardinality-monitoring mitigation depends on directly.

---

## 7. Capacity Math and the First Real Bottleneck

### Checking the bandwidth-vs-count question explicitly, rather than assuming it
Docs 07 and 09 both found bandwidth, not request count, was the hidden real constraint for large-payload systems. **Worth checking here rather than reflexively applying the same correction**: a single data point is tiny (timestamp + value, compressed to ~1.37 bytes per Section 4), so unlike those two docs, **this system is very likely count-bound, not byte-bound** — the discipline is in verifying that each time, not assuming the same gotcha recurs automatically.
```
10,000,000 data points/sec ÷ 200,000 data points/sec/node
  (a reasonable order-of-magnitude figure for a well-optimized TSDB ingestion node)
= 50 nodes
```

### Storage — small, because of Section 4's compression
```
10,000,000 points/sec × 1.37 bytes/point (Gorilla-style compression)
= 13.7 MB/sec
× 86,400 sec/day
≈ 1.18 TB/day (raw-resolution, compressed)
```
A genuinely manageable number, in sharp contrast to the PB-scale findings in docs 07 and 09 — worth being explicit that this is *because* of the compression technique in Section 4, not despite the enormous point-ingestion rate. Label/series metadata overhead is comparatively small since it's amortized once per series, not once per point.

### The first real bottleneck: cardinality, not throughput or storage
**Every unique combination of metric name + label values is a distinct time series.** A single careless deploy — adding a high-cardinality label (a raw `user_id` or `request_id`) to an already-widely-instrumented counter — can add millions of new distinct series in minutes, not through legitimate load growth but through one label choice.
```
Example: a counter instrumented across 10,000 hosts gets a new label with
a fresh unique value on essentially every request

At ~2 KB/series of in-memory index overhead (a realistic figure for real TSDBs):
10,000,000 new series × 2 KB = 20 GB of unplanned memory pressure,
appearing almost instantly, on top of whatever headroom was actually provisioned
```
**This is structurally the same bottleneck shape as doc 05's machine-ID exhaustion** — not a volume-of-data problem, but a fixed-size-index problem blown through by an unrelated deployment pattern, not by legitimate scale growth. Worth having that parallel ready explicitly: both are "the failure isn't about how much data exists, it's about the size of an index/address space that's supposed to stay small," triggered by a decision made somewhere else entirely (a Kubernetes autoscale event in doc 05, a careless label choice here).

### Iterate
- **Cardinality limits enforced at ingestion** — reject or drop series beyond a configured per-target/per-metric ceiling, rather than accepting unbounded growth.
- **Label-value validation/allowlisting** for known-safe label keys, catching the mistake before it reaches the TSDB at all.
- **Meta-monitoring** — monitor active series count as a first-class metric in this very system (Section 6's "who watches the watchmen," applied concretely), so a cardinality spike triggers an alert before it exhausts capacity, not after.

---

## 8. Further Reading

- **"Gorilla: A Fast, Scalable, In-Memory Time Series Database"** (Facebook) — **free**, **essential** — the direct source of Section 4's compression technique and the ~1.37 bytes/point figure.
- **Prometheus documentation on cardinality best practices** — **free**, **essential**, directly actionable for Section 7's mitigation strategies.
- **Google SRE Workbook, "Alerting on SLOs"** (already in your reading list) — **free**, **essential** — direct connection to Section 5's flapping/hysteresis discussion and your planned Tier 1 SLO Error Budget Service project.

---

## 9. Summary

| Dimension | Value | Binding constraint? |
|---|---|---|
| Ingestion nodes | 50 | Count-bound, not byte-bound — checked explicitly, not assumed |
| Storage | ~1.18 TB/day | No — small, thanks to Gorilla-style compression |
| **Cardinality** | **~2KB/series index overhead, unbounded without limits** | **Yes — the actual first bottleneck, same shape as doc 05's address-space exhaustion** |

## 10. Follow-up Questions

- **"How does this connect to your own SLO Error Budget Service project?"** — directly: Section 5's flapping/hysteresis mitigation *is* burn-rate alerting in miniature — the SRE Workbook's chapter on this is the natural next reading step, and the alert-grouping/routing mechanics in this doc are exactly the pipeline that project's error-budget alerts would need to flow through.
- **"What's the difference between monitoring and observability?"** — monitoring (this doc) answers pre-defined questions via metrics and alerts; observability more broadly includes traces and logs for answering *novel* questions about a system's internal state you didn't anticipate needing to ask in advance — worth naming as a real distinction if pushed on it, not just a buzzword swap.
- **"How would you detect a cardinality explosion before it takes down ingestion?"** — Section 7's meta-monitoring answer directly, but worth adding: rate-of-change on active series count is a better signal than an absolute threshold, since a legitimate slow rollout adding new (bounded) series looks very different from a runaway label generating unbounded new series every second.
