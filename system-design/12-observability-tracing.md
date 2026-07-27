# Observability: Logs + Metrics + Traces — Comprehensive NALSD Reference

*Twelfth in the system-design set. The third pillar — distributed tracing — plus the actual point of "observability" as a concept: correlating all three pillars via a shared identifier. Closes out a trilogy with [10-monitoring-alerting.md](./10-monitoring-alerting.md) (metrics) and [11-logging-pipeline.md](./11-logging-pipeline.md) (logs).*

---

## 1. Requirements & Capacity Estimation

### Functional requirements
- Propagate a unique `trace_id` (and per-hop `span_id`) across every service a single request touches.
- Record timing/metadata for each **span** — a unit of work within one service.
- Assemble spans sharing a `trace_id` into a full trace, visualized as a waterfall/flame graph across service boundaries.
- Correlate traces with logs (doc 11) and metrics (doc 10) via that same identifier.

### Non-functional requirements
- **Tracing overhead on the request path must be negligible** — instrumentation that meaningfully slows production traffic would never survive a production rollout. This single constraint is what makes sampling (Section 4) not just a cost optimization but a hard requirement.
- **Causal ordering must survive clock skew between hosts** — span timestamps are recorded independently on different machines with imperfectly synchronized clocks, and a naive rendering can show a child span starting before its own parent (Section 5) — a genuinely real, only partially solvable problem, not a hypothetical edge case.
- **Coverage, not just correctness, is a requirement** — a trace with gaps because one service in the call path wasn't instrumented is far less useful than an incomplete design suggests (Section 6's honest caveat).

### Scale target
- **500,000 incoming requests/sec**, each fanning out across an average of **10 downstream service hops**.
```
Spans at 100% sampling: 500,000 × 10 = 5,000,000 spans/sec
```

---

## 2. High-Level Architecture

```
Client request ──▶ Service A (creates trace_id, span_id=1)
                        │ propagates trace context (Section 3) on every downstream call
                        ▼
                   Service B (span_id=2, parent=1) ──▶ Service C (span_id=3, parent=2)
                        │
              Each span emitted async, off the critical path (Section 1's overhead requirement)
                        │
                   Local collector/agent ──▶ Kafka (same buffer pattern as doc 11, Section 7)
                        │
                Trace assembly (group by trace_id, link parent/child span_ids)
                        │
                Trace storage (Cassandra or Elasticsearch — doc 03.1 or doc 11's stack, reused directly)
                        │
                   Jaeger/Zipkin UI
```

---

## 3. Trace Context Propagation — the Mechanism That Makes Reconstruction Possible

**The W3C Trace Context standard** (a `traceparent` HTTP header, or the gRPC metadata equivalent) is what carries a request's identity across service boundaries. The mechanism, worked through concretely:

```
Service A: generates trace_id=abc123, creates span_id=1
  → calls Service B, sending traceparent: trace_id=abc123, parent_span_id=1

Service B: receives the header, creates its own span_id=2 as a child of span_id=1
  → if B calls Service C, sends traceparent: trace_id=abc123, parent_span_id=2

Service C: creates span_id=3 as a child of span_id=2
```

**This simple ID-chaining is the entire mechanism** — trace reconstruction later is just: group every span sharing `trace_id=abc123`, then connect each span to its stated `parent_span_id` to rebuild the tree. No coordination service, no central registry needed at request time — the same "structural uniqueness without per-request coordination" instinct as doc 05's Snowflake IDs, applied to a tree relationship instead of a flat namespace.

---

## 4. Sampling — the Central Cost/Completeness Tension

### Head-based sampling
The trace-or-not decision is made **once, at the very start** of a request (the entry-point service flips a weighted coin — say, keep 5% of all requests), and that decision propagates through the entire trace via a flag in the trace context. **Cheap and simple**, but has a real, structural weakness: you can't know in advance which requests will turn out to be the interesting ones — which will error, or run unusually slow — so a low sampling rate can easily miss precisely the traces you'd most want.

### Tail-based sampling
Buffer **all** spans for a trace temporarily, and make the keep/discard decision only after the full trace is available — specifically favoring traces that are slow, contain an error, or are otherwise anomalous. This directly fixes head-based sampling's blind spot — you actually capture the interesting traces, not a random slice — **but the "savings" are smaller than they first appear**: you still pay collection/buffering cost for essentially 100% of spans, just not long-term persistent storage cost for the ones you discard. Worth stating this trade-off precisely rather than presenting tail-based sampling as a free upgrade.

### Decision
Production systems typically run a **hybrid**: a moderate head-based baseline rate (1–10%) for general statistical visibility, combined with an "always keep" override for requests that error or exceed a latency threshold. The same "combine techniques rather than force a single winner" conclusion this series has reached repeatedly (doc 03's ID-generation options, doc 08's geospatial techniques) — the same instinct, again.

**Quantified, worth being precise about which lever does the real work:**
```
At 100% sampling: 5,000,000 spans/sec
At a realistic 5% head-based rate: 5,000,000 × 0.05 = 250,000 spans/sec persisted

The always-keep error/latency override adds back only a small increment on top —
head-based sampling alone accounts for the overwhelming majority of the cost
reduction; tail-based/error-override sampling is a precision refinement on an
already-cheap base, not the primary cost lever.
```

---

## 5. Clock Skew — a Second Manifestation of Doc 05's Problem

Span start/end timestamps are recorded independently on each host's own local clock. Even a few milliseconds of skew — realistic even under well-configured NTP — can make a **child span's recorded start time appear earlier than its parent's**, which is logically nonsensical: a child operation cannot start before the parent that spawned it. **Direct callback to doc 05's clock-rollback discussion** — a different manifestation of the same underlying reality (distributed clocks are never perfectly synchronized), not a coincidence.

**Mitigations, stated honestly rather than as a solved problem**: rely on a span's own internally consistent duration (measured start-to-end on a single host's own clock, always self-consistent regardless of skew relative to other hosts); use the **parent-child structural relationship** as the authoritative ordering signal for visualization rather than trusting raw absolute timestamps across hosts; some systems clamp a child span's displayed start time to never render before its stated parent, while preserving the raw recorded values underneath for deeper analysis. **This is a genuinely unsolved problem in the general case** — the same honest posture as doc 08's "quadtree doesn't actually fix pole distortion" finding, not a claim that this doc has a clean answer.

---

## 6. Correlating Logs, Metrics, and Traces — the Actual Point of "Observability"

**The unifying mechanism**: every log line (doc 11) and, where feasible, every metric emission (doc 10) should carry the current `trace_id`/`span_id` as a field or label. This is what makes cross-pillar navigation possible in practice, not just in principle:
- From a doc 10 alert showing elevated p99 latency for a service, an engineer can query traces for that service in the alert's time window and find concrete slow examples.
- From one specific slow trace, an engineer can jump directly to the exact log lines emitted during that request (querying doc 11's Elasticsearch for `trace_id=X`) instead of manually correlating timestamps across two separate systems by hand.

**This is the actual practical payoff of "observability" as a unified concept** rather than three separately useful but disconnected tools — worth stating as this doc's central thesis directly, not implying it.

**The honest caveat**: this requires `trace_id` to be threaded through *every* layer of logging code, not just the tracing SDK — a real adoption and instrumentation-discipline challenge in practice. A system can have well-implemented tracing *and* well-implemented logging independently and still fail to deliver unified observability if the trace_id doesn't consistently make it into the log context. Worth naming as a people/process gap, not just a technical one.

---

## 7. Capacity Math and the First Real Bottleneck

### Reusing doc 11's pipeline shape directly, not re-deriving it
This is structurally the same Kafka-buffer-then-process pattern as doc 11 — worth stating that explicitly rather than re-deriving from scratch, just with a different per-item size and count:
```
At 100% sampling: 5,000,000 spans/sec × ~200 bytes/span (smaller than a log line,
mostly metadata) = 1 GB/sec

Compare to doc 11's logging pipeline: 15,000,000 lines/sec, 7.5 GB/sec at peak
```
**Tracing at full sampling is cheaper than logging in both dimensions** in this example's assumptions — worth noting as a genuine, useful cross-doc comparison: there are structurally fewer natural "span" boundaries per request than log lines typically emitted within it.

### The real finding: for this system, the numbers aren't where the hardest problem lives
Every prior math-pass doc in this series found its hardest problem *in* the capacity math — a probability calculation (doc 03.3), an address-space ceiling (doc 05), a policy limit (doc 06). **This doc's numbers are all comparatively modest** (Section 4's sampling math keeps volume well within easy infrastructure reach). **The actual hardest problems here — clock skew (Section 5) and instrumentation coverage (Section 6's caveat) — are correctness and adoption problems that no amount of node-count math would ever surface.**

Worth closing the series' math-pass tradition on this point explicitly: **not every system's hardest problem shows up in a capacity calculation.** Recognizing when a design's real risk is a consistency or adoption problem rather than a scaling problem is itself a mature system-design instinct — and it's easy to miss if the habit from nine prior docs is "run the math and find the bottleneck there."

---

## 8. Further Reading

- **W3C Trace Context specification** — **free**, **essential** — the direct source for Section 3's propagation mechanism.
- **OpenTelemetry documentation** — **free**, **essential** — the modern, vendor-neutral instrumentation standard actually used to implement everything in Sections 3–4 in practice.
- **"Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"** (Google) — **free**, **essential** — the foundational paper this entire field is built on, playing the same grounding role here that the Bigtable paper played for doc 07 and the Gorilla paper played for doc 10.

---

## 9. Summary

| Dimension | Value | Binding constraint? |
|---|---|---|
| Spans at 100% sampling | 5M/sec, 1 GB/sec | No — cheaper than doc 11's logging pipeline in both dimensions |
| Spans at realistic 5% sampling | 250,000/sec | No — sampling alone does almost all the cost reduction |
| **Clock skew / causal ordering** | **Not fully solvable, only mitigated** | **Yes, in the correctness sense — but invisible to any capacity calculation** |
| **Instrumentation coverage** | **An adoption problem, not a capacity one** | **Yes, in practice — and also invisible to the math** |

## 10. Follow-up Questions

- **"How would you retrofit tracing onto a system that wasn't built with it from the start?"** — incrementally, starting at the highest-value service boundaries (the ones most implicated in past incidents) rather than attempting full coverage at once — directly acknowledging Section 6's coverage caveat rather than assuming a big-bang rollout is realistic.
- **"What's the actual difference between a span and a log line?"** — a span has a duration and a structural position in a call tree (parent/child); a log line is a single point-in-time event with no inherent duration or tree position — related but genuinely different data shapes, which is why they're stored differently (Section 2) even when unified by the same `trace_id`.
- **"Why not just trace everything at 100% and store it cheaply?"** — Section 1's overhead requirement is about the *production request path*, not just storage cost — even if storage were free, the act of recording and propagating context on every single call has a real, non-zero latency cost that compounds across a 10-hop request chain, which is the deeper reason sampling exists even independent of infrastructure economics.
