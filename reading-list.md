# Reading List — Mapped to sre-prep-notes Docs

*A curated reading list organized around what's already grounded the system-design and database docs in this repo, plus a few additions not yet cited anywhere. Not tied to a single topic, so it lives at the repo root rather than inside `system-design/`.*

---

## Core books (already owned)

### DDIA (Kleppmann) — the backbone
Cited directly or indirectly across nearly every doc. Chapter map, since "read the whole book" isn't actionable:

| Chapter | Where it applies |
|---|---|
| Ch. 5 (Replication) | `system-design/03.1-distributed-kv-store-nalsd.md`, `system-design/14-distributed-lock-service.md` — quorum vs. consensus |
| Ch. 6 (Partitioning) | `system-design/03.1`, `06`, `07`, `08`, `10`, `11` — the range-vs-hash theme running through the whole series |
| Ch. 7 (Transactions) | `databases/isolation-levels-and-anomalies.md` directly |
| Ch. 8–9 (Trouble with Distributed Systems / Consistency & Consensus) | `system-design/14` — and the three-way consistency-mechanism comparison across docs 03.1, 09, and 14 |

### SDI Vol 1 (Alex Xu)
Already grounded `system-design/01.1-url-shortener-nalsd.md`, `07-email-system.md`, `09-video-platform.md`. **SDI Vol 2 exists** and covers more of what's already been built here: proximity service, monitoring/alerting, and payment systems specifically. Worth a comparison exercise for `08-proximity-service.md` and `10-monitoring-alerting.md`, the same treatment as the `01.3` and `07-email-system-bytebytego-comparison.md` docs.

---

## Essential papers (all free) — grounding specific docs

| Paper | Doc it grounds |
|---|---|
| Dynamo (Amazon) | `03.1-distributed-kv-store-nalsd.md` — the AP/quorum design |
| Bigtable (Google) | `07-email-system.md` — row-key/column-family storage |
| Spanner (Google) | `09-video-platform.md` — TrueTime/external consistency |
| Chubby (Google) | `14-distributed-lock-service.md` — the actual source system |
| Raft, "In Search of an Understandable Consensus Algorithm" | `14-distributed-lock-service.md` — the more approachable Paxos alternative |
| Gorilla (Facebook) | `10-monitoring-alerting.md` — delta-of-delta compression |
| Dapper (Google) | `12-observability-tracing.md` — distributed tracing foundations |
| Mercator | `06-web-crawler.md` — the two-level frontier design |

**Worth reading Dynamo and Chubby back to back specifically** — they're the two papers behind the AP-vs-CP split docs 03.1 and 14 spend real effort contrasting, and reading the originals side by side sharpens that contrast more than any secondary summary can.

---

## Not yet cited anywhere — worth adding

- **Google SRE Book** (sre.google/books, free) — the *Workbook*'s "Alerting on SLOs" chapter is already cited from `10-monitoring-alerting.md`; the original SRE book's monitoring and "eliminating toil" chapters are the direct philosophical grounding for that doc and connect straight to your own MIM background.
- **"Understanding Distributed Systems" by Roberto Vitillo** (paid, ~$10 ebook) — more concise and interview-paced than DDIA; good as a fast, compressed review pass across everything in this series rather than DDIA's textbook depth. Worth it specifically closer to the interview date.
- **raft.github.io** (free, interactive visualization) — not a reading, a tool. Watching leader election and log replication animate live builds intuition for `14-distributed-lock-service.md`'s consensus mechanics faster than another paragraph of explanation would.
- **Martin Kleppmann's blog** (martin.kleppmann.com) — beyond the "How to do distributed locking" post already cited in doc 14, his writing consistently sits at the target depth this whole repo aims for.

---

## Priority order if time-constrained before Aug 10

1. **Raft paper + raft.github.io** — doc 14 is the newest and least battle-tested topic.
2. **DDIA Ch. 6** — the single most recurring theme across the whole series; worth having partition-key reasoning fully automatic.
3. **Dynamo + Chubby back to back** — for the AP/CP contrast in one sitting.
