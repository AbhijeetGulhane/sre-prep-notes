# Behavioral Interview — 2-Minute Spoken Answers

Read these as scripts to internalize, not to memorize word-for-word. Each is ~260–300 words, roughly 2 minutes at a natural pace. Probes for each story are in your STAR doc — these are just the delivery bodies.

---

## Story 1 — Ownership (eliminated the manual patching sweep)

I was on the team running monthly OS patching across our 60,000-node Linux estate. After every patch wave, engineers had to manually sweep servers — checking for services that hadn't come back up, missing mounts, drifted configs — and fix each one by hand. Roughly 30% of servers came back with minor issues and another 10% with major ones, so close to 40% of the estate needed a hands-on touch every single cycle. Everyone just treated that as the cost of patching.

That wasn't part of my assigned duties, but I decided to eliminate it. I catalogued the recurring failure states — stopped services, unmounted filesystems, processes that hadn't restarted, config drift off baseline. Then I built a Bash-and-Python tool that detected each of those states automatically after a patch wave. I extended it to remediate them without a human — restarting services, restoring mounts, bringing processes back up, correcting drift. And I drove it across the full estate by iterating live over the servers in each patching window, via an SSH loop, so every monthly wave got the automated pass instead of the manual sweep.

The result: that 40% intervention rate got absorbed by the tool. Engineers only touched a box when something fell outside what it handled — which was rare. It ran that way across the entire 60,000-node estate, every cycle, for as long as I was on that team.

What made it stick with me is that nobody asked for it. The sweep was invisible toil — it had just always been done by hand, so nobody questioned it. I think the job isn't just clearing your assigned tickets; it's noticing the recurring pain nobody's questioning and deciding to make it not exist.

---

## Story 2 — Technical Excellence (the file-descriptor leak)

I was on-call when the app team reported a production server had gone fully dark — every service call was being denied. It looked, on the surface, exactly like the server was out of memory or out of disk.

I started with vitals and ruled that out fast — health checks were clean, no memory spike, no disk-full, network was up. So the "out of resources" story didn't hold; it was a lead, not the cause. I went to the application logs next. Nothing conclusive at first glance, but on a closer pass I saw the app opening files heavily and never releasing them. That pointed me at file descriptors as the real constraint.

I confirmed it by counting the process's live open descriptors under `/proc/<pid>/fd`, and used Kibana to see the error pattern building over time rather than a single spike. Other applications sharing the same host were healthy, which ruled out a host-level ceiling and isolated the leak to this one application specifically.

With the app team's approval, I restarted the application, which released the leaked descriptors and restored service. Then I wrote up the incident documenting the FD leak — files opened without cleanup — and handed root cause to the app team as an action item for the actual code fix.

We were back to full service in about an hour. It was an internal application, not customer-facing, but every consumer of that service was fully blocked while it was down, so getting to the real mechanism fast — instead of chasing the memory-or-disk story the symptom was pointing at — is what kept it to an hour instead of a much longer chase.

What I took from it: a misleading alert is a lead, not a verdict. When the symptom and the vitals disagree, trust the mechanism you can actually measure. It's also why I'd push for FD-utilization alerting as the durable control — catching a slow leak with headroom instead of it surfacing as a full outage.

---

## Story 3 — Conflict Resolution (the Hadoop team's pushback)

Between 2019 and 2020 I owned an Active Directory authentication migration end-to-end — moving Barclays servers off an old tool called Likewise onto VAS — from change request through CAB approval to execution and post-migration testing. The Hadoop team refused to let their servers be migrated. In an earlier wave, users had hit login failures right after cutover, and they'd concluded the new tool was breaking authentication.

I needed their servers migrated, but I wasn't going to force it through over a legitimate concern — their objection came from a real prior incident, so I had to understand it, not overrule it. I traced what had actually happened: the old tool allowed expired and locked accounts to keep authenticating, so those accounts looked healthy. The new tool validated live against AD and correctly rejected them. The login failures users blamed on the migration were pre-existing bad accounts the old tool had been silently masking — the migration just exposed them.

Instead of just asserting that, I proved it. I got access to the internal database synced with AD data, used SQL to validate every account's status per application, and handed the app team a concrete list of exactly which accounts would be affected. Then I adapted the plan — I added a phase upfront where affected users reset or fixed their accounts before migration, so nobody would be surprised at cutover. I also built self-service Confluence pages so anyone who still hit a login issue could self-diagnose it as an account problem, not a tool failure.

That evidence and the cleanup phase gave the Hadoop team the confidence to try a small batch first. It migrated cleanly, the audit-first process became the template, and we carried it through to migrate the entire Barclays environment off Likewise — no escalation, ever.

I didn't change my mind about the goal. I changed the plan. Their pushback was protecting against a real scar, and treating it as a signal to de-risk rather than an obstacle to push past is what actually got the migration done.

---

## Story 4 — Failure (the subnet config error)

Early in my career, during server-build work, I was configuring network settings across a batch of servers. I made a mistake — 16 hosts got the wrong subnet mask, and a couple got the wrong IP outright — so those hosts weren't correctly placed on their intended subnet.

I caught it myself during connectivity testing after the build. Testing from the jump host, the wrong-IP hosts were flatly unreachable, because the jump host's firewall rules only permitted the correct subnet, so those addresses got dropped — that's what tipped me off. Digging into why the rest were also failing, I found the wrong subnet masks: those hosts had the right IP, but a miscalculated mask meant return traffic wasn't routing correctly and same-subnet communication broke.

Rather than risk a second wrong change on production-bound servers by guessing at a fix outside my depth, I owned the mistake and brought in the network team to correct it together. Then I changed how I worked so it couldn't recur silently: I added a manual verification step before submitting any build, and wrote a small script that pulled the actual config off each host — network, storage, Chef, host details — and exported it to Excel so I could diff intended versus actual before handover.

I corrected the affected hosts and completed the build, but the real outcome was the process change. The failure wasn't the typo — it was that I had no verification step at all; I was trusting manual config across a batch with nothing checking intended against actual. That same error could have reached production on a bigger build. The verify-don't-assume instinct that came out of it — check actual state against intended state — carried into everything I built afterward.

---

## Story 5 — Cross-team Collaboration (DR events)

Twice a year, per site, across three datacenter sites, we ran disaster-recovery events — failing production over to DR or testing full-site resiliency — so one landed roughly every two months. Each involved 500 to 1,000 servers and 20 to 50 application teams, all executing inside a fixed window on a live bridge with hundreds of people, run by a command-center coordinator driving the sequencing.

I owned my share of the Unix server layer — bringing servers up cleanly per runbook in my slot, then enabling app teams to validate on top of them. When an application didn't come up clean, the app team couldn't see the cause from their side, so they depended on me to go in as root, pull logs, check app config and server state, and translate "the app is down" into a concrete server-side reason fast enough for a decision inside the window.

I managed load by working the coordinator's sequence rather than setting my own priority, confirming with app teams if a server needed to jump the queue, and we ran a secondary on-call so multiple simultaneous issues got split instead of bottlenecking on one person. On one event, an app's RAID configuration couldn't complete due to a hardware issue, and my confirmation it wasn't recoverable in the window drove a clean rollback instead of burning time. On another, an app's traffic wasn't getting through despite every server-side test passing on my end. I worked it live with the network team, and when a hardened firewall path couldn't be cleared before the deadline, our combined diagnosis made the rollback decisive instead of a gamble.

I wasn't running the event — the coordinator was. My part was being the diagnostic bridge 20 to 50 app teams relied on when things didn't come up clean, so the recover-or-rollback call under a hard clock got made on real information, not a guess.

---

## Story 6 — Dealing with Ambiguity (the holiday reporting bug)

In my current role, an application release went out — a UI change plus mostly-DDL database changes. Shortly after, traders reported that cumulative trade reports over certain periods showed wrong numbers, even though the per-trade balance summary looked fine. There was no obvious cause and no clear owner: the DB team checked the schema and found nothing, the app team checked the UI and found nothing, and diagnostics led nowhere.

Rather than re-checking layers others had already cleared, I went at the data itself and narrowed it top-down. Monthly totals summed correctly. Weekly totals also matched. I dropped to the daily summary — and that's where the numbers broke. The daily view localized it precisely: errors landed on specific days — the holiday itself and the Monday right after it. That two-day fingerprint reframed the problem from "the cumulative report is wrong somewhere" to "something about how the system handles holidays and the following business day is off."

I took that pattern to the developers and worked through the code with them against that specific lens, which led us straight to the defect — a single line that accounted for weekends but not holidays, so those periods didn't count the holiday correctly. The fix was patched, tested, and rolled out.

I turned an undefined, cross-team problem that had stalled — schema clean, UI clean, diagnostics dead-ended — into a located, fixed defect by decomposing the data systematically instead of re-litigating layers that were already cleared. I don't own the app codebase; the developers fixed the line. My contribution was defining the problem they'd been unable to define — giving them the holiday correlation so they could go to the exact logic instead of searching blind.

---

## Story 7 — Mentorship / Lifting Others

In 2024, a grad joined our Unix engineering team on her first rotation — eight months on our team, no Linux background at all. The program's stated goal was just to get her familiar with how the org works. I thought that was a low bar for eight months, so I proposed to my manager that we put her on real patching work instead, and he agreed.

I built it as a ladder. She started on read-only incidents — pulling information, nothing that changes state. For anything harder, I'd assign it, sit with her and walk through the pattern for that class of problem rather than just the fix, then route similar tickets to her over the following days so she had to apply it cold. She kept hitting undocumented walls on basic access, so I wrote it up in Confluence as we went — five to ten pages that are still there two years later; every new joiner still works off them.

Then I moved her onto post-patch restarts in non-prod specifically, because the blast radius was contained — it was the one place she could fail on real work at no cost. And I stopped answering. One time she was restarting an app as root and it kept failing because that app needed its own service account. I let her sit on it 20-30 minutes, then instead of telling her, I asked her to pull the logs and find which account the last successful start had used. She got it, and never asked me that class of question again.

Once she moved to prod patching — a hard four-hour window with app teams waiting to test — she hit a service that wouldn't come up and did exactly what I'd trained her to do: she worked it. The window closed with services still down, and I had to tell the app teams the box was up but not in the state they needed. That was my miss, not hers. In non-prod, persistence is the whole lesson. In prod, the lesson is knowing when to stop and hand off — I taught the first and assumed the second would come with it.

By the end of the rotation she was handling 200 to 300 servers a day of post-patch issues, up from ten or twenty, clearing routine incidents unassisted. The environment where struggling is cheap is the easy half of the lesson — the harder half, which I learned from getting it wrong, is that when you move someone out of that environment, you have to say the rules changed out loud.

---

## Story 8 — Intellectual Humility / Being Wrong

In 2023, an active-active application setup went down — no traffic flowing from any server in the subnet. We were on a bridge with the app team, incident manager, and service owners, and I was the Unix side.

I ran server-level health checks and everything came back green — OS fine, servers reachable, internal communication working. The only thing failing was application traffic, and it was failing across every server in the subnet, uniformly. That pointed hard at network, in my read — a server-level fault is usually contained to one or two hosts, not fleet-wide. So I told the network team it was on their side.

They pushed back with their own clean health-check results. I held my position — my argument was the uniformity: if it were a server issue, it would be contained, not affecting every server identically. We went back and forth for about thirty minutes without resolving anything.

The standoff wasn't producing progress, so the app team started tracing their own code path and pulled the list of ports the application actually required. I checked those against the server config and against servers where the same app had been running fine — and one required port had never been in the config on this set. It wasn't a regression; it had simply never been exercised, because these servers had never carried live traffic in the active-active setup until now.

I said on the bridge that the issue was on my side — ports missing in the server config — added them, and asked the network team to confirm nothing was blocking at the port level. We tested and it resolved.

The inference was reasonable, and the conclusion was wrong. "Every server is affected" ruled out a server-specific fault, but it didn't rule out a server-config fault common to all of them — in an identically-built active-active pair, those look the same from outside. The real error underneath was that I treated green Unix health checks as proof my whole layer was clean. They weren't — application-level port config sits in my layer too, and those checks never touched it. A full config diff against a known-working server would have found it in minutes, and that's the question I now ask first: when something works somewhere else, what's actually different about the place it doesn't.

The part I'd still change: I corrected the diagnosis publicly, but I never went back to the network team and acknowledged I'd argued against their data for thirty minutes and they'd been right. The technical fix landed clean. That one didn't.
