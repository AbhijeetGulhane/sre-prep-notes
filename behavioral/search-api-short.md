# search-api — Short Behavioral Answer

Use this when asked: "Tell me about a project you built" or "Walk me through
something you built end to end."

---

## The 60-Second Version

"I built a semantic search service in Python — FastAPI with a sentence-transformers
neural network model — that lets you search an SRE glossary using natural language.
So instead of keyword search, you can ask 'what stops cascading failures' and it
returns Circuit Breaker because it understands the meaning, not just the words.

The interesting part wasn't the model — it was everything around it. I deployed it
to Kubernetes with three replicas, wired up Prometheus metrics covering the four
golden signals, built a Grafana dashboard, ran a chaos experiment that measured a
36-second recovery time per pod failure, wrote a blameless postmortem, implemented
HPA that auto-scaled from 2 to 6 replicas under load, applied NetworkPolicy and
RBAC for least-privilege access, and provisioned a real EKS cluster on AWS using
Terraform — verified it, then destroyed it the same session.

The whole thing is on GitHub with a CI pipeline, design documentation, and two
postmortems with real timestamps and error budget calculations."

---

## The 30-Second Version

"I built a semantic search API in Python on Kubernetes — three replicas, Prometheus
metrics, Grafana dashboard, chaos testing with a measured 36-second recovery window,
HPA that scaled 2 to 6 replicas under load, and deployed to real AWS EKS via
Terraform. Full CI pipeline, two blameless postmortems with error budget math,
all on GitHub."

---

## One-Line Version

"A semantic search service on Kubernetes — Prometheus, Grafana, HPA, chaos testing,
EKS on Terraform — full reliability engineering stack."

---

## Follow-up Hooks (what each sentence opens up)

| What you say | Follow-up they'll ask |
|---|---|
| "semantic search... neural network" | How does it actually work? What model? |
| "four golden signals" | Walk me through them. Which metrics? |
| "36-second recovery time" | How did you measure that? What broke? |
| "blameless postmortem" | What was the root cause? What did you change? |
| "HPA scaled 2 to 6 replicas" | What triggered it? What did you observe? |
| "EKS on Terraform" | What did Terraform provision? What did it cost? |
| "RBAC and NetworkPolicy" | What permissions did Prometheus need? |

---

## Key Numbers to Drop Naturally

- Model: all-MiniLM-L6-v2, ~80MB, ~670MB RAM per pod
- Search latency: 300–500ms p99 (at the SLO boundary of <300ms)
- Probe latency: <10ms (pure in-memory)
- Recovery time: 36 seconds per pod failure (8s container + 28s model load)
- Error budget: 36s = 0.08% of monthly budget consumed
- HPA: 174% CPU under load, scaled 2→6 in ~60 seconds
- EKS: 15 Terraform resources, 18 minutes to provision, ~$0.50 total
- CI: 4 pytest tests, 1m40s, green on main
- Postmortems: 2, with real timestamps

---

## If They Ask "What Would You Do Differently?"

"Two things. First, the p99 latency is 300–500ms — right at the SLO boundary.
I'd add response caching for repeated queries or try a quantized model to get
under 100ms. Second, the memory limit is only 30MB above observed usage — too
tight. In production I'd set limits at p99 observed usage plus 50% headroom,
not just above idle."

This shows self-awareness and production thinking.
