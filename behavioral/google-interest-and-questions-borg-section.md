# Section: "A Google system you admire technically" — Borg

> Drop-in section for `google-interest-and-questions.md`.
> Round 1 behavioral, Tue Sept 15, 2:00pm.
> Target length when spoken: **55–65 seconds**. This is the behavioral round, not a design round.

---

## Decision: Borg, not Spanner

Locked. Do not reopen.

**Why Borg:**
- capsule is a direct bridge — namespaces, cgroups v2, pivot_root, user namespaces in Go. The containment layer Borg is built on.
- Follow-ups land on isolation, preemption, bin-packing, scheduling under failure. Operational ground.
- Referral is a Borg engineer, so the topic is natural.

**Why not Spanner:**
- Follow-ups land on TrueTime, Paxos groups, external consistency, 2PC across Paxos groups.
- Consensus depth is Raft, not Paxos. No built artifact anywhere near it.
- Reciting a paper to someone who may have written part of the system.

---

## The 60-second script

Anchor mechanism: **resource reclamation.** One mechanism, not an architecture tour.

> The system I keep coming back to is Borg, specifically resource reclamation. Prod jobs at Google request resources based on their worst case, so there's a permanent gap between what's requested and what's actually used. Borg estimates that gap continuously and schedules non-prod batch work into it, with prod work able to preempt batch when it needs the headroom back.
>
> What I find interesting is that it isn't really a scheduling optimization — it's a decision to run the fleet in a deliberately oversubscribed state, and then build the isolation and preemption machinery strong enough that the safety property still holds. The reliability guarantee comes from the containment layer, not from leaving the capacity empty.
>
> That's what pulled me in. I wrote a minimal container runtime in Go — namespaces, cgroups v2, pivot_root — and building it made concrete how much of that guarantee rests on the cgroup limits actually holding under pressure. At Barclays the equivalent question was fleet capacity sitting idle because nobody trusted the isolation enough to share it. Borg's answer is to invest in the isolation until sharing is safe.

**Then stop.** Let them ask. Sliding into a fifteen-minute architecture lecture fails the round you're actually in.

### Backup anchor (if steered away from reclamation)

Scheduler decoupling: the scheduler runs against a **cached** copy of cell state and acts as an advisor. Borgmaster validates and rejects placements that turn out infeasible, sending them back for rescheduling. Good answer to "how do you scale a logically centralized controller."

---

## Fact sheet — safe to state

All from Verma et al., *Large-scale cluster management at Google with Borg*, EuroSys 2015. Say **"as of the paper"** for anything numeric.

| Thing | Detail |
|---|---|
| Cell | Unit of management. Median ≈ 10k machines. Size limited by blast radius, not by Borg's scalability. |
| Borgmaster | 5-way replicated, Paxos-based store, leader elected via Paxos. Replicas serve read-only RPCs. Periodic checkpoints. |
| fauxmaster | Replays checkpointed state for debugging and what-if analysis. |
| Borglet | Agent on every machine. **Borgmaster polls it** every few seconds via stateless link shards. Pull, not push. |
| Scheduling | Two phases: feasibility checking, then scoring. Scheduler decoupled from Borgmaster, works off cached state. |
| Priority bands | monitoring / production / batch / best-effort. Prod is not preemptible. |
| Admission control | Quota expressed as a resource vector at a priority, over a time window. Over-quota jobs rejected at submission. |
| Alloc | Reserved set of resources on a machine. Roughly the Kubernetes pod. |
| Job spec | BCL. Naming via BNS. |
| Sigma | Web UI; annotates unscheduled jobs with "why pending?" — a debuggability detail, one sentence max if toil comes up. |

### Connective tissue (use at most one)

- **Pull-based Borglet polling → Prometheus pull model.** Ties to the observability prep.
- **"Why pending?" → toil.** Self-service diagnosis instead of a ticket to the cluster team.
- **cgroup limits holding under pressure → CFS throttling / direct reclaim drill.** Only if they push on isolation failure modes.

---

## Do not say

- ❌ Any utilization percentage, or the "segregating prod and batch would cost N% more machines" figure. **Not verified — do not cite until read in the paper directly.**
- ❌ "Borg is the predecessor to Kubernetes" as the opening line. Everyone says it; it signals a blog post, not the paper.
- ❌ Present-tense claims about how Borg works *today*. The paper is 2015.
- ❌ Anything implying operational familiarity. Frame as reader-of-the-paper plus builder-of-a-toy. Nothing more.

---

## Boundaries — where to stop honestly

**If the follow-up goes to Borgmaster replication / Paxos:**

> "My consensus depth is in Raft — leader election plus a replicated log. I understand Borgmaster at that level rather than anything Paxos-specific."

Clean boundary. Costs nothing. Bluffing here costs a lot.

**If asked how reclamation estimates usage:** don't invent a formula. Say the paper describes a usage-based reservation that decays toward the request, and that the interesting part is what happens when the estimate is wrong — batch gets preempted, prod is protected.

**If asked "have you used Borg/Kubernetes at this scale":** no. Prometheus/Grafana via Helm, K8s objects and CNI at study depth, capsule at build depth. State it flatly and move.

---

## Pre-interview checklist

- [ ] **Read the actual paper end to end.** ~14 pages. Non-negotiable — one probing follow-up exposes summary-level knowledge.
- [ ] Verify the utilization figures in the paper; either add them to the fact sheet with real numbers or leave them permanently out.
- [ ] Ask the Borg-engineer referral what he'd probe on. Free calibration, and the connection is already on record.
- [ ] Confirm capsule is describable — what got working, what didn't. The whole differentiation rests on it. If mid-build, say so plainly.
- [ ] Rehearse the script out loud, timed. Cut to 60s. It reads shorter than it speaks.
