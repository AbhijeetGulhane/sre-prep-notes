# Master Fix List — SRE Prep, Resume & LinkedIn
**Compiled: July 14, 2026 (Week 5, Day 2 of prep)**
Everything outstanding from the full review, in priority order, each with a copy-paste prompt to execute it later. Items marked ✅ are already done — listed so nothing is double-tracked.

---

## SECTION A — Prep Plan Status

### A1. ✅ Applied (41 edits, live in sre_combined.html — push to repo if not done yet)
1. Week 5 Sun mock time typo fixed (10:00 AM – 11:00 AM)
2. Guide Day 8 converted to verification pass (was duplicating Guide Day 4)
3. Coding maintenance dose added Weeks 5–8 (2 timed mediums/week: Wk5 Sun, Wk6 Wed, Wk7 Tue, Wk8 Fri)
4. All 6 troubleshooting scenarios BLIND (sadservers.com / AI-generated, answers stripped, renumbered #1–#6 chronologically: Wk8 Mon/Tue/Wed, Wk9 Mon/Thu/Sun)
5. SRE Workbook Ch 12 (NALSD) replaces SDI Ch 1 skim — Wk5 Mon work-hour reading
6. NALSD math pass appended to all 10 design sessions + all 3 mocks (13 items)
7. Capacity drill #3 added (Gmail estimate, Wk6 Wed)
8. Googleyness & Leadership mapping + 3 hypotheticals — Wk8 Thu evening
9. README 30-second skim test — Wk7 Thu Guide Day 14
10. Protect-the-mock note on Interviewing.io troubleshooting session — Wk9 Tue
11. Anki conversion of Wks 3–4 one-liners — Wk10 Wed
12. Week 11 Maintenance mode as proper 11th week (own phase, excluded from main progress %, getWeekNum stays at W11+ forever)

### A2. ⏳ Pending decisions — plan
| # | Change | Status | Trigger |
|---|--------|--------|---------|
| P1 | Re-theme blind scenario #5 or #6 to fintech flavor (low-latency p99 spike under load) | Offered, not applied | Anytime — zero time cost |
| P2 | Amazon LP story remapping item (Wk8 Thu evening or Wk9 flex, ~45 min) | Conditional | Only if Amazon added to target list — currently NOT in scope |
| P3 | Meta PE Linux verbal drill (30-min narrate-without-notes: strace/procfs/signals/cgroups) | Note-to-self, not a plan item | Use a Wk9 flex block; Anki review approximates it |
| P4 | Week 11 checkbox auto-reset (key by calendar week so maintenance weeks reset each Monday) | Deferred by choice ("leave for now") | If manual unchecking gets annoying after 2+ maintenance weeks |

---

## SECTION B — Resume Fixes (priority order)

### B1. 🔴 CRITICAL — integrity risks (fix THIS WEEK, ~30 min, before any application)
1. **Title claim.** Summary says "Senior Site Reliability Engineer" — actual title is Production Engineer. Background checks verify titles; discrepancy can rescind an offer post-loop. Fix: *"Production Engineer with 11 years of SRE and large-scale Linux infrastructure experience…"* — same framing, truthful.
2. **Years of experience.** Summary says 10; it's 11 (2014–2026 with a study gap 2017–2019). Pick the defensible number and use it consistently everywhere (resume, LinkedIn, applications).
3. **Skills tiering.** Current flat list implies production experience in everything. Restructure:
   - *Production scale:* RHEL, Python, Bash, Ansible, Chef, VMware vCenter/ESXi, ELK, Splunk, Prometheus, Autosys, Oracle/MySQL, TCP/IP, Kerberos, SELinux
   - *Working proficiency (lab/project):* Kubernetes, Docker, Terraform, Go, FastAPI, GitHub Actions CI
   - **CUT:** AWS (no supporting bullet or story), Distributed Tracing (post-plan project, doesn't exist yet)
4. **"Nominated for Star Award"** — cut unless won. Nominations read as padding.

### B2. 🟠 STRUCTURAL — the SWE-signal gap (Week 7–8, feeds scheduled resume sessions)
5. **Add Projects section** (this is the load-bearing wall for the ops→SRE-SWE pivot):
   - **search-api** — semantic search API: FastAPI, sentence-transformers, Prometheus 4-Golden-Signals instrumentation, Grafana, K8s (HPA, RBAC, NetworkPolicy), chaos test + 2 postmortems, GitHub Actions CI. 2–3 quantified bullets (Wk7 Thu Guide Day 14 already schedules writing these).
   - **Nonprofit education platform** — FastAPI, React, Databricks Genie, multi-service. 2 bullets.
6. **Add GitHub link to header** (github.com/AbhijeetGulhane) next to LinkedIn. Pin search-api + sre-prep-notes.
7. **Resume variants** (already scheduled Wk8 Fri): 3 SRE-specific (per top Google/Meta JD) + 1 DevOps/Platform variant (Datadog/HashiCorp/Robinhood/Coinbase keywords: platform engineering, developer experience, IaC).

### B3. 🟡 BULLET-LEVEL — quantify or kill (Week 8 Fri resume session)
8. **"1,000+ daily active users"** — actively harmful next to "60,000 nodes." Reframe in a dimension where scale is real: transactions/day, dollar volume, downstream system count, regulatory criticality — or delete the number.
9. **MTTR bullet has no number.** "Systematically reducing MTTR" → from what to what, or by X%. You have MIM data.
10. **Unquantified bullets — add a number or cut:** "Engineered reliability strategies…", "Architected proactive observability frameworks…", "Collaborated…safe deployment pipelines…PRRs" (how many PRRs? how many releases?). Rule: every bullet matches the pattern of your best ones (50+ hrs toil, 200+ parallel updates/hr, 70% latency, $153K).
11. **Verb soup.** Architected/Engineered/Orchestrated/Spearheaded/Directed on nearly every line — reads as padding and as AI-polished. Vary with plain strong verbs: built, automated, migrated, reduced, led, debugged, cut.
12. **Consistency sweep:** same tense discipline, same number formatting (60,000+ vs 20,000+ vs 2,500+ — fine, but be uniform), em-dash/hyphen consistency, one-page-per-decade check (2 pages max at 11 yrs).

### B4. 🟡 LEVEL CALIBRATION (feeds Week 8 STAR sessions — framing, not resume text)
13. You'll be graded at **L5 (Google) / E5 (Meta)**. Every STAR story must answer *"what did YOU decide and drive"* at technical-leadership altitude — designs you drove, direction you set, teams you influenced — not "I executed the runbook well." Patch pipeline + NFS migration bullets have this shape; make all 8 stories match it or you risk down-level/reject at hiring committee despite passing technicals.

---

## SECTION C — LinkedIn (after resume v1 locks, Week 7–8)
1. Headline: mirror fixed summary (Production Engineer | SRE | 60K-node Linux at scale | Python automation) — no title inflation.
2. About section: rewritten summary + link to pinned search-api.
3. Experience: mirror the fixed, quantified resume bullets exactly (recruiters cross-check; discrepancies look bad).
4. Skills section: match the tiered resume list — remove AWS/Distributed Tracing there too.
5. Featured: pin search-api repo + (post-plan) SLO Error Budget Service.
6. Set Open-to-Work (recruiters only) aligned with Aug 10 application date.

---

## SECTION D — Spend / Resources checklist
| Item | Cost | Decision |
|---|---|---|
| Interviewing.io — SRE troubleshooting session | ~$225 | ✅ Budgeted, Wk9, HIGHEST priority |
| Interviewing.io — system design session | ~$225 | Do if budget allows (Wk9 Tue AM already scheduled) |
| LeetCode Premium | ~$35/mo × 2 | Turn on NOW through September (Google-tagged list + frequency) |
| SDI Vol 2 | ~$40 | ❌ Skip — one chapter cited; use free ByteByteGo video |
| ByteByteGo paid, more books, O'Reilly titles | — | ❌ Skip until post-plan projects |
| sadservers, Pramp, Workbook, Cloudflare, iximiuz | $0 | Already in plan |

---

## SECTION E — Copy-Paste Prompts Library
Use these verbatim in future sessions. Attach the referenced file where noted.

### E1. Resume — critical rewrite (do first)
> Attaching my current resume PDF. Rewrite three things per the master fix list: (1) Professional Summary — remove the "Senior Site Reliability Engineer" title claim, frame as "Production Engineer with 11 years of SRE and large-scale Linux infrastructure experience," keep it 3 lines max, no buzzword inflation. (2) Skills section — restructure into "Production Scale" and "Working Proficiency (lab/projects)" tiers per the fix list; delete AWS and Distributed Tracing. (3) Give me the corrected header line with GitHub link added. Output as plain text I can paste into my resume editor. Do not touch the experience bullets yet.

### E2. Resume — bullet quantification session
> Attaching my resume. Go bullet by bullet through the Experience section. For each bullet, classify: KEEP (already quantified), FIX (rewrite needed — draft the rewrite and mark where I must insert a real number, e.g. [X% MTTR reduction]), or CUT (says nothing measurable). Rules: kill verb soup (no Architected/Orchestrated/Spearheaded openers unless earned), reframe the "1,000+ daily active users" line in transaction-volume or criticality terms, quantify the MTTR bullet, remove "Nominated for Star Award." Every surviving bullet must match the pattern: [action] + [number] + [impact]. Also confirm the whole resume fits 2 pages.

### E3. Resume — Projects section draft (run after Guide Day 14, ~Aug 30/Jul 30)
> Draft a Projects section for my resume with two entries: (1) search-api — pull the specifics from what we built: FastAPI semantic search, Prometheus 4-Golden-Signals metrics, Grafana dashboard, K8s with HPA/RBAC/NetworkPolicy, chaos test with 2 postmortems, GitHub Actions CI, load test showing HPA scale 2→5. Write 3 bullets, each quantified, SRE-language. (2) Nonprofit education platform — FastAPI, React, Databricks Genie, CSV data layer. 2 bullets. Format to match my resume style. Then give me the same content reformatted for LinkedIn Featured/Projects.

### E4. Resume — JD tailoring (Week 8 Fri, run once per target JD)
> Here is a job description [paste JD] and my resume [attach]. Produce a tailored variant: reorder skills to match JD keywords, adjust the summary's last line to the company, pick which 2 projects/bullets to emphasize, and list any JD requirement I don't cover with an honest gap note. Do NOT invent experience. Flag any keyword the ATS will screen for that's missing.

### E5. LinkedIn rewrite (after resume locks)
> My resume is final (attached). Rewrite my LinkedIn: headline (under 220 chars, no title inflation — I'm a Production Engineer), About section (first person, 4 short paragraphs: scale story, SRE pivot + search-api, what I'm looking for, certifications), and confirm my Experience bullets should mirror the resume verbatim. Give me the exact text to paste.

### E6. Plan edit — fintech scenario re-theme (P1, apply anytime)
> In sre_combined.html, re-theme blind Troubleshooting Scenario #6 (Wk9 Sun) to fintech flavor: "low-latency service p99 spike under sustained load — diagnose live." Keep it BLIND (no embedded answer), keep the session name/position so checkbox keys don't shift, only change the item text. Verify with the same anchor-assert method as before and give me the updated file.

### E7. Plan edit — Amazon LP mapping (P2, ONLY if Amazon added to targets)
> I'm adding Amazon to my target companies. Add one item to Week 8 Thu Evening Study in sre_combined.html: "Map all 8 STAR stories to Amazon Leadership Principles; identify 2 LPs with no story and draft one each (~45 min)." Append at END of that session's items so keys don't shift. Also tell me which of my existing 8 stories map to which LPs, and which LPs are uncovered.

### E8. Plan edit — Week 11 auto-reset (P4, if manual unchecking annoys)
> In sre_combined.html, make Week 11 checkboxes reset weekly: key W11 items by ISO calendar week (e.g. 11-2026W36_Mon_0_0) so each Monday starts fresh, past maintenance weeks stay in progress.json history, Weeks 1–10 keys and main progress % untouched. Smoke-test like before and give me the file.

### E9. Blind troubleshooting scenario generator (Weeks 8–9, use for AI-generated reps)
> Act as an SRE interviewer running a live troubleshooting round. Invent a production incident (Linux or Kubernetes) — do NOT tell me the root cause. Give me only the initial alert/symptom. I will ask for specific evidence (dmesg, /proc data, kubectl output, metrics graphs, logs) one item at a time — reveal ONLY what I ask for by name, realistically fabricated and consistent with your hidden root cause. Push back if I'm vague ("which pod? which node?"). Time-box 20 minutes. At the end: reveal root cause, grade my diagnosis path (hypothesis quality, tool choice, anchoring, speed), and give 3 specific improvements. Difficulty: [medium/hard]. Flavor: [Linux/K8s/networking/fintech-latency].

### E10. NALSD mock evaluator (Sundays, Weeks 5–7)
> Act as a Google SRE interviewer running an NALSD round (SRE Workbook Ch 12 format). Problem: [design X at Google scale]. I'll talk through my design; you interject only as a real interviewer would. Grade specifically on NALSD criteria: did I compute actual machine counts, per-machine disk/IOPS/RAM math, identify the first numerical bottleneck, and iterate the design when it broke? After 45 min, give a scorecard: requirements, estimation math, API, storage, scaling iteration, failure modes, SLO awareness — each rated with the single biggest gap.

### E11. STAR story L5 framing review (Week 8)
> Here are my 8 STAR stories [paste]. Grade each against the L5/E5 bar: does it show technical leadership (decisions I drove, direction I set, influence beyond my lane) or just strong execution? For any story graded "execution-level," show me exactly how to reframe it using facts I already stated — do not invent. Then map each story to Google G&L signals (ambiguity, intellectual humility, bias to action) and flag which signals have no coverage.

### E12. Capacity drill generator (Week 7+, extra reps)
> Give me one timed 15-minute capacity estimation drill for [system]. State only the user-facing assumptions (users, activity rates, object sizes). I'll work it cold and paste my answer; then check my math, show the reference calculation, and point out which estimation habit cost me the most accuracy.

### E13. Meta PE Linux verbal drill (P3, Week 9 flex slot)
> Act as a Meta Production Engineer interviewer running the Linux systems round. Ask me a chain of progressively deeper questions on ONE topic from: process lifecycle/signals, /proc + debugging with strace, cgroups v2, namespaces/containers, memory management, TCP internals. Start practical ("a process won't die with kill -9 — walk me through it") and drill into internals with follow-ups based on my answers. 30 minutes. Then grade depth, precision, and where I hand-waved.

---

## SECTION F — Execution timeline (when each fix happens)
| When | Items |
|---|---|
| **This week (Wk5)** | B1 critical resume fixes (E1) · LeetCode Premium on · push amended plan file · optionally P1 (E6) |
| **Week 7** | Guide Day 14 → project bullets → E3 Projects section · GitHub link/pins |
| **Week 8 Fri** | E2 bullet quantification · E4 per-JD variants · DevOps variant |
| **Week 8 Thu–Sun** | E11 STAR/L5 review · G&L mapping (in plan) · E7 only if Amazon confirmed |
| **Weeks 8–9** | E9 for AI-generated blind scenarios · E13 in a flex slot |
| **Before Aug 10** | E5 LinkedIn · final consistency sweep (B3 #12) |
| **Week 11+** | E8 if unchecking annoys · post-plan: SLO Error Budget Service, then Distributed Tracing → only then re-add "Distributed Tracing" to skills |
