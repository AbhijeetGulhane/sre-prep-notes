# Complete Conversation Review — Observations, Decisions, Actions
**Compiled Jul 16, 2026 (Week 5) · Consolidates everything across this conversation.**
Companion files (all should live in sre-prep-notes/planning/): master-fix-list.md (E-prompts) · profile-gap-prompts.md (P-prompts) · linkedin-overhaul.md (paste-ready text + L-prompts) · todo-wk05.md.

---

## PART 1 — KEY OBSERVATIONS (what we learned)

### Prep plan
1. **Rated 8.5/10** — top-5% structure. Strengths: correct phase ordering, rare Linux-internals depth, theory↔hands-on coupling via search-api, honest slippage tracking, self-correcting (networking remediation).
2. Original weaknesses (all now fixed via 41 verified edits): self-authored troubleshooting scenarios with embedded answers → 6 BLIND scenarios; 6-week coding dormancy → weekly maintenance dose Wks 5–8; ByteByteGo-style design ≠ Google NALSD → Workbook Ch 12 + math pass on all 13 design/mock sessions; no post-Aug-23 maintenance → Week 11 (in PLAN array, excluded from main %, getWeekNum stays W11+).
3. **Execution status: ahead.** 50% (186/369) on day 31/70. Weeks 1/3/4 fully closed. Guide Day 7 (chaos test + Postmortem #1) done Jul 15.
4. **One negative pattern in the data: hard-problem avoidance** — all 12 unchecked revision items skew hard (LFU, Sliding Window Max, Trapping Rain Water, Min Window Substring…), exactly the Google-tagged genre.
5. Week 2 FULL TIMED MOCK was never done — zero coding-mock reps to date.
6. Week 6 correction: outings are Fri–Sun only; Thu Jul 23 is a full study day → Guide Day 13 lands on schedule, no cram.
7. Plan transfers ~80–85% to other FAANG/infra targets; NALSD math format is the only Google-specific piece (and it improves all design rounds anyway).

### Project & resources
8. search-api: excellent as a skills refresher (its main job), good-not-spectacular as portfolio; its ceiling-raiser is the README reliability story + the post-plan SLO Error Budget Service.
9. Books owned (SRE Book, DDIA 2e, TLPI, SDI v1) = sufficient core. Total remaining spend: ~$300–500 (Interviewing.io session(s) + LeetCode Premium ×2 months). Skip: SDI Vol 2, ByteByteGo paid, more books, **LinkedIn Premium** (referral + 1st/2nd-degree outreach bypass what it sells; optional 1-month trial in August only if cold recruiter outreach becomes necessary).

### Resume (current PDF)
10. **Two integrity risks:** "Senior Site Reliability Engineer" title claim (verification risk — actual titles: Service Engineer, prev. AVP Unix Server Engineer); skills inflation (K8s/Terraform/Go listed as production-peer skills; AWS with zero supporting evidence; Distributed Tracing doesn't exist yet). Also 10 vs 11 years inconsistency.
11. **SWE signal thin** — the axis Google/Meta grade hardest: no Projects section, no GitHub link; all bullets ops-flavored.
12. "1,000+ daily active users" actively harms next to "60,000 nodes"; MTTR bullet unquantified; ~⅓ of bullets have no number; verb soup (Architected/Orchestrated/Spearheaded); "Nominated for Star Award" = cut.
13. **Level calibration: graded at L5/E5** with 11 yrs — stories must show technical-leadership altitude or risk down-level at HC despite passing technicals.

### LinkedIn (was worse than the resume)
14. Profile was ~6 years stale: "Unix System Administrator with 9+ years"; **Top Skills = Product Development/Management/R&D** (indexed as a PM); current PE role EMPTY; "keep the lights on" verbatim; "Service Recovery Events (SREs)" acronym collision; SAP tool inventory indexing legacy-BI searches; high school listed; zero recommendations.
15. Full replacement text written for every section (linkedin-overhaul.md). Recruiter-simulated verdict: 4/10 → 7.5/10 after edits → 8.5/10 ceiling after L1–L7. Non-fixable residuals: all-bank pedigree, lab-only K8s (honestly framed), single-employer tenure (countered by visible 3-promotion arc), no recommendations yet (L7).

### Google-specific intelligence
16. **Routing: SRE-SysEng, not SWE-SRE.** The 3 bookmarked "SWE III SRE" reqs are **L4** — an 11-year profile on L4 reqs raises flags. L4-vs-L5 decision required before Aug 10 (intel call converts this from guesswork to data).
17. **Borg-engineer referral (good college friend) = single biggest lever in the entire effort.** Sequence: referral BEFORE any application. RHCSA is neutral at Google (valuable at fintech targets). MS clears Google's education soft-filter.
18. Title/verification resolved: Workday says **Service Engineer** (both fields) → use **"Service Engineer (Production Engineering)"** identically on resume + LinkedIn + applications. Still open: verify AVP-era recorded title in Workday.
19. BA4-after-AVP internal level-down: **never in writing** (internal grades invisible/meaningless externally; verification checks titles+dates only). One 20-sec verbal script for the timeline walk (restructuring → deliberate move toward reliability). Externally the move reads as a positive title change. Consequence: L5 evidence leans on AVP-era stories.

### Skill gaps the plan won't fill (handled by narrative, not study)
20. Production cloud (hurts platform-co's most, Google least — Borg argument); production software ownership (no real pager); internet-scale deploy rhythm vs bank cadence; low-latency trading specifics (avoid core-trading reqs); recent formal leadership scope (down-level risk); Go depth/OSS (minor). Stored mitigation candidate: deploy search-api to GKE Autopilot in Wk11+ (in memory, add on request).

---

## PART 2 — DECISIONS LOG
| Decision | Outcome |
|---|---|
| Week 11 in PLAN array, main % = Wks 1–10 only | ✅ Implemented |
| Week 11 checkbox auto-reset | Deferred ("leave for now") |
| 12 hard revision problems | This week: 4 Thu / 4 Fri / 4 Sun |
| W2 timed mock | This Sunday, upgraded coding-maintenance slot; check both keys |
| Week 6 shape | Mon–Thu study, Fri–Sun outings |
| GKE deployment | Saved to memory, add later on request |
| Current-role title | "Service Engineer (Production Engineering)" everywhere |
| BA4 level-down | Not in writing; verbal script only |
| LinkedIn Premium | Skip; optional Aug trial only if cold outreach needed |
| Referral sequencing | Referral first week of Aug BEFORE applying; intel call ASAP |
| PENDING: L4 vs L5 floor | Decide before Aug 10 — informed by intel call |
| PENDING: Amazon in target list? | Drives P2/E7 |
| PENDING: P1 fintech re-theme of scenario #6 | Recommended yes; prompt E6 |
| PENDING: regenerate resume/LinkedIn docs with hybrid title | Offered, unanswered |

---

## PART 3 — MASTER ACTION LIST (chronological)

### 🔴 THIS WEEK (Wk5, by Sun Jul 19)
- [ ] **E1 resume critical rewrite** — with the NEW hybrid title "Service Engineer (Production Engineering)"; 11 years; tiered skills (cut AWS, Distributed Tracing); GitHub in header; cut Star Award nomination
- [ ] **Message Borg friend** → schedule intel call (routing, leveling, hiring teams). Critical path: everything downstream depends on it
- [ ] Workday: verify AVP-era recorded title (one HR question covers all three Barclays entries)
- [ ] **Extract real MTTR number** from MIM data — feeds resume + LinkedIn [X] + incident STAR story
- [ ] LinkedIn immediate: turn OFF "share edits with network" FIRST → new headline + About (hybrid title version) → skills purge (delete Product-*) + top-3 = SRE/K8s/Python → fill PE role section → delete "keep the lights on"/SRE-acronym/Equities cert/high school/SAP tool dump
- [ ] LeetCode Premium ON (through Sept)
- [ ] Commit 5 planning docs to sre-prep-notes/planning/ (incl. this file)
- [ ] Hard-problem clears: 4 Thu · 4 Fri · 4 Sun
- [ ] Sunday: NALSD Mock #1 (recorded, math pass) + W2 timed mock (2×45) + Guide Day 9 + Workbook Ch 1
- [ ] Decide: P1 re-theme (recommend yes) · Amazon in/out

### WEEK 6 (Jul 20–23 study days)
- [ ] Designs + Guide Days 10–13 per corrected preview · capacity drill #3 · 2 LC mediums
- [ ] Send fixed resume to friend for insider review

### WEEK 7 (Jul 27–Aug 2)
- [ ] Guide Day 14 SHIP search-api → README 30-sec skim test
- [ ] L1 Featured entry · P-A GitHub audit · E3 Projects section onto resume
- [ ] P-E referral outreach (non-Google targets) · L7 recommendation requests
- [ ] Aug 1: L3 LinkedIn post

### WEEK 8 (Aug 3–9)
- [ ] First week of Aug: **friend submits Google referral** against chosen req IDs (L4/L5 decided)
- [ ] E2 bullet quantification (MTTR number in) → L2 LinkedIn sync → E4 per-JD variants + DevOps variant
- [ ] STAR stories with L5 altitude → E11 review · G&L mapping (in plan) · P-B K8s narrative · P-C timeline walk (incl. BA4 script) · P-D ops-to-SWE reframe
- [ ] Blind scenarios #1–3 (in plan) · E7 only if Amazon confirmed
- [ ] Aug 8–9: L4 consistency sweep · Open to Work recruiters-only ON · P-F recruiter-screen prep

### WEEK 9 (Aug 10–16)
- [ ] Applications window opens — referral already in
- [ ] Pramp mocks · Interviewing.io troubleshooting (PROTECTED — highest priority of Wks 8–9) · friend-run systems mock (absorbs P3) · blind scenarios #4–6
- [ ] P-G weakness-probe gauntlet (incl. AVP→engineer probe)

### WEEK 10 (Aug 17–23)
- [ ] Taper per plan · Anki conversion of Wks 3–4 one-liners · final story polish

### WEEK 11+ (Aug 24 →)
- [ ] Maintenance loop weekly (2 LC · 1 design sketch · Linux review · pipeline check)
- [ ] SLO Error Budget Service → L5/P-H positioning when real
- [ ] Candidates on request: GKE deployment (memory) · Wk11 auto-reset · L6 if GKE done
- [ ] Friend nudges recruiter after 2+ weeks of silence

---

## PART 4 — THE CRITICAL PATH (if everything else fell away)
1. E1 resume fix → 2. Intel call with Borg friend → 3. L4/L5 decision → 4. Referral submitted first week of Aug → 5. Interviewing.io troubleshooting mock Wk9 → 6. MTTR number threaded through resume/LinkedIn/STAR story.
Everything else amplifies; these six carry the outcome.
