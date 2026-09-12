# search-api — Complete Interview Reference

**For behavioral and technical interviews. Every detail, every number, every decision.**

---

## 1. The One-Paragraph Answer

"I built a semantic search service in Python — FastAPI with a sentence-transformers
model — that lets you search an SRE glossary using natural language. The interesting
part isn't the model; it's everything around it. I containerized it, deployed it to
Kubernetes with three replicas, wired up Prometheus metrics and a Grafana dashboard,
ran chaos experiments measuring exact recovery times, wrote blameless postmortems,
implemented HPA that scaled from 2 to 6 replicas under load, applied NetworkPolicy
and RBAC for least-privilege access, and provisioned a real EKS cluster on AWS using
Terraform — then tore it down the same session. The whole thing is documented with
design decisions, two postmortems, and a CI pipeline that's green."

Use this as your opening. Then answer whatever specific follow-up they ask.

---

## 2. End-to-End Request Flow

**Q: What does it actually do? Request in, what happens, response out?**

```
User: curl "http://localhost:8080/search?q=what+stops+cascading+failures"

1. curl → TCP connection to localhost:8080
   ↓
2. kubectl port-forward (or kube-proxy on EKS) receives connection
   Routes to one of 3 search-api pod IPs (round-robin via iptables DNAT)
   ↓
3. uvicorn receives HTTP GET /search?q=what+stops+cascading+failures
   FastAPI parses query parameter, validates: min_length=1, max_length=200
   ↓
4. @track("/search") decorator starts timer, records request start
   ↓
5. search() function called:
   a. Check _model is not None (503 if model not loaded)
   b. Call _model.search("what stops cascading failures", top_k=3)
   ↓
6. Inside SearchModel.search():
   a. model.encode(query) → runs neural network inference
      Input: "what stops cascading failures" (5 tokens)
      Output: 384-dimensional float32 vector
      Time: ~200-400ms on CPU
   b. Cosine similarity between query vector and 15 pre-computed corpus vectors
      scores = dot(embeddings, query_vec) / (norm(embeddings) × norm(query_vec))
      Result: 15 similarity scores between -1 and 1
   c. argsort(scores)[::-1][:3] → top 3 indices by descending score
   ↓
7. Returns list of {term, definition, score} for top 3 matches:
   - Circuit Breaker: 0.553
   - Readiness Probe: 0.290
   - OOM Killer:      0.266
   ↓
8. FastAPI serializes to JSON, uvicorn sends HTTP 200 response
   ↓
9. @track decorator records:
   - REQUEST_COUNT{endpoint="/search", status="200"}.inc()
   - REQUEST_LATENCY{endpoint="/search"}.observe(elapsed_seconds)
   ↓
Response: {"query": "...", "top_k": 3, "results": [...]}
Total time: ~300-500ms
```

**Why no exact keyword matches?** "Cascading failures" doesn't appear in the
Circuit Breaker definition. The model converts both the query and the definition
to vectors in a 384-dimensional semantic space where similar meanings are
geometrically close. Keyword search would return zero results. Semantic search
returns the right answer.

---

## 3. The Five Endpoints

| Endpoint | Method | Purpose | Returns |
|----------|--------|---------|---------|
| `/healthz` | GET | Liveness probe | `{"status": "ok", "uptime_seconds": N}` |
| `/readyz` | GET | Readiness probe | `{"status": "ready", "model": "all-MiniLM-L6-v2"}` or 503 |
| `/search` | GET | Semantic search | `{"query": "...", "top_k": 3, "results": [...]}` |
| `/metrics` | GET | Prometheus scrape | Prometheus text format |
| `/docs` | GET | Swagger UI | Auto-generated API documentation |

**The /healthz vs /readyz distinction is critical to know:**

`/healthz` returns 200 immediately when uvicorn starts — before the model loads.
`/readyz` returns 503 until the model finishes loading (~30 seconds), then 200.

Why this matters: Kubernetes uses /healthz for liveness (should I restart this?)
and /readyz for readiness (should I send traffic here?). Without separate probes,
you'd either restart the pod unnecessarily during model loading, or send traffic
to a pod that can't serve it yet.

---

## 4. The Model — What, Why, How

**Model:** `all-MiniLM-L6-v2` from HuggingFace sentence-transformers

**What it does:** Converts any text into a 384-dimensional vector (a list of 384
floating-point numbers). Texts with similar meanings produce vectors that point
in similar directions — this is what "semantic" means.

**Why this model specifically:**
- Size: ~80MB download, ~500MB RAM when loaded — fits in K8s memory limits
- Speed: ~200-400ms per inference on CPU without a GPU
- Accuracy: Good semantic understanding of technical vocabulary
- No infrastructure: runs in-process, no vector database, no GPU required

**Corpus embeddings — the key optimization:**
At startup, the model encodes all 15 SRE definitions once and stores the results
as a 15×384 float32 matrix (~23KB). Every search query only needs to encode the
query itself (~200-400ms), not the whole corpus. Without pre-computation, each
search would take 15x longer.

**Why cosine similarity, not dot product:**
Cosine similarity measures the angle between vectors, not their magnitude.
This makes it scale-invariant — a short query can still match a long definition
if they point in the same semantic direction. Dot product would favor longer texts
with larger magnitude vectors.

**Score interpretation:**
- >0.5: strong semantic match (Circuit Breaker for "cascading failures")
- 0.2-0.5: related concept
- <0.2: weak or unrelated
- Real observed scores: 0.553 (Circuit Breaker), 0.378 (Toil), 0.585 (OOM Killer)

---

## 5. The Code — File by File

### `data/corpus.json`
15 SRE term definitions as JSON objects with id, term, definition.
The "database" the search runs against. Static — loaded once at startup.
Terms: SLO, SLI, Error Budget, Toil, Postmortem, Circuit Breaker, Four Golden
Signals, OOM Killer, CFS Scheduler, PDB, HPA, etcd, Readiness Probe, USE Method, mTLS.

### `app/model.py` — SearchModel class

```python
class SearchModel:
    def __init__(self):
        self.model = SentenceTransformer(MODEL_NAME)  # load the neural network
        self.corpus = self._load_corpus()              # read the 15 definitions
        self.embeddings = self._embed_corpus()         # pre-compute all 15 vectors

    def search(self, query: str, top_k: int = 3) -> list[dict]:
        query_embedding = self.model.encode(query)     # encode the query
        scores = cosine_similarity(self.embeddings, query_embedding)
        top_indices = argsort(scores)[::-1][:top_k]
        return [{term, definition, score} for i in top_indices]
```

`MODEL_NAME` comes from `os.getenv("MODEL_NAME", "all-MiniLM-L6-v2")` — reads
from environment variable, defaulting to the standard model. In K8s, the ConfigMap
sets this value.

**Thread safety issue we hit and fixed:** First version loaded the model in a
background thread so the server started immediately. When a search request arrived
during loading, two threads called PyTorch simultaneously → segfault in the C++
layer → WSL2 crashed. Fixed: synchronous loading in the lifespan context. Server
blocks until model is ready, then starts accepting connections.

### `app/main.py` — The FastAPI Application

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _model
    _model = SearchModel()   # loads synchronously before accepting any requests
    yield
    # shutdown

app = FastAPI(lifespan=lifespan)

@app.get("/healthz")
def healthz():
    return {"status": "ok", "uptime_seconds": ...}

@app.get("/readyz")
@track("/readyz")
def readyz():
    if _model is None:
        raise HTTPException(503)
    return {"status": "ready", "model": MODEL_NAME}

@app.get("/search")
@track("/search")
def search(q: str = Query(..., min_length=1, max_length=200)):
    if _model is None:
        raise HTTPException(503)
    return {"query": q, "results": _model.search(q)}
```

**Why lifespan, not @app.on_event("startup"):** `on_event` is deprecated in
newer FastAPI. `lifespan` is the current recommended pattern — uses Python's
async context manager protocol.

### `app/metrics.py` — Prometheus Instrumentation

```python
REQUEST_COUNT = Counter(
    "search_api_requests_total",
    "Total requests by endpoint and status",
    ["endpoint", "status"]
)

REQUEST_LATENCY = Histogram(
    "search_api_request_latency_seconds",
    "Request latency by endpoint",
    ["endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.2, 0.3, 0.5, 1.0, 2.0, 5.0]
)

def track(endpoint: str):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            status = "200"
            try:
                return func(*args, **kwargs)
            except Exception as e:
                status = str(getattr(e, "status_code", "500"))
                raise
            finally:
                REQUEST_COUNT.labels(endpoint=endpoint, status=status).inc()
                REQUEST_LATENCY.labels(endpoint=endpoint).observe(time.time() - start)
        return wrapper
    return decorator
```

**Why a histogram, not a summary:**
Histograms store counts per bucket — additive across replicas. You can sum
histograms from 3 pods and compute accurate fleet-wide p99.
Summaries compute percentiles inside each process — cannot be meaningfully
aggregated. `histogram_quantile(0.99, sum by (le) (rate(...)))` works because
we used a histogram.

**Why these specific bucket boundaries:**
`[0.01, 0.05, 0.1, 0.2, 0.3, 0.5, 1.0, 2.0, 5.0]` — buckets at 0.2 and 0.3
bracket the SLO target of <300ms p99. We can see exactly what fraction of
requests land in the 200-300ms range.

---

## 6. The Latency Numbers

**Q: Do you have latency numbers?**

Yes — from the Prometheus metrics and Grafana dashboard.

| Endpoint | p50 | p99 | Source |
|----------|-----|-----|--------|
| `/healthz` | <10ms | <10ms | Pure in-memory, no compute |
| `/readyz` | <10ms | <10ms | Pure in-memory, no compute |
| `/search` | ~350ms | ~450-500ms | Neural network inference on CPU |

**Observed from Grafana p99 panel:**
- Probe traffic (/healthz, /readyz): flat ~10ms line
- Search queries: 300-500ms range

**The SLO says <300ms p99.** We're at 300-500ms — right at the boundary.
This is a real operational concern documented in the Known Limitations section:
the model runs inference on CPU without batching. Fix paths: smaller model
(all-MiniLM-L6-v2 is already small), GPU inference, or response caching for
repeated queries.

**From the HPA load test:**
Under 100 req/s load (10 workers × 10 req/s), CPU hit 174% of the 250m request
per pod — confirming CPU is the bottleneck, not memory or I/O.

---

## 7. What Breaks It

**Q: What breaks it? What are the failure modes?**

### Cold Start / Model Load Time
**The dominant failure mode.** Each pod takes ~28-30 seconds to become ready
after starting. This is the model loading time — SentenceTransformer downloads
and initializes the PyTorch model, then encodes all 15 corpus definitions.

Measured from the Day 7 chaos test:
- Container creation: 8 seconds
- Model loading + readiness: 28 seconds
- Total outage per pod failure (single replica): 36 seconds

**Why 36 seconds = 0.08% of monthly error budget:**
36s ÷ 2,592,000s/month = 0.000014 = 0.0014% per incident.
Against 99.5% SLO: 43,200s budget/month. 36s = 0.08% of budget consumed.

**Fix:** Pre-bake model weights into the Docker image, or cache in an emptyDir
volume on the node. This eliminates the download and reduces load time to ~10s.

### Memory — At the Boundary
Each pod uses ~670MB RAM at idle (model weights in memory). Memory limit is 700Mi.
The gap is ~30MB. Under load with large request payloads, this could trigger OOM.

Symptoms: pod exits with code 137, kubectl describe shows OOMKilled.
Fix: increase memory limit, or use a quantized/smaller model.

### CPU Throttling
CPU request is 250m, limit is 1000m. Under sustained load, the CFS scheduler
throttles the container when it exceeds 1000m in a 100ms window. This causes
latency spikes (not failures) because the process gets pulled off the CPU queue.

Symptom: p99 latency spikes while CPU average looks fine.
Distinguishing: `container_cpu_cfs_throttled_seconds_total` climbing in Prometheus.

### Thread Safety (Caught and Fixed)
PyTorch is not thread-safe when called from multiple threads simultaneously.
Original implementation loaded model in a background thread — any search request
during loading caused a segfault. Fixed by loading synchronously before server
starts.

### Minikube/WSL2 Specific
`python3 -m json.tool` piped after a curl command caused WSL2 to crash — the
pipe spawned a subprocess that shared PyTorch's loaded memory space and segfaulted.
Fix: write curl output to a file (`-o /tmp/result.json`) and cat separately.
Not a production concern — only affects local dev on WSL2.

---

## 8. Kubernetes Setup — Every Detail

**Q: Is it running anywhere, or local only?**

Both. Minikube locally for development, EKS on AWS for the production verification.

### Current State (Minikube)

```
Namespace: search-sre
Deployment: search-api
  replicas: 3 (was 1, scaled after Postmortem #1)
  image: search-api:v1 (built into Minikube's local Docker daemon)
  imagePullPolicy: Never

Resources per pod:
  requests: cpu=250m, memory=512Mi
  limits:   cpu=1000m, memory=700Mi

Probes:
  readiness: GET /readyz, initialDelaySeconds=30, periodSeconds=10, failureThreshold=6
  liveness:  GET /healthz, initialDelaySeconds=60, periodSeconds=15, failureThreshold=3

Env:
  MODEL_NAME=all-MiniLM-L6-v2  (from ConfigMap)
  OMP_NUM_THREADS=1             (prevents PyTorch thread pool crash in WSL2)
  TOKENIZERS_PARALLELISM=false  (prevents HuggingFace tokenizer parallelism crash)

Service: ClusterIP, port 8080 → targetPort 8000
```

### Why 3 Replicas (Not 1, Not 2)?

Single replica: any pod failure = 36-second complete outage (measured, Postmortem #1).
Two replicas: rolling update drops to 1 replica temporarily (0% headroom).
Three replicas: tolerate 1 pod failure (2 remain serving) AND a rolling update
(1 unavailable at a time, 2 still serving). Minimum for meaningful HA.

### Why These Resource Numbers?

**250m CPU request:** Conservative. At idle, pod uses ~4m CPU (confirmed via
`kubectl top pods`). Request set higher because HPA uses requests as the baseline
for utilization calculation. 4m actual vs 250m request = 1.6% utilization at idle.

**512Mi memory request:** Observed memory at idle: ~670Mi (confirmed via
`docker stats`). Wait — request is lower than observed? Yes, intentionally.
Requests represent the scheduler's placement decision. The actual usage is higher
but within the limit. The pod is Burstable QoS class (request < limit) which
means it can be evicted under severe node pressure but has overhead headroom.

**1000m CPU limit:** 4× the request. Allows search-api to burst to 1 full CPU
core during inference without constant throttling. Under sustained load, the HPA
scales out (more pods) rather than relying on vertical limits.

**700Mi memory limit:** ~30Mi headroom above the ~670Mi observed at idle.
Tight. Should ideally be 750-800Mi. Under our current corpus and request sizes,
it hasn't OOMKilled — but it's a known risk.

### ConfigMap vs Secrets

`MODEL_NAME` is in a ConfigMap (`k8s/configmap.yaml`) because it's not sensitive.
Changing the model: update ConfigMap → rolling restart → all pods pick up new
value. No image rebuild, no CI run.

If we had API keys or database passwords, those would be Secrets. Key distinction:
Secrets are base64-encoded (NOT encrypted by default). Real security comes from
RBAC restricting who can `kubectl get secret` + optional etcd encryption at rest.

### HPA Configuration

```yaml
minReplicas: 2     # never below 2 (HA requirement)
maxReplicas: 6     # node capacity ceiling
target: 50% CPU   # 50% of 250m request = 125m per pod

scaleUp:
  stabilizationWindowSeconds: 0   # scale up immediately
  maxPodsPerStep: 2 per 15s

scaleDown:
  stabilizationWindowSeconds: 60  # wait 60s before scaling down
  maxPodsPerStep: 1 per 30s
```

**Why 50% target?** Leaves 2x headroom. If load suddenly doubles between HPA
sync cycles (15s), we're still within limits before the next scale-up.

**Why asymmetric windows?** Scale up fast (users are waiting), scale down slow
(prevent thrashing — a brief lull then another spike shouldn't cause immediate
scale-down then scale-up).

### NetworkPolicy

```yaml
ingress:
  - from same namespace (search-sre pods)
  - from monitoring namespace (Prometheus scraping)

egress:
  - DNS port 53 UDP+TCP (for name resolution)
  - HTTPS port 443 (for model download from HuggingFace)
  - to same namespace
```

Default Kubernetes behavior: all traffic allowed. NetworkPolicy changes search-api
to deny-by-default — only listed rules pass. Verified with `kubectl auth can-i`
for RBAC + confirmed Prometheus targets still UP (network layer).

---

## 9. Observability — The Full Stack

**Q: Have you instrumented anything with Prometheus?**

Yes — complete observability stack:

### What's Instrumented

**Two Prometheus metrics:**

```
search_api_requests_total{endpoint, status}  — Counter
search_api_request_latency_seconds_bucket{endpoint, le}  — Histogram
```

**Four Golden Signals coverage:**
- Traffic: `rate(search_api_requests_total[1m])` per endpoint
- Latency: `histogram_quantile(0.99, sum by (le) (rate(..._bucket[1m])))`
- Errors: `rate(search_api_requests_total{status=~"5.."}[1m])`
- Saturation: CPU/memory from K8s cAdvisor (not in app code)

### What Prometheus Sees

From the Day 8 verification on the running K8s cluster:

```
search_api_requests_total{endpoint="/readyz", status="200"} 2879.0
search_api_requests_total{endpoint="/healthz", status="200"} 1918.0
search_api_requests_total{endpoint="/search", status="200"} 2.0
```

2879 readiness probe hits × 10s interval = ~28,790s = ~8 hours uptime.
1918 liveness probe hits × 15s interval = ~28,770s. Matches — confirms timing.

**Key insight:** Probe traffic dominates over user traffic on a low-traffic service.
This is realistic — in production, health probes + monitoring scrapes often
generate more requests than actual users.

Search p99 latency from histogram buckets:
```
search_api_request_latency_seconds_bucket{endpoint="/search", le="0.3"} 0.0
search_api_request_latency_seconds_bucket{endpoint="/search", le="0.5"} 2.0
```
Both search requests completed between 300ms and 500ms. Confirms at-boundary SLO.

### How Prometheus Discovers Pods (RBAC + Service Discovery)

```yaml
kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [search-sre]

relabel_configs:
  - keep pods with label app=search-api
  - override address to :8000
  - add pod label to each metric series
  - add namespace label
```

Prometheus calls the K8s API every 15 seconds to get current pod IPs. This
requires a ServiceAccount with ClusterRole granting `list`/`watch` on pods.
The `pod` label added by relabel_configs makes each pod's metrics separately
queryable — essential for per-pod debugging.

**RBAC verification:**
```
kubectl auth can-i list pods   --as=system:serviceaccount:monitoring:prometheus → yes
kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:prometheus → no
```

### Grafana Dashboard — 3 Panels

Prometheus datasource: `http://prometheus.monitoring.svc.cluster.local:9090`
(in-cluster DNS, not localhost — Grafana is a separate pod)

**Panel 1 — Request Rate:**
`rate(search_api_requests_total[1m])`
Shows per-pod, per-endpoint time series. Confirmed: 3 separate lines for
/healthz (one per pod), each at ~0.06-0.12 req/s from kubelet probes.

**Panel 2 — p99 Latency:**
`histogram_quantile(0.99, sum by (le) (rate(search_api_request_latency_seconds_bucket[1m])))`
Shows ~10ms for probe traffic. Search queries would show 300-500ms.
Threshold line at 0.3s (SLO boundary) — violations immediately visible.

**Panel 3 — Error Rate:**
`rate(search_api_requests_total{status=~"5.."}[1m])`
Showed "No data" — zero 5xx errors. Correct state.

---

## 10. Chaos Engineering

**Q: Have you actually tested failure?**

Yes — `chaos/chaos_test.sh` deletes the running pod and records exact timestamps.

**Results (Jul 15, 2026):**
```
Pod deleted:      09:35:05
New pod appeared: 09:35:13   (+8 seconds — container creation)
Pod ready (1/1):  09:35:41   (+36 seconds total outage window)
```

**Breaking down the 36 seconds:**
- 8 seconds: Deployment controller detects pod count dropped → creates replacement →
  container image already on node → container starts
- 28 seconds: model loading — SentenceTransformer loads weights + encodes 15 corpus
  definitions before readiness probe passes

**Model loading dominates (78% of outage window).** This insight drove the
Day 9 decision to scale to 3 replicas — with 3 replicas, any single pod failure
is invisible to users because 2 pods remain serving. The 36-second gap still
exists, but it's now a pod replacement event rather than a service outage.

**Postmortem #1 error budget math:**
36s outage ÷ 2,592,000s/month = 0.0014% consumed.
Monthly SLO budget (99.5%): 43,200s = 0.08% of budget per incident.

---

## 11. HPA Load Test

**What we ran:**
10 async workers, each sending 10 requests/second = ~100 req/s total.
Duration: 120 seconds. Queries rotated through 10 different search terms.

**What happened:**

| Time | CPU | Replicas | Event |
|------|-----|----------|-------|
| T+0 | 1% | 2 | Idle — HPA at minReplicas |
| T+0 | 174% | 2 | Load test starts — 100 req/s |
| T+60s | 174% | 4 | HPA fires: ceil(2 × 174/50) = 7, capped policy 2 per step |
| T+75s | 200% | 6 | Second step: maxReplicas hit |
| T+120s | 15% | 6 | Load test ends |
| T+180s | 1% | 5 | 60s stabilization elapsed, scale-down begins |
| T+210s | 1% | 4 | Continuing: 1 pod per 30s |

**Why CPU spiked to 200% at 6 replicas:**
Pods 5 and 6 were starting their model load (~28s) while pods 1-4 served all
traffic. New pods consumed CPU for initialization but served zero requests.
Average inflated temporarily. Resolved when all 6 passed readiness.

**HPA formula applied:**
`desiredReplicas = ceil(currentReplicas × (actual / target))`
`= ceil(2 × (174 / 50)) = ceil(6.96) = 7`
Capped to maxReplicas=6. Scale-up policy limited first step to +2 pods (→4),
then +2 more (→6).

---

## 12. EKS + Terraform

**Q: Is it running anywhere, or local only?**

Both. Local (Minikube) for all development. Production verification on AWS EKS.

### What Terraform Provisioned

15 resources:
```
VPC:           10.0.0.0/16, DNS enabled
Subnets:       2× public, /24 each, across us-east-1a and us-east-1b
IGW:           internet access for public subnets (no NAT Gateway — saves cost)
Route tables:  0.0.0.0/0 → IGW, associated with both subnets
IAM:           cluster role (AmazonEKSClusterPolicy)
               node role (EKSWorkerNodePolicy + EKS_CNI_Policy + ECRReadOnly)
EKS cluster:   v1.31, public endpoint, search-api
Node group:    2× t3.medium (2 vCPU, 4GB), managed by AWS
```

`terraform plan` showed 15 resources, 0 errors before any money was spent.
`terraform apply` took 18 minutes (control plane provisioning is the slow part).

### What Was Deployed on EKS

Same K8s manifests as Minikube, two changes:
- `image:` → ECR URI (`<account>.dkr.ecr.us-east-1.amazonaws.com/search-api:v1`)
- `imagePullPolicy: IfNotPresent` (not `Never` — nodes pull from ECR)

**Results:**
- All 3 pods: 1/1 Ready at 2m11s (vs 42s on Minikube — extra ~90s for ECR pull)
- API verification: Circuit Breaker returned, score 0.553 (identical to local)
- Same model, same code, same result — confirmed portability

**Why EKS pods took longer:**
On Minikube: image already inside the daemon, zero pull time.
On EKS: 2 t3.medium nodes had to pull the ~5GB image from ECR over the AWS
network. After first pull, subsequent restarts would be faster (image cached on node).

**Cost:**
~$0.50 for the session (~15 minutes of cluster time).
`terraform destroy` immediately after verification — zero ongoing charges.
ECR repository also deleted to stop storage charges (~$0.017/day for 5GB).

---

## 13. CI Pipeline

GitHub Actions on every push and PR to main. Two jobs:

**Test (1m 40s):**
- Python 3.10 setup
- pip cache keyed on `requirements.txt` hash (avoids re-downloading PyTorch)
- `python -m pytest tests/ -v`
- 4 tests must pass

**Lint (6s):**
- `ruff check app/ --ignore E501`
- Checks import sorting, unused imports, style violations
- Failed first run (imports not sorted) — fixed with `ruff --fix`

**Green badge** in README confirms current state of main branch.

---

## 14. The Numbers to Know Cold

Memorize these — they come up in every follow-up question:

| Metric | Value | Context |
|--------|-------|---------|
| Model size | ~80MB | Downloaded, cached in `~/.cache/huggingface` |
| RAM per pod at idle | ~670MB | Confirmed via `docker stats` and `kubectl top pods` |
| Model load time | ~28-30 seconds | Until readiness probe passes |
| Container creation time | 8 seconds | From pod delete to new pod Running |
| Total recovery time (1 replica) | 36 seconds | Measured in chaos test |
| Error budget consumed per incident | 0.08% | 36s ÷ 43,200s monthly budget |
| Search p50 latency | ~350ms | CPU inference |
| Search p99 latency | ~450-500ms | CPU inference |
| Probe latency | <10ms | Pure in-memory |
| HPA scale-up time | ~60 seconds | 2 → 6 replicas in 2 steps |
| CPU under load | 174% of request | 100 req/s across 2 pods |
| Terraform apply time | 18 minutes | EKS control plane dominates |
| EKS pods ready | 2m11s | Includes ECR image pull |
| AWS session cost | ~$0.50 | 15 minutes of EKS |
| Number of corpus terms | 15 | SRE glossary |
| Embedding dimensions | 384 | all-MiniLM-L6-v2 output |
| pytest suite | 4 tests, 6.42 seconds | All passing |
| Prometheus scrape interval | 15 seconds | Default |

---

## 15. STAR Stories — Ready to Tell

### Story 1 — Chaos Engineering and Postmortem #1

**Situation:** Deployed search-api as a single-replica Kubernetes service on Minikube.
Wanted to understand the actual blast radius of a pod failure before scaling.

**Task:** Measure the real recovery time from a pod crash, identify the dominant
failure mode, and drive architectural improvements from data.

**Action:** Wrote `chaos/chaos_test.sh` that deletes the running pod, records exact
timestamps for pod deletion, new pod appearance, and ready state, then verifies
service recovery. Found: 8 seconds container creation + 28 seconds model loading
= 36 seconds complete service unavailability per pod failure. Identified model
loading as 78% of the outage window. Wrote a blameless postmortem with timeline,
impact, root cause, and four specific action items.

**Result:** Drove the Day 9 decision to scale to 3 replicas — with 3 replicas,
any single pod failure is invisible to users. Documented that a 36-second outage
consumes 0.08% of the monthly error budget, giving concrete data for future
capacity decisions.

---

### Story 2 — HPA Load Test

**Situation:** search-api was deployed with 2 replicas (HPA minimum) in Kubernetes.
Needed to verify the autoscaling configuration would respond correctly under real load.

**Task:** Generate enough traffic to trigger HPA scale-up, observe the scaling behavior,
and document any unexpected behavior for the postmortem.

**Action:** Wrote a 10-worker async load generator in Python (httpx + asyncio)
sending ~100 req/s total. Watched `kubectl get hpa -w` in real time. Observed:
CPU spiked to 174% of the 250m request per pod. HPA scaled 2→4→6 replicas in two
steps over ~60 seconds. Noticed CPU briefly hit 200% at 6 replicas — traced to
new pods consuming CPU for model loading before serving traffic, temporarily inflating
the per-pod average.

**Result:** Confirmed HPA scales correctly under load. Documented the 200% spike
as expected behavior (not a bug — readiness probe correctly prevented traffic routing
to unready pods). Recorded scale-down: 60-second stabilization window then 1 pod
per 30 seconds.

---

### Story 3 — EKS Production Deployment

**Situation:** search-api was running on Minikube locally. The production gap was
that no real cloud deployment had been done — 11+ years of ops experience but no
recent hands-on cloud infrastructure provisioning.

**Task:** Provision a real EKS cluster, deploy search-api with the same manifests
used locally, verify end-to-end functionality on real cloud hardware, and destroy
cleanly to minimize cost.

**Action:** Wrote Terraform covering VPC with public subnets across 2 AZs, internet
gateway, route tables, IAM roles for the cluster and nodes, EKS control plane, and
a managed node group with t3.medium instances. Built the Docker image and pushed to
ECR. Updated K8s manifests to reference the ECR image URI and set imagePullPolicy
to IfNotPresent. Applied manifests to EKS. Port-forwarded and hit the search API.

**Result:** 15 Terraform resources provisioned in 18 minutes. All 3 pods reached
1/1 Ready at 2m11s on real EC2 nodes. Search returned Circuit Breaker as top result
(score 0.553) — identical to local results, confirming the stack is portable.
Total AWS cost: ~$0.50. Terraform destroy completed same session. Zero ongoing charges.

---

## 16. Hard Questions and Answers

**"Why not use a proper vector database like Pinecone or Weaviate?"**
15 items in the corpus. A vector database is the right answer at 10,000+ items
where the in-memory O(n) cosine similarity becomes slow, or when you need
persistence and real-time updates. For 15 static SRE definitions, a numpy matrix
multiply is faster, simpler, and has zero operational overhead. The model wrapper
(SearchModel) is abstracted enough that swapping in a vector database client would
be a one-file change.

**"How would you scale this to 10 million documents?"**
The current approach breaks at scale: 10M × 384 dimensions × 4 bytes = 15GB matrix
in RAM per pod, and O(n) similarity becomes ~seconds per query. The production path:
FAISS or HNSW index for approximate nearest neighbor search (O(log n)), chunked
batch encoding for ingestion, and a dedicated vector store (Weaviate, Qdrant, or
Pinecone). The API and probe design would be identical.

**"The p99 is at your SLO boundary. What would you do in production?"**
Three options in priority order: (1) Response caching — identical queries return
cached results without model inference, effectively reducing p99 to <10ms for
repeated queries. (2) Model optimization — quantized version of all-MiniLM-L6-v2
(int8 quantization, ~2× faster inference, ~5% accuracy loss). (3) GPU inference
— p99 drops to <50ms, but adds GPU node cost and complexity.

**"Your memory limit is only 30MB above observed usage. Isn't that risky?"**
Yes, it's tight. The correct value is 750-800Mi. I set 700Mi during initial
deployment and it hasn't caused OOM kills in testing, but under a traffic spike
with large request payloads, it could trigger OOMKilled. This is in the Known
Limitations section. In production I'd set the limit at p99 observed usage + 50%
headroom, not just above idle.

**"You said model loading takes 28 seconds. How does Kubernetes handle that?"**
The readiness probe with `initialDelaySeconds: 30` tells Kubernetes not to even
start checking until 30 seconds after container start. During this window, the pod
IP is not in the Service's Endpoints — no traffic routes here. After 30 seconds,
the probe starts checking `/readyz` every 10 seconds. `/readyz` returns 503 until
`_model` is not None, then 200. Once 200 is returned, the pod IP enters Endpoints
and traffic starts routing. This is exactly what readiness probes are designed for.

**"How did you pick 50% CPU utilization as the HPA target?"**
50% leaves a 2x buffer for traffic spikes between HPA evaluation cycles (15 seconds).
If the target were 90%, a sudden 2x traffic spike could push utilization to 180%
before the next scale-up fires — during those 15 seconds, the service would be
overloaded. At 50%, a 2x spike hits 100% — high but manageable — before the
next cycle adds capacity.

**"What would happen if you forgot to run terraform destroy?"**
EKS control plane: $0.10/hr = $2.40/day = $72/month.
Two t3.medium nodes: ~$0.083/hr combined = ~$2/day = ~$60/month.
Total: ~$4.40/day, ~$132/month if left running indefinitely. This is why I
added a cost reminder as a Terraform output and ran destroy immediately after
verification.

**"What's the difference between what you built and a real production system?"**
Honest gaps: (1) No TLS on the Service — production would have an Ingress with
cert-manager for TLS termination. (2) No persistent storage for Prometheus/Grafana
— dashboards lost on pod restart. (3) Model cache not shared — each pod downloads
independently on start. (4) No alerting rules in Prometheus — dashboard only, no
PagerDuty integration. (5) NetworkPolicy may not enforce on Minikube's CNI. (6)
Single cluster, single region — no disaster recovery. These are documented in
Known Limitations and are reasonable tradeoffs for a development environment.
