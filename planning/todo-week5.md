# SRE Prep — Detailed TODO
**Generated: Wed Jul 15, 2026 (Week 5 Day 3, global day 31/70) · Main progress: 186/369 = 50% — slightly ahead of pace**
State source: live progress.json (updated today 6:51 PM). W1 ✅ 40/40 · W2 35/36 · W3 ✅ 33/33 · W4 ✅ 57/57 · W5 21/47.
Already done today: KV store design + vector clocks + NALSD math pass + **Guide Day 7 chaos test & Postmortem #1** — that's the milestone item and your first STAR-story artifact. Strong day.

---

## 1 · TONIGHT (Wed Jul 15) — remaining
- [ ] 🏢 SDI Ch 5 quick skim (10 min — virtual nodes diagram only)
- [ ] 🏢 SDI Ch 6 quick skim (5 min — compare vs your KV design)
- [ ] 🔄 Revision day 31: **LC 76 Minimum Window Substring (H)** — cold recall
- [ ] ⚡ 15-min spillover clear (both from this week, tiny):
  - [ ] W5 Mon: ByteByteGo TinyURL video (~15 min) — watch before Sunday's URL-shortener mock, it's direct prep
  - [ ] W5 Tue: Rate limiter design vs SDI Ch 4 — list 3 gaps

## 2 · THU Jul 16
- [ ] 🔨 Guide Day 8 (verification pass, ~20 min): curl /metrics from all 3 replicas via port-forward; pytest still green; commit only if drift
- [ ] Morning 1 (6:45–9:15): Design Distributed Cache — cache-aside/write-through/write-back · LRU/LFU/TTL eviction + stampede (SETNX, probabilistic expiry) · warming + thundering herd
- [ ] 📐 NALSD math pass on the cache design — machine count, RAM per node, first bottleneck
- [ ] 🎧 Commute: ByteByteGo — Redis internals (single-thread, AOF/RDB, Sentinel vs Cluster)
- [ ] 🏢 SDI Ch 7 skim (5 min — Snowflake bit layout)
- [ ] Evening: Unique ID Gen 15-min sketch (41+10+12 bit)
- [ ] 🔄 Revision day 32 (auto-shown in app)

## 3 · FRI Jul 17
- [ ] Morning: Web Crawler design — URL frontier, politeness, Bloom-filter dedup
- [ ] 📐 NALSD math pass on crawler (pages/day → fetch QPS → storage/day → machine count)
- [ ] Evening: calligraphy/rest — protect it, Sunday is heavy

## 4 · SAT Jul 18 — 🧳 OUTING, full rest (check the rest item so the day closes)

## 5 · SUN Jul 19 — biggest day of Week 5
- [ ] ⏩ Timed URL Shortener 45 min, no notes (req 5 · estimate 5 · API 5 · DB 10 · flow 10 · scale 10)
- [ ] 🎯 **FIRST TIMED NALSD MOCK** — URL Shortener at Google scale (1B users), voice-recorded, full verbal
- [ ] Debrief 15 min: 3 strengths / 3 gaps
- [ ] 📐 NALSD math pass explicitly in the mock: machine counts, disk math, first numerical bottleneck, iterate
- [ ] 🔨 Guide Day 9: ConfigMap + replicas:3 + envFrom → apply → watch load balancing in logs → commit
- [ ] 📖 SRE Workbook Ch 1 (Why-SRE ammo)
- [ ] 🔄 Buffer: clear anything above that slipped + SDI Ch 11-12 skim
- [ ] ⌨️ Coding Maintenance slot → UPGRADED to the outstanding W2 FULL TIMED MOCK: 2 problems × 45 min, interview format (clarify → plan → code → complexity, out loud). Check off BOTH this slot and the W2 Fri mock item when done.

## 6 · REVISION BACKLOG — 12 unchecked (days 18–31)
Pattern is clear: **the hards are being skipped.** These are exactly the Google-tagged ones.
- [ ] 460 LFU Cache (H) · [ ] 239 Sliding Window Maximum (H) · [ ] 42 Trapping Rain Water (H) · [ ] 76 Min Window Substring (H)
- [ ] 128 Longest Consecutive Seq · [ ] 261 Graph Valid Tree · [ ] 323 Connected Components · [ ] 355 Design Twitter · [ ] 528 Random Pick w/ Weight · [ ] 973 K Closest Points · [ ] 3 Longest Substring w/o Repeat · [ ] 424 Longest Repeating Char Replace
**Plan (DECIDED Jul 15): clear all 12 THIS WEEK.** Schedule: 4 on Thu (day-32 revision slot + evening), 4 on Fri (morning tail + evening pre-calligraphy), 4 in Sun buffer. Hards first while fresh: 460, 239, 42, 76 on Thu–Fri. Sat stays a rest day.

## 7 · OLD SKIPPED ITEM — decide, don't ignore
- [x] **DECIDED (Jul 15): option (a)** — W2 FULL TIMED MOCK runs this Sunday in the upgraded coding-maintenance slot (see section 5). First full coding-mock rep lands 7 weeks before it otherwise would have.

## 8 · OFF-PLAN ACTIONS (from master-fix-list — this week)
- [ ] 🔴 Resume E1 critical rewrite (30 min): remove "Senior SRE" title claim → "Production Engineer with 11 years…"; tier skills (cut AWS, cut Distributed Tracing); fix 10→11 years; add GitHub to header. **Before anything else résumé-related.**
- [ ] 🔴 Cut "Nominated for Star Award" (or confirm won)
- [ ] 💳 LeetCode Premium ON (through September)
- [ ] Decide P1: fintech re-theme of blind scenario #6 (prompt E6 — 2-min change, recommend yes)
- [ ] Confirm/deny Amazon in target list (drives P2/E7)
- [ ] Save master-fix-list.md into sre-prep-notes repo

## 9 · WEEK 6 PREVIEW (Jul 20–26) — corrected: outings are FRI–SUN only
✅ **Thu Jul 23 is a full study day** (office Thursday, ~4h). Effective study week = Mon–Thu, 4 days. Outings: Fri Jul 24 (single day) + Sat–Sun Jul 25–26 (weekend).
- Mon: Gmail design + NALSD pass · AI-SRE block · **Guide Day 10 (Prometheus + RBAC deploy)** · SDI Vol 2 Ch 1
- Tue: YouTube design + NALSD pass · **Guide Day 11 (Grafana 3-panel dashboard + screenshot)** · JD research (Google + broadened targets)
- Wed: Monitoring/Alerting design + NALSD pass · **Guide Day 12 (NetworkPolicy + RBAC validation screenshots)** · Workbook Ch 4–5 · JD-skill grid · ⌨️ 2 LC mediums · 📐 Capacity drill #3 (Gmail)
- Thu: **Guide Day 13 — HPA load test** lands on its scheduled day (no cram needed) · Logging Pipeline stays deferred to Wk7 Mon as already planned
- Front-load rule: anything that slips Mon–Thu defers to Wk7 Mon (already the landing zone for NALSD Mock #2 + Logging design). Nothing gets borrowed from Fri–Sun.

## 10 · STANDING WATCHLIST
- Sunday NALSD mock is the week's most important artifact — record it; debrief honestly against Workbook Ch 12 criteria
- Postmortem #1 exists as of today → it becomes STAR Story #1 raw material in Wk8; keep the timestamps
- Hard-problem avoidance (item 6) is the only negative pattern in your data — name it, fix it this week
- W2 mock debt (item 7) — decide by Sunday
