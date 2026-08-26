# search-api

A semantic search service over an SRE glossary, built and operated as a
complete reliability engineering project for Google SRE interview preparation.

> Not just a model behind an API — this is about everything *around* the
> model: how it's monitored, how it scales, how it fails, and how it recovers.

---

## What It Does

Accepts a natural language query and returns the most semantically similar
SRE terms from a 15-term glossary, ranked by cosine similarity score.

```bash
curl "http://localhost:8000/search?q=what+stops+cascading+failures"
# → Circuit Breaker (0.553), Readiness Probe (0.290), OOM Killer (0.266)

curl "http://localhost:8000/search?q=repetitive+manual+work"
# → Toil (0.378), HorizontalPodAutoscaler (0.145), Circuit Breaker (0.140)
```

Keyword search would return nothing — these queries share no exact words with
the definitions. The semantic model understands meaning, not just keywords.

---

## Stack

| Layer | Technology |
|-------|-----------|
| API | FastAPI + uvicorn |
| ML model | sentence-transformers `all-MiniLM-L6-v2` |
| Metrics | Prometheus client (Counter + Histogram) |
| Observability | Grafana |
| Container | Docker (`python:3.10-slim`) |
| Orchestration | Kubernetes (Minikube + EKS) |
| Infrastructure | Terraform (VPC + EKS) |
| Testing | pytest + FastAPI TestClient |
| CI | GitHub Actions (pytest + ruff) |

---

## SLOs

| SLI | Target | Notes |
|-----|--------|-------|
| Availability (`/search` HTTP 200) | 99.5% over 30 days | ~43 min error budget/month |
| p99 latency (`/search`) | < 300ms | Currently 300–500ms — at boundary |
| Readiness (`/readyz`) | 99.9% over 30 days | ~4.3 min error budget/month |

---

## Running Locally

```bash
git clone git@github.com:AbhijeetGulhane/search-api.git
cd search-api
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --port 8000
# Model loads in ~30 seconds, then:
curl http://localhost:8000/healthz
curl http://localhost:8000/readyz
curl "http://localhost:8000/search?q=error+budget"
```

---

## Running on Minikube

```bash
# Start Minikube and point Docker at its daemon
minikube start --cpus=4 --memory=8192 --driver=docker
eval $(minikube docker-env)

# Build image inside Minikube
docker build -t search-api:v1 .

# Deploy
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Wait for all 3 pods to be Ready (~40 seconds)
kubectl get pods -n search-sre -w

# Port-forward and test
kubectl port-forward -n search-sre svc/search-api 8080:8080
curl http://localhost:8080/healthz
curl "http://localhost:8080/search?q=cascading+failures"
```

---

## Running on EKS (Terraform)

```bash
# Provision EKS cluster (~18 minutes)
cd terraform/
terraform init
terraform plan    # review: 15 resources
terraform apply

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name search-api

# Push image to ECR
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPO="$AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/search-api"
aws ecr create-repository --repository-name search-api --region us-east-1
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
docker tag search-api:v1 $ECR_REPO:v1
docker push $ECR_REPO:v1

# Deploy and verify
kubectl apply -f terraform/eks-deployment.yaml
kubectl get pods -n search-sre -w

# ⚠️ DESTROY when done — EKS costs ~$0.50/hr
terraform destroy
```

---

## Project Structure

```
search-api/
├── app/
│   ├── model.py        # SearchModel: SentenceTransformer + cosine similarity
│   ├── main.py         # FastAPI: /healthz /readyz /search /metrics endpoints
│   └── metrics.py      # Prometheus: REQUEST_COUNT counter + REQUEST_LATENCY histogram
├── data/
│   └── corpus.json     # 15 SRE term definitions (the search corpus)
├── tests/
│   └── test_api.py     # 4 pytest tests: healthz, readyz, Toil query, metrics format
├── k8s/
│   ├── namespace.yaml        # search-sre namespace
│   ├── configmap.yaml        # MODEL_NAME env var
│   ├── deployment.yaml       # 3 replicas, readiness/liveness probes, resource limits
│   ├── service.yaml          # ClusterIP: port 8080 → container 8000
│   ├── hpa.yaml              # HPA: minReplicas:2, maxReplicas:6, CPU target 50%
│   ├── networkpolicy.yaml    # Restrict ingress/egress to least privilege
│   ├── prometheus-rbac.yaml        # ServiceAccount + ClusterRole + ClusterRoleBinding
│   ├── prometheus-config.yaml      # kubernetes_sd_configs scrape config
│   ├── prometheus-deployment.yaml  # Prometheus in monitoring namespace
│   └── grafana-deployment.yaml     # Grafana dashboard in monitoring namespace
├── terraform/
│   ├── provider.tf           # AWS provider, us-east-1
│   ├── vpc.tf                # VPC + public subnets + IGW + route tables
│   ├── eks.tf                # IAM roles + EKS cluster + node group
│   └── eks-deployment.yaml   # K8s manifests for EKS (ECR image)
├── chaos/
│   └── chaos_test.sh         # Delete pod, record timestamps, measure recovery
├── load_test/
│   └── loadgen.py            # 10 async workers, 10 req/s each, 120s duration
├── docs/
│   ├── DESIGN.md             # Engineering decisions for all 14 days
│   ├── RBAC.md               # RBAC authorization flow + security analysis
│   ├── POSTMORTEM.md         # Postmortem #1: 36-second outage
│   └── POSTMORTEM2.md        # Postmortem #2: HPA load test + EKS deployment
├── .github/
│   └── workflows/
│       └── ci.yml            # GitHub Actions: pytest + ruff on push/PR
├── Dockerfile                # python:3.10-slim, layer-optimized build
└── requirements.txt
```

---

## Endpoints

| Endpoint | Type | Description |
|----------|------|-------------|
| `GET /healthz` | Liveness | Process alive check. Always 200 once server starts. |
| `GET /readyz` | Readiness | Model loaded check. 503 during load, 200 after. |
| `GET /search?q=...&top_k=N` | Search | Semantic search, returns top N results (default 3). |
| `GET /metrics` | Prometheus | Prometheus text format metrics. |
| `GET /docs` | Swagger UI | Auto-generated API documentation. |

---

## Reliability Engineering

### Architecture

```
                    ┌─────────────────────────────────────┐
                    │  Kubernetes (search-sre namespace)  │
                    │                                     │
curl → port-forward │  Service (ClusterIP :8080)          │
                    │         │                           │
                    │  ┌──────┴──────────────────────┐   │
                    │  │  Deployment (3 replicas)     │   │
                    │  │  ┌──────┐ ┌──────┐ ┌──────┐ │   │
                    │  │  │ Pod  │ │ Pod  │ │ Pod  │ │   │
                    │  │  │  ✅  │ │  ✅  │ │  ✅  │ │   │
                    │  └──└──────┘─└──────┘─└──────┘─┘   │
                    │                                     │
                    │  HPA: 2–6 replicas, CPU target 50%  │
                    │  NetworkPolicy: least privilege      │
                    │  ConfigMap: MODEL_NAME               │
                    └─────────────────────────────────────┘
                    ┌─────────────────────────────────────┐
                    │  monitoring namespace               │
                    │  Prometheus (RBAC service discovery)│
                    │  Grafana (4 golden signals dashboard)│
                    └─────────────────────────────────────┘
```

### Key Engineering Decisions

**Synchronous model loading:** Model loads in lifespan context before server
accepts connections. Background thread loading caused PyTorch segfaults under
concurrent load — synchronous loading is the correct pattern for K8s where
readiness probes handle the slow startup window.

**Histogram over summary for latency:** Histograms from multiple pods can be
aggregated with `histogram_quantile()`. Summaries cannot — they compute
percentiles in-process and produce non-aggregatable results across replicas.

**ConfigMap for MODEL\_NAME:** Changing the model requires only a ConfigMap
update and rolling restart — no image rebuild, no CI run. Infrastructure as
configuration, not code.

**3 replicas minimum:** Single-replica failure = 36-second outage (measured).
3 replicas mean pod failure is invisible to users (Postmortem #1 action item).

**HPA minReplicas: 2, not 3:** HPA reclaims one pod's resources (~500MB RAM)
during sustained low traffic while maintaining redundancy.

---

## Chaos Test Results

```bash
bash chaos/chaos_test.sh

# RESULTS
# Pod deleted:      09:35:05
# New pod appeared: 09:35:13  (+8s  container creation)
# Pod ready (1/1):  09:35:41  (+36s total outage window)
```

Error budget impact: 36s ÷ 2,592,000s/month = **0.08% of monthly error budget**.

---

## HPA Load Test Results

| Event | CPU | Replicas |
|-------|-----|----------|
| Idle | 1% | 2 |
| Load test start | 174% | 2 |
| First scale-up | 174% | 4 |
| Max scale | 200% | 6 |
| Load ends | 15% | 6 |
| Scale-down | 1% | 4 |

Scale-up: 2→6 in ~60 seconds. Scale-down: 1 pod per 30s after 60s stabilization.

---

## Cloud Deployment (EKS via Terraform)

```
terraform/
├── provider.tf   # AWS provider, us-east-1
├── vpc.tf        # VPC + 2 public subnets across 2 AZs + IGW + route tables
└── eks.tf        # IAM roles + EKS control plane v1.31 + t3.medium node group
```

**Results:**
- 15 Terraform resources provisioned in 18 minutes
- All 3 pods reached 1/1 Ready at 2m11s on real EC2 nodes
- API verified: Circuit Breaker top result (score 0.553) on AWS
- Total cost: ~$0.50 — cluster destroyed same session

---

## CI/CD

![CI](https://github.com/AbhijeetGulhane/search-api/actions/workflows/ci.yml/badge.svg)

Two jobs on every push and PR to `main`:
- **Test:** `python -m pytest tests/ -v` — 4 tests, 1m 40s
- **Lint:** `ruff check app/ --ignore E501` — import sorting + style

---

## Postmortems

- [Postmortem #1 — Single Replica Pod Crash](docs/POSTMORTEM.md)
  36-second outage. Root cause: single replica + 28-second model load.
  Fixed by scaling to 3 replicas (Day 9).

- [Postmortem #2 — HPA Load Test + EKS Deployment](docs/POSTMORTEM2.md)
  HPA: 2→6 replicas in 60s under 174% CPU load.
  EKS: 15 Terraform resources in 18 min, pods Ready at 2m11s, cost ~$0.50.

---

## Test Suite

```bash
source .venv/bin/activate
python -m pytest tests/ -v

# test_healthz        PASSED
# test_readyz         PASSED
# test_search_toil    PASSED  (Toil is top result, score > 0.3)
# test_metrics_format PASSED
# 4 passed in 6.42s
```

---

## Observability

- [x] Day 10 — Prometheus scraping pod metrics via RBAC
- [x] Day 11 — Grafana dashboard: request rate, p99 latency, error rate
- [x] Day 12 — NetworkPolicy (restrict traffic to namespace) + RBAC validation
- [x] Day 13 — HPA: auto-scale on CPU, load test with `load_test/loadgen.py`
- [x] Day 14 — GitHub Actions CI, Postmortem #2, tag v1.0

---

## Known Limitations

- **p99 latency 300–500ms** — at the SLO boundary. Cause: `all-MiniLM-L6-v2`
  inference on CPU without batching. Fix: smaller model, GPU, or response caching.
- **Model reloads on every pod start** — 30-second readiness gap per restart.
  Fix: cache model weights in an emptyDir volume (future action item).
- **WSL2/Minikube requires session restore** — run `mk-restart` after laptop
  restart to rebuild image and redeploy.
- **HPA not applied to EKS deployment** — load test was run on Minikube.
  EKS deployment verified functional; HPA on EKS is future work.

---

## Resume Bullets

- Provisioned production EKS cluster via Terraform (VPC, IAM, node group) in us-east-1; deployed and verified 3-replica service on real EC2 nodes
- Built chaos engineering test measuring 36-second pod recovery time, driving scale-to-3 architectural decision documented in blameless postmortem
- Implemented Four Golden Signals (Traffic, Latency, Errors, Saturation) in Prometheus + Grafana with RBAC-scoped pod discovery and NetworkPolicy enforcement
- Designed HPA configuration scaling 2→6 replicas in 60 seconds under load; load tested with 10-worker async generator achieving ~100 req/s
- Automated CI pipeline (GitHub Actions) running pytest + ruff on every push; all 4 tests green
