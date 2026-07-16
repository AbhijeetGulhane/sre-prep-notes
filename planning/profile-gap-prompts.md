# Profile Gap Assessment & Fix Prompts (P-series)
**Companion to master-fix-list.md (E-series = resume/plan/LinkedIn mechanics). This file = profile-level gaps: capability narratives, career story, network, screens.**
Compiled: Jul 15, 2026.

---

## The assessment in one table

| Gap | Type | Closeable by Aug 10? | Fix |
|---|---|---|---|
| Production K8s / cloud (lab-only) | Capability | ❌ No | Honest narrative + lab depth (P-B) |
| Production software ownership (scripts ≠ services) | Capability | ❌ Partially | search-api + SLO project + reframe (P-D, E3) |
| Distributed systems design experience | Capability | ❌ In progress | Phase 3 + NALSD reps (in plan) |
| Title/skills integrity on resume | Framing | ✅ 30 min | E1 (do first) |
| L5 leadership altitude in stories | Framing | ✅ Wk8 | E11 |
| Scale metrics (1,000 DAU problem) | Framing | ✅ Wk8 | E2 |
| Career narrative arc (pivot story, timeline walk) | Framing | ✅ Wk8–10 | P-C |
| GitHub portfolio presentation | Framing | ✅ Wk7 | P-A + E3 |
| LinkedIn | Framing | ✅ Wk7–8 | E5 |
| NALSD / troubleshooting / coding format | Format | ✅ In plan | Already scheduled |
| No referrals yet, no FAANG brand | Structural | Partially | P-E, P-F |
| Weakness probes ("no cloud?", "why now?") | Structural | ✅ Wk8–10 | P-G |

**Bottom line:** capability gaps get honest prepared narratives (never bluffed), framing gaps get rewritten this month, format gaps are already handled by the plan. Your upside is concentrated in the framing column.

---

## P-series prompts (copy-paste later)

### P-A · GitHub portfolio audit (Week 7, after Guide Day 14)
> Act as a FAANG SRE hiring manager spending exactly 90 seconds on my GitHub profile (github.com/AbhijeetGulhane) before a phone screen. Audit: profile README (exists? says what?), pinned repos (right ones? search-api and sre-prep-notes should lead), search-api repo (does the README pass the 30-second skim test — architecture diagram, SLOs, Grafana screenshot, postmortem links at top?), commit history signal (steady? meaningful messages?), and anything that looks unfinished or embarrassing. Give me a ranked fix list with exact text/structure for the profile README and repo descriptions. Then tell me honestly: what impression does this profile leave in 90 seconds, and what single change most improves it?

### P-B · The K8s/cloud gap narrative (Week 8, rehearse before any screen)
> I'm interviewing for SRE/PE roles. My Kubernetes and cloud experience is lab/project-only (minikube, Docker, a full Prometheus+Grafana+HPA+RBAC project) — my production experience is 60,000-node RHEL/VMware at a regulated bank. Interviewers will probe this. Draft and drill me on honest, confident answers to: (1) "Tell me about your production Kubernetes experience." (2) "You've never run cloud infrastructure — why should we trust you with ours?" (3) "How would you ramp on our K8s platform?" Rules: no inflation, no apology spiral. Structure each answer as: what I HAVE operated at scale → what transfers directly (cgroups, networking, scheduling, incident response are the same primitives) → what I built in lab to close the gap → concrete ramp plan. Keep each under 90 seconds. Then role-play the follow-up probes an unconvinced interviewer would ask.

### P-C · Career narrative + timeline walk (Week 8, before "tell me about yourself" polish in Wk10)
> Build my career narrative for SRE interviews. Timeline: Infosys India 2014–2017 (systems engineer, ETL/reporting) → MS Computer Science, Stony Brook 2017–2019 → Barclays 2019–now (Infrastructure Developer → AVP Unix Server Engineer → Production Engineer, 60K-node estate, MIM, patch automation). Craft: (1) the 90-second "tell me about yourself" with a clear arc — each move must have a WHY that points toward SRE; (2) answers to timeline probes: "why did you leave India for the MS?", "why Barclays over tech after Stony Brook?", "you've been at Barclays 7 years — why move now?", "why SRE specifically and not staying in infrastructure?"; (3) flag anything in my story an interviewer could read negatively and give me the honest reframe. No fabrication — work only with these facts.

### P-D · Ops-to-SWE reframe drill (Week 8, feeds resume + stories)
> My background is infrastructure operations; target roles grade software engineering signal. Take these artifacts: patch-automation pipeline (Python/Bash, 200+ parallel updates/hr, 50+ hrs toil eliminated), Chef InSpec compliance framework (2,500+ nodes), MS SQL automation (95% DBA-effort reduction), search-api project, nonprofit FastAPI/React platform. For each: reframe it as a SOFTWARE story — design decisions, interfaces, error handling, testing, iteration — not an ops story. Give me (a) one resume bullet in software-engineering language, (b) a 60-second verbal version emphasizing engineering judgment, (c) the follow-up question an interviewer would ask about the CODE (not the outcome) and my answer's key points. Flag which artifact is my weakest software story so I lead with the others.

### P-E · Referral outreach drafts (Week 7 Sun — the Aug 2 milestone)
> Draft referral outreach messages for SRE/PE applications. Contexts: (1) Stony Brook alum at Google I've never met — LinkedIn message, under 100 words; (2) former colleague now at Meta — warmer, still concise; (3) cold message to an SRE whose blog/talk I genuinely engaged with. Each must: name the specific role/req ID, give ONE concrete credibility hook (60K-node Linux estate at Barclays + SRE project with SLOs/postmortems), make the ask trivially easy ("would you be open to referring, or a 15-min chat first?"), and NOT read as a template. Also give me the 1-week polite follow-up for each, and tell me what NOT to say (no life story, no resume dump in message one).

### P-F · Recruiter screen prep (before Aug 10 applications)
> Prep me for FAANG recruiter screens as an 11-year infra engineer targeting L5/E5 SRE-SE and PE roles, based in Jersey City with flexibility for NYC and openness to other US hubs. Drill me on: (1) the level conversation — how to signal L5/E5 without demanding it, and what to say if they float L4/E4; (2) compensation expectations question — current comp disclosure strategy by state law, deflection phrasing that doesn't anchor me low; (3) location/hybrid questions; (4) timeline coordination — how to pace multiple pipelines so offers land together; (5) the "walk me through your background" 2-minute version tuned for a non-technical recruiter (outcome words, not internals). Role-play the screen, then critique my answers.

### P-G · Weakness-probe gauntlet (Week 9–10, after mocks)
> Run a hostile-but-fair probe session. You are a skeptical FAANG interviewer/HC member; ask me the questions my profile invites, one at a time, with follow-ups when my answer is weak: "No public cloud in production — this role is 100% GCP/AWS." · "Your K8s is a side project." · "Seven years at one bank — can you handle our pace?" · "Your coding is bank-infra scripts; our bar is SWE." · "This resume says SRE but your title says Production Engineer." · "1,000 daily users — our smallest service does that per second." · "Why didn't you go to tech after your MS?" After each answer I give, grade it (convincing / defensive / evasive), show the stronger version, and only move on when my answer lands. End with the 3 probes I handled worst.

### P-H · Post-plan project positioning (Week 11+, once SLO Error Budget Service is partially built)
> My SLO Error Budget Service now exists (state: [describe what works]). Position it: (1) 3 resume bullets, quantified, software-engineering language; (2) LinkedIn Featured blurb; (3) the 2-minute "walk me through a project" interview version with a NALSD-style number in it; (4) the anticipated deep-dive questions (burn-rate math, multi-window alert logic, why these thresholds) with my answer outlines; (5) honest scope framing — what it does NOT do, said confidently. Then tell me whether it's strong enough to REPLACE the nonprofit platform on my resume or should sit alongside it.

---

## Sequencing (merges with master-fix-list Section F)
| When | Run |
|---|---|
| Week 7 (post Guide Day 14) | P-A (GitHub audit) · P-E drafts before Aug 2 outreach |
| Week 8 | P-C (narrative) · P-D (reframe) → both feed the STAR sessions · P-B (K8s narrative) |
| Before Aug 10 | P-F (recruiter screens) |
| Weeks 9–10 | P-G (probe gauntlet — after mocks, when answers are battle-tested) |
| Week 11+ | P-H (once SLO project is real) |
