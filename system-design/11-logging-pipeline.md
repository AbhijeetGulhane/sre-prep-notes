# Logging Pipeline — Comprehensive NALSD Reference

*Eleventh in the system-design set. Kafka → Flink → Elasticsearch → Kibana. Shares a fleet size with [10-monitoring-alerting.md](./10-monitoring-alerting.md) for direct comparability, and this is the doc where the "check bytes vs. count each time" discipline from doc 10 gets refined further — the answer here is neither, cleanly, it's both, and you have to check which one actually binds.*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- Services emit structured (JSON) log lines.
- The pipeline ingests, buffers, transforms, and indexes them.
- Engineers search/filter logs interactively via a UI (Kibana), by free text and structured fields simultaneously.
- Log-derived signals can feed doc 10's alerting pipeline (e.g., an error-rate-per-service metric computed from the log stream itself).

### Non-functional requirements
- **Durability against transient consumer failure** — a restart of the indexing layer must not lose logs.
- **Near-real-time is enough, true real-time isn't required** — seconds to low minutes of lag is acceptable for debugging use, unlike a genuinely real-time system.
- **Retention/cost management** — logs are voluminous and the overwhelming majority are never read; storage strategy has to reflect that economic reality (Section 5).
- **The pipeline must keep working exactly when log volume spikes** — an incident is precisely when error logging surges *and* when the pipeline is most needed. The same self-referential reliability requirement as doc 10's "the monitor must outlive what it monitors."

### Scale target
Same 100,000-host fleet as doc 10, for direct comparability.
- **50 log lines/sec/host average**, 3x peak factor → **150/sec/host peak**.
```
Average: 100,000 × 50 = 5,000,000 (5M) lines/sec
Peak:    100,000 × 150 = 15,000,000 (15M) lines/sec
```
- **Average log line size: ~500 bytes** (structured JSON, several fields) — meaningfully larger than doc 10's per-metric-point payload, which matters directly in Section 6.

---

## 2. High-Level Architecture

```
App services ──▶ local log agent (Fluentd/Filebeat-style, batches) ──▶ Kafka (durable buffer)
                                                                            │
                                                                      Flink (parse, enrich, filter, aggregate)
                                                                            │
                                                              ┌─────────────┴─────────────┐
                                                        Elasticsearch                Doc 10's alerting pipeline
                                                        (indexed storage)          (log-derived metrics, e.g. error rate)
                                                              │
                                                           Kibana
```

---

## 3. Why Kafka as the Buffer — the Pattern's Sixth Appearance

**Log production is bursty in a way Elasticsearch's sustained indexing throughput isn't built to absorb directly** — and it's bursty at exactly the worst time: an incident is precisely when error-logging volume spikes across many services simultaneously, which is also precisely when the pipeline can least afford to fall over. Kafka sits between the two as a durable, elastic shock absorber, decoupling producers entirely from Elasticsearch's own availability — if Elasticsearch is slow, restarting, or mid-upgrade, Kafka retains the backlog rather than dropping logs or blocking producers.

**Worth naming explicitly at this point in the series**: this is the same accept-fast-durable-queue-process-async pattern as doc 07's MTA and doc 09's transcode queue — the sixth explicit recurrence, and very likely the single most repeated architectural idea across this entire notes set.

### Partitioning — where this pattern actually differs from the last several docs
Kafka partitions here are keyed by **service name** (or a hash of service+host), spreading load evenly — **and it's worth being precise that this is a genuinely different partitioning philosophy than docs 06 through 10's range-by-natural-key pattern**, not a reflexive reapplication of it. Those systems all had a dominant "give me one entity's data as a contiguous range scan" read pattern (one user's inbox, one region's POIs, one series' time range) that justified collocating that entity's data. **Log search doesn't have that same locality benefit** — a Kibana query typically searches *across* many services and hosts by field/free-text filters, not "give me this one host's logs as a range." Hash-based partitioning, spreading load evenly rather than collocating by entity, is the right default here — closer to doc 03.1's KV store logic than doc 07's mailbox logic. Worth stating this distinction directly rather than pattern-matching "partition by natural key" onto every system just because it's applied correctly most of the time.

---

## 4. Why Flink, Not a Simple Pass-Through Consumer

Three real jobs justify a stateful stream processor here rather than a dumb forwarder:
- **Parsing and enrichment**: extracting structured fields from unstructured text (grok-style pattern matching), attaching metadata that wasn't in the original line (datacenter, deployment version, geo-IP lookups), sometimes joining against a separate reference stream.
- **Filtering before the expensive step**: dropping known-low-value logs (routine DEBUG-level noise) *before* they ever reach Elasticsearch — a genuine, quantifiable cost lever, worked out precisely in Section 7.
- **Windowed aggregation as a side output**: computing rolling metrics directly from the log stream (error-rate-per-service-per-minute) and feeding that into doc 10's alerting pipeline — a concrete, real integration point between this doc and doc 10, not just a thematic similarity.

**Worth being honest about why Flink specifically, not just "because it's the standard stack"**: the stateful operations above (windowed aggregation, deduplication, stream joins) genuinely need Flink's checkpointing and state management machinery. If this pipeline only needed dumb pass-through forwarding, a much simpler Kafka Connect sink would do the job at lower operational cost — the same match-the-tool-to-the-actual-requirement instinct as doc 08's "S2 is justified specifically at Google's actual scale, not by default."

---

## 5. Elasticsearch — Why a Different Storage Choice Than Doc 07's Row Store

**Direct callback to doc 07, Section 6**: full-text search alongside structured-field filtering (a specific error string, *and* `service=X AND status_code=500 AND time-range`) is fundamentally an inverted-index problem, not a row-key lookup problem — the same reason doc 07 needed a separate search layer in front of its Bigtable-style store. It's not a coincidence that both docs converge on "you need a dedicated search engine for this access pattern" — it's the same underlying requirement showing up twice.

**Index-per-time-window, a genuinely elegant mechanism worth knowing precisely**: Elasticsearch commonly uses daily indices (`logs-2026.07.27`) — **this is itself a range-partition-by-natural-key move**, partitioning by time, which both enables efficient time-range queries *and* makes retention trivially cheap: dropping an entire old index is vastly cheaper than deleting matching documents one at a time. Direct parallel to doc 07's tiered hot/cold storage, but with a cleaner mechanism specific to how Elasticsearch's index model works.

---

## 6. Capacity Math and the First Real Bottleneck

### Kafka — checking both dimensions, and finding they disagree
```
Bandwidth check: 15,000,000 lines/sec × 500 bytes = 7.5 GB/sec
÷ 0.5 GB/sec/node (a reasonable modern Kafka broker figure)
= 15 nodes (bandwidth-bound estimate)

Message-count check: 15,000,000 lines/sec ÷ 200,000 msgs/sec/node
(a plausible per-broker message-handling ceiling — parsing/batching/network
overhead per message, independent of raw bytes)
= 75 nodes (count-bound estimate)
```
**This is the opposite direction from docs 07 and 09's finding** — there, bandwidth was the hidden binding constraint because payloads were large. Here, with smaller-but-numerous messages, **per-message handling overhead binds tighter than raw bytes: 75 nodes, not 15.** The real lesson, refined from doc 10's "check bytes, don't assume count" instinct: **check both dimensions explicitly and take the max — don't assume either one wins by default.** Large payloads tend to be bandwidth-bound; small, numerous payloads tend to be count-bound; the only way to know which applies to a given system is to compute both.

### Elasticsearch — the dominant cost, and a second confirmation of a doc 07 pattern
```
15,000,000 lines/sec ÷ 20,000 docs/sec/node (indexing is more expensive per-document
than raw ingestion — building inverted-index structures, analyzing text fields)
= 750 nodes
```
**750 nodes for indexing, against Kafka's 75 — a 10x gap, even larger than doc 07's 87-vs-26 finding for email search.** This is now the **second independent system** in this series where full-text-search indexing turns out to be the most expensive layer in the entire pipeline, by a wide margin over the ingestion front door. Worth stating as a generalizable pattern now, not a one-off: **any pipeline that includes full-text search should expect indexing, not ingestion, to dominate its node count.**

### The first real bottleneck, and the fix the design already made
750 nodes just for raw indexing throughput is the real constraint — and the fix is exactly what Section 4's Flink filtering step already justified qualitatively, now quantified:
```
If Flink drops/samples low-value logs (DEBUG-level filtered entirely,
routine INFO logs sampled at ~1-in-100) and this reduces indexed
volume by ~90% (a plausible real-world reduction, since routine logs
vastly outnumber genuinely valuable ones):

750 nodes → 75 nodes
```
**A 10x reduction, and the design decision that produces it was already made in Section 4 for a different stated reason** (cost-consciousness about what's worth indexing) — worth noting explicitly that the math pass didn't need to invent a new fix here, it just quantified the payoff of a decision the design already got right.

---

## 7. Further Reading

- **Kafka documentation on partitioning and broker sizing** — **free**, **essential** — direct source for Section 6's capacity figures.
- **Flink documentation on stateful stream processing and checkpointing** — **free**, **essential** — the mechanism Section 4 leans on to justify Flink over a simpler consumer.
- **Elasticsearch Index Lifecycle Management (ILM) documentation** — **free**, **essential** — directly relevant to Section 5's retention-by-index-drop mechanism.

---

## 8. Summary

| Layer | Figure | Binding constraint? |
|---|---|---|
| Kafka | 75 nodes | **Count-bound, not bandwidth-bound — opposite of docs 07/09, confirming "check both, don't assume"** |
| Elasticsearch (unfiltered) | 750 nodes | Indexing cost — the dominant layer, second confirmation of doc 07's pattern |
| Elasticsearch (after Flink filtering) | 75 nodes | A 10x reduction from a decision the design already made for cost reasons |

## 9. Follow-up Questions

- **"How do you avoid losing logs during a Flink job restart/redeploy?"** — Flink's checkpointing (Section 4) persists processing state, and Kafka's own retention (Section 3) means unprocessed messages simply wait in the topic — the same durability guarantee doc 07's MTA queue provides, applied here.
- **"What if a single service starts logging at 1000x its normal rate (a logging bug, not a real incident)?"** — worth a per-service rate cap at the local agent level, the same instinct as doc 02's rate limiter, applied to protect the shared pipeline from one misbehaving producer rather than an external API consumer.
- **"How does this connect to distributed tracing?"** — deliberately deferred to the next doc in this series, since tracing is a genuinely distinct pillar with its own crux decisions (sampling, context propagation) rather than a variant of this pipeline.
