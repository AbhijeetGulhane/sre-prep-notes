# Behavioral STAR Stories — SRE / PE Loops

> Three locked, pressure-tested stories. Every detail is verified against real work — no invented actions, no placeholder metrics. Probe answers are load-bearing; rehearse them, not just the STAR body.

---

## Story 1 — Ownership

**Competency:** Took initiative beyond assigned scope; eliminated toil.

**Situation:** I was on the team running monthly OS patching across our 60,000-node Linux estate. After each patch wave, engineers had to sweep servers by hand — checking for down services, missing mounts, processes that hadn't come back up, and config drift, then fixing each one manually. Every cycle, roughly 30% of servers came back with minor issues and another ~10% with major ones, so nearly 40% needed a hands-on touch — and everyone treated that sweep as an unavoidable part of patching.

**Task:** Outside my BAU patching duties, I took it on myself to eliminate that manual post-patch sweep by building automation for it.

**Actions:**
- Catalogued the recurring post-patch failure states engineers were resolving by hand every cycle — stopped services, unmounted filesystems, processes that hadn't restarted, and configuration drift off baseline.
- Built a Bash-and-Python tool that ran post-patch to detect each of those states automatically.
- Extended it to remediate them without a human — restarting stopped services, restoring mounts, bringing processes back up, and correcting drift back to baseline.
- Drove it across the estate by iterating over the servers live in each patching window via an SSH loop, so every wave of the monthly program got the automated pass instead of a manual one.

**Result:** Absorbed the manual post-patch sweep across the 60,000-node estate. Where ~40% of servers per cycle (30% minor, 10% major) previously required a hands-on fix, the tool caught and remediated those states automatically — engineers only touched a box when something fell outside what it handled.

**Metric discipline:** Scope is the metric — 60,000 nodes, ~40% intervention rate absorbed. Do **not** attach an hours-saved figure unless intervention counts per cycle were actually tracked; "eliminated the manual sweep" beats a soft percentage.

**Probes to have ready:**
- *"Did the tool auto-remediate the 10% major issues, or escalate them?"* — Answer with the real boundary between auto-fix and escalate. If it auto-fixes minors and escalates majors, say so explicitly; that kills the "were you masking real failures?" probe.
- *"Serial SSH loop over a window — what happens on an unreachable/hung host? Is it safe to re-run mid-window (idempotent)?"* — Have a one-liner on skip/retry behavior and re-run safety.
- *"How did you reach 60K servers with a Bash/Python tool?"* — The SSH loop over servers live in the patching window is the distribution mechanism; if it rode on an existing orchestration/patching layer, say so.

---

## Story 2 — Technical Excellence (systematic debugging / root-cause)

**Competency:** Cut through a misleading symptom to the true mechanism.
**Framing rule:** This is the *"a time the obvious answer was wrong"* story — **not** "hardest problem." Single server, ~1 hour, restart-restored. Slot it correctly or it underdelivers.

**Situation:** I was on-call when the application team reported that a production app server running one of the bank's applications had gone fully dark — every service call was being denied. The symptom presented like the server was out of memory or disk space.

**Task:** As on-call engineer, restore the server and find the actual cause, not the one the symptom was pointing at.

**Actions:**
- Started with the vitals and ruled out the obvious: health checks were clean — no memory spike, no disk-full, network up — so the "out of resources" read didn't hold, which told me the surface symptom was a lead, not the cause.
- Went to the application logs; not conclusive at first, but on a closer pass I saw the app opening files heavily and never releasing them, which pointed me at file descriptors as the real constraint.
- Confirmed it by counting the process's live open descriptors under `/proc/<pid>/fd`, and used Kibana to see the error pattern building over time rather than a one-off spike — then isolated it to this one application, since other apps sharing the same host were healthy, ruling out a host-level ceiling and pointing at an application leak.
- With the app team's approval, restarted the application — which released the leaked descriptors and restored service — then wrote up the incident summary documenting the FD leak (files opened without cleanup) and handed the root cause to the app team as an action item for a durable code-level fix.

**Result:** Restored the server to full service in about an hour by isolating the true mechanism — an application file-descriptor leak that had exhausted the process's FD table until it could no longer accept new connections — instead of chasing the memory/disk symptom. The value is the diagnostic path and the clean root-cause handoff; no metric beyond restoration.

**Causal chain (say it out loud):** File leak fills the process's FD table → no descriptor free for new sockets → `accept()` fails → all calls refused. Files and sockets draw from the same FD table; the file leak starved the sockets.

**Tool lanes (don't blur these):** `/proc/<pid>/fd` = live descriptor count (the confirmation). Kibana = log/error pattern over time (the buildup), **not** a live FD gauge.

**Probes to have ready:**
- *"Restart just masked it — did it recur?"* — Restart was the **mitigation**; the durable fix was the app team's code change, driven via the incident summary. Don't claim you prevented recurrence — diagnosed, mitigated, escalated. That's the correct on-call boundary.
- *"Why not just raise the ulimit?"* — Left it deliberately; it was a standard bank-wide parameter, and raising a limit on a genuine leak only delays the same outage without fixing a process that never closes what it opens.
- *"How did you know it was a leak vs. legitimate high load?"* — Shared-host isolation: other apps on the same box were healthy, so it wasn't a host ceiling; it isolated to the one app, files opened and never released.
- *"Did you see `too many open files`/EMFILE in the logs, or infer it?"* — Be honest: logs *pointed* at heavy file-opening; `/proc/<pid>/fd` *confirmed* exhaustion. Don't upgrade to a log line you didn't see.
- **Prevention (memorize — converts the one real gap into competence):** *"The durable SRE control is FD-utilization alerting — `fd_used / fd_limit` over ~80% pages you with headroom, turning a full outage into an early warning."*

---

## Story 3 — Conflict Resolution

**Competency:** Treated a stakeholder's fear as legitimate; resolved via evidence and plan-adaptation, not authority.
**Anchor line:** *"I changed the plan, not my mind about the goal."*

**Situation:** In 2019–2020 I owned the end-to-end flow of an Active Directory authentication migration — moving Barclays servers off the old Likewise tool onto VAS — running it from change request through CAB approval to execution and post-migration testing. The Hadoop team pushed back and refused to let their servers be migrated: in an earlier migration of a different server set, users had hit login failures right after cutover, and they'd concluded the new tool caused auth problems.

**Task:** I needed their servers migrated, but I wasn't going to force a change through over a legitimate concern. Their objection came from a real prior incident, so I had to understand it and resolve it — not overrule it.

**Actions:**
- Took their pushback seriously and traced what had actually gone wrong: the old Likewise tool allowed expired and locked accounts to keep authenticating, so those accounts appeared healthy; VAS validated live against AD and correctly rejected them. The login failures users blamed on the new tool were pre-existing bad accounts the old tool had been masking — the migration only exposed them.
- Instead of just asserting that, proved it: got access to the internal database synced with AD data and used SQL to pull and validate every account's status per application, then handed the app team the concrete list of exactly which accounts would be affected — showing the exposure was locked/expired accounts, not the migration.
- Adapted my plan around their concern rather than pushing my original timeline: added a new phase up front and notified affected users to reset or fix their accounts *before* migration, so no one would be surprised at cutover.
- Created Confluence self-service pages documenting the issue so any user who still hit a login problem could self-diagnose it as an account-status issue rather than a tool failure — reducing the support burden on both teams.

**Result:** The evidence and the pre-migration cleanup phase gave the Hadoop team the confidence to agree to a small batch of servers first, which migrated cleanly. That pilot validated the approach, and the account-audit-first process became the template that carried through the rest — we completed the migration and moved the *entire* Barclays environment off Likewise. The whole disagreement resolved through the audit-and-communication process, with no escalation.

**Probes to have ready:**
- *"Weren't you just right and they came around?"* (the disqualifier) — *"I didn't change my mind about migrating — that was correct. I changed the plan. Their pushback made me add an account-cleanup phase so we wouldn't repeat the failure they'd seen. That was the common ground — fix accounts first, then migrate in stages."*
- *"Why did expired accounts work under the old tool but not the new one?"* — Describe the **observable behavior**: old tool let invalid accounts authenticate; new tool enforced live AD status and rejected them. If pushed on the exact internal reason (offline caching vs. not enforcing status flags): *"I verified the account side via the AD-synced data — I couldn't tell you the precise internal credential-handling of the old tool without checking its config."* Don't invent the internals.
- *"Isn't CAB escalation?"* — *"CAB approved the change as standard governance — but I never escalated the disagreement itself. I resolved that directly with the Hadoop team by de-risking their actual concern."*
- *"Did adding a phase slip the schedule?"* — *"It added time up front but prevented a failed migration and the rework and lost trust that follow. A cutover that locks out real users isn't faster — you pay for it twice."*
- *"How senior were you?"* — Straight: *"I was the engineer running the migration end-to-end — owned it from change request through testing."* Accurate and strong for early-career; don't inflate to "led the team."

---

## Cross-cutting rules
- Every action beat above is confirmed real. Never add a beat you can't defend under a "how did you measure/verify that?" follow-up.
- No MTTR figure anywhere — none was ever tracked.
- When unsure of a number six years on, say "I don't recall the exact figure" and pivot to what you *did* verify. Confident non-answers beat guessed specifics.
