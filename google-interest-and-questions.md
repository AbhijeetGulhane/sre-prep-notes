# Google Interest + Questions for Interviewers

Two checklist items the recruiter named: a Google system you admire technically, and 2-3 real questions per session.

---

## A Google system I admire (technical, ~60 sec)

Pick ONE and go deep. Two options grounded in your infra background:

### Option A — Borg (the one to use)
Best fit — it's the ancestor of Kubernetes and directly relevant to your 60k-node estate experience.

> "Borg — because it solved cluster management at a scale I've felt the pain of from the other side. Running sixty thousand nodes, the hard part isn't any single machine, it's bin-packing thousands of heterogeneous workloads onto a shared fleet while respecting priorities and failure domains, and doing it so a machine failure is a non-event. What I admire technically is the priority/preemption model and how it drives utilization — batch jobs soaking up headroom that production leaves idle, then getting preempted cleanly when production needs it back. Kubernetes inherited the shape but Borg had to make it survive Google's scale first. Coming from managing a large estate largely by tooling-on-top, the idea of that being the *substrate* is what pulls me toward SRE here."

**Why it lands:** shows you understand scheduling/utilization/failure-domains (real SRE substance), ties to your actual experience, and connects to the role.

### Option B — Spanner
Use if the conversation goes toward data/consistency.

> "Spanner — specifically TrueTime. Everyone's told you can't have consistency and availability across global distance, and Spanner's answer was to attack the assumption underneath: make time itself a bounded-uncertainty primitive with atomic clocks and GPS, then *wait out* the uncertainty window on commit. Trading a few milliseconds of commit latency for externally-consistent global transactions is a genuinely clever engineering tradeoff — solving a distributed-systems problem with a hardware/physics move instead of a software-only one."

**Why it lands:** shows distributed-systems depth (CAP, consistency) and appreciation for non-obvious tradeoffs.

**Delivery:** know ONE cold. Don't recite both. If they push for detail, have one more layer ready (Borg → alloc/quota; Spanner → Paxos groups under the hood).

---

## Questions for interviewers (2-3 per session, pick from these)

Have more than you need; read the room and pick.

### For the CODING interviewer (engineer)
- "What does on-call actually look like day-to-day for this team — what's the paging load, and how much of the week is interrupt-driven vs project work?"
- "What's the split between writing new tooling/automation versus operating existing systems on this team?"
- "What's a recent reliability problem the team solved that you found interesting?"
- "How does the team decide what to automate next — is there a systematic way you pick down toil?"

### For the BEHAVIORAL interviewer (often a manager)
- "How does this SRE org partner with the product/dev teams it supports — embedded, consulting, or something else?"
- "What does growth look like here from where I'd start — what separates a strong SRE from a great one on your team?"
- "What's the biggest reliability or scaling challenge the org is focused on over the next year?"
- "How does the team handle the balance between reliability work and shipping — who owns the error budget conversation?"

### Avoid
- Anything Googleable (comp bands, general perks, "what does Google do")
- Yes/no questions
- Questions that signal you haven't thought about the role

**Rule:** ask at least 2 per session. Going blank here reads as low interest — it's a cheap place to lose points and an easy place to show genuine engagement.
