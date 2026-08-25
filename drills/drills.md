# Rapid-Fire Recall — Drill Question Bank

Daily 5–10 min drill: **5 named-technique + 5 Fermi, interleaved**. Graded on precision — a right-shape-but-imprecise answer (e.g. "version number" instead of "vector clocks") is a **partial miss**, not a pass.

**How to run it:** ask Claude to "drill me." Claude fires 10 mixed questions, grades terse (✅ / correction / partial), and ends with a 2-line summary of what to drill next. Recurring error patterns are tracked in `drill-log.md`.

This file is the source pool. Rotate across topics so all 15 areas cycle over ~2 weeks. Weight toward whatever the last mock flagged weak.

---

## Named-Technique Questions (scenario → precise mechanism)

### Distributed KV Store
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Two concurrent writes conflict, neither causally dominates — what do you store to detect it? | **Vector clocks** | "version number"/"timestamp" — can't distinguish concurrent from causal |
| Distribute keys so adding a node relocates minimal data? | **Consistent hashing** (virtual nodes) | "hash mod N" — remaps nearly everything |
| Guarantee a read observes the latest write in a quorum system? | **W + R > N** | — |
| Replicas reconcile after transient failure, no leader? | **Anti-entropy: Merkle trees + read repair + hinted handoff** | "just re-sync" — name the mechanism |
| Write-heavy store, fast sequential writes, tolerate slower reads? | **LSM-tree** | "B-tree" — read-optimized, write-amplifying |

### Cache
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Evict least-recently-used in O(1)? | **HashMap + doubly-linked list** | — |
| Stop many clients recomputing the same expired key at once? | **Single-flight/lock or probabilistic early expiration** (stampede protection) | "set a TTL" — TTL causes the stampede |
| Evict by access frequency, not recency? | **LFU** (freq counts + buckets) | — |
| App updates cache and DB together, read-through on miss? | **Write-through** | vs cache-aside / write-back — name which |

### Rate Limiter
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Enforce average rate but allow short bursts? | **Token bucket** | "leaky bucket" — forbids bursts |
| Rate-limit across a fleet with shared low-latency state? | **Redis INCR + EXPIRE** (or sorted-set sliding window) | — |

### Unique ID
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| 64-bit time-sortable IDs across nodes, no coordination? | **Snowflake** (timestamp + machine-id + sequence) | "UUIDv4" — not sortable, 128-bit |

### Web Crawler
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Test "seen this URL?" minimal memory, tolerate false positives? | **Bloom filter** | "hash set" — too much memory |
| Crawl broadly without hammering one domain? | **Per-domain politeness queue + rate limit** in URL frontier | — |

### URL Shortener
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Short unique keys without cross-server coordination? | **Base62 of a counter, or per-server key ranges** | "MD5 hash" — collision risk, length |
| 301 vs 302 — which trades analytics for load? | **302** keeps every hit (analytics); **301** cached by browser (saves load) | — |

### Video / YouTube
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Smooth playback over variable bandwidth? | **Adaptive bitrate streaming (HLS/DASH)**, chunked segments | — |
| Count views at scale without a DB write per view? | **Batched/approximate counting (count-min sketch)**, async | — |
| Deliver popular content globally, low latency? | **CDN edge caching** | — |

### Proximity / Geo
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| All points within X km, shardable across nodes? | **Geohash** (or S2 / quadtree) | "lat-long BETWEEN range query" — boundary + sharding issues |

### Monitoring & Alerting
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Know you'll breach the SLO before you do? | **Error-budget burn-rate alerting** (multi-window) | "alert at 80% CPU" — not SLO-based |
| Count unique users approximately over a huge stream? | **HyperLogLog** | "hash set of user IDs" — memory blows up |
| Store high-cardinality time-series efficiently? | **TSDB with rollups/downsampling** | — |

### Distributed Lock Service
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Elect one leader among nodes, tolerate failures? | **Consensus (Raft/Paxos)** or lease from ZooKeeper/etcd | — |
| Stop a paused lock-holder corrupting data after its lease expired? | **Fencing tokens** (monotonic, reject stale) | "re-check the lock" — race still exists |

### Chat System
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Push to online clients in real time? | **Persistent WebSocket + pub/sub fanout** | "poll every second" — not real-time/scalable |
| Deliver to offline users on reconnect? | **Per-user inbox queue / offline store** | — |
| Consistent message order in a group? | **Per-conversation sequence numbers** (or logical clock) | — |

### Ad-Click Aggregation
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Real-time aggregation, exactly-once, late events? | **Stream processor + windowing + watermarks (Flink)**, idempotent writes | — |
| Handle a viral hot key in aggregation? | **Salt/shard the hot key + pre-aggregate** | "add more consumers" — hot key still lands on one |

### Payment / Wallet
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Stop a double-charge when the client retries? | **Idempotency keys** | "check if it exists first" — race window remains |
| Keep balances correct across a transfer? | **Double-entry ledger + atomic transaction** | — |
| Multi-service transaction without 2PC? | **Saga pattern** with compensating actions | — |

### Email / Logging
| Scenario | Precise answer | Partial-miss trap |
|---|---|---|
| Full-text search over billions of emails? | **Inverted index** | — |
| Decouple a log firehose from slow consumers? | **Kafka** (durable log buffer) | "write straight to Elasticsearch" — no backpressure buffer |
| Exactly-once aggregation over the log stream? | **Stream processor with checkpointing (Flink)** | — |

---

## Fermi Questions (10-sec estimate, units forced)

Operation tag flags the div-vs-multiply reflex. Force explicit units on every answer.

| Question | Answer | Operation |
|---|---|---|
| 1B requests/day → QPS? | 1e9 / 86,400 ≈ **11.6K QPS** | DIVIDE daily by ~86.4K |
| Peak = 3× avg, avg 12K QPS → peak? | **~36K QPS** | MULTIPLY |
| 1KB/record × 1B records → storage? | **1 TB** (1e9 × 1e3) | MULTIPLY |
| 500M photos/day × 2MB → daily storage? | **1 PB/day** (5e8 × 2e6) | MULTIPLY |
| 1TB dataset, 100GB/node → nodes? | **10 nodes** | DIVIDE |
| 1M concurrent streams × 5 Mbps → bandwidth? | **5 Tbps** | MULTIPLY |
| Bloom filter, 1B items, 1% FP → size? | **~1.2 GB** (≈9.6 bits/item ÷ 8) | MULTIPLY items×bits ÷8 |
| Cache hit 90%, 10K QPS → DB QPS? | **1K QPS** (10K × 0.10) | MULTIPLY by miss rate |
| Read:write 100:1, 10K QPS → writes/s? | **~99/s** (10K ÷ 101) | DIVIDE |
| 6-char base62 → keyspace vs 1B? | 62^6 ≈ **56.8B** > 1B ✓ | EXPONENT |
| 100 GB/day logs × 365 → yearly? | **~36.5 TB** | MULTIPLY |
| 40GB working set, 16GB RAM/node → nodes? | **≥3 nodes** (round up) | DIVIDE round-up |
| 1M writes/s replicated 3× → total ops/s? | **3M ops/s** | MULTIPLY |
| 10M followers, fanout-on-write → inserts? | **10M inserts** | MULTIPLY (motivates fanout-on-read) |
| QPS 12K, each holds 200ms → concurrency? | 12K × 0.2 = **2,400** | MULTIPLY (Little's Law L=λW) |
| Need 50K QPS, server does 5K → servers? | **10 servers** | DIVIDE |
| 1B users, 20% DAU → daily actives? | **200M DAU** | MULTIPLY by 0.2 |
| 1hr video at 5 Mbps → file size? | 5e6×3600÷8 ≈ **2.25 GB** | MULTIPLY then ÷8 for bytes |
| 10B rows / 1000 shards → rows/shard? | **10M rows/shard** | DIVIDE |
| p99 budget 200ms, 5 sequential calls → per call? | **~40ms each** | DIVIDE |
| 100K read QPS, cache absorbs 95% → DB QPS? | **5K QPS** | MULTIPLY by 0.05 |
| 100K msgs/s × 1KB × 86,400 × 7d → storage? | **~60 TB** | MULTIPLY chain |
| 100M new URLs/month → per second? | **≈38/s** (÷ ~2.6M s/month) | DIVIDE by 2.6M |
| 5KB/message × 1B/day → daily volume? | **5 TB/day** | MULTIPLY |
| 1B events × 8 bytes → counter memory? | **8 GB** | MULTIPLY |
| 1M metrics/s × 2 bytes → ingest bandwidth? | **2 MB/s** | MULTIPLY |
| Storage grows 1 PB/yr, replicated 3× → raw/yr? | **3 PB/yr** | MULTIPLY |
| 10M uploads/day × 500KB → daily ingest? | **5 TB/day** | MULTIPLY |
| 1KB row × 1M rows → cache RAM? | **1 GB** | MULTIPLY |
| 200M DAU × 10 req ÷ 86,400 → QPS? | **~23K QPS** | MULTIPLY then DIVIDE |
