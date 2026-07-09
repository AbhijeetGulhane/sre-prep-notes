# gRPC vs REST, K8s Workloads, ConfigMaps vs Secrets — Complete Study Notes

**Date:** Week 4, Day 2 (Wednesday evening)

---

## PART A: gRPC vs REST

## 1. REST — The Baseline

REST (Representational State Transfer) is an **architectural style**, not a protocol. It uses HTTP/1.1 or HTTP/2 as transport and JSON (usually) as its data format.

**Core model:** resources identified by URLs, manipulated via HTTP verbs.

```
GET    /users/123    → fetch user 123
POST   /users        → create a new user
PUT    /users/123    → replace user 123
PATCH  /users/123    → partially update user 123
DELETE /users/123    → delete user 123
```

**What REST gives you:**
- **Universal support** — every language, browser, and HTTP client speaks REST. curl, Postman, any framework.
- **Human readable** — JSON payloads self-describing and debuggable without tooling
- **Stateless** — each request independent; scales horizontally easily
- **Cacheable** — HTTP caching (CDN, browser, proxy) works naturally on GET responses
- **Firewall/proxy friendly** — standard HTTP ports 80/443, standard HTTP semantics

**What REST costs you:**
- **Verbose payloads** — JSON field names repeated in every response. A list of 10,000 users sends the string `"firstName"` 10,000 times.
- **No schema enforcement** — no built-in way to enforce field presence or types. External via OpenAPI/Swagger.
- **Request-response only** — one request, one response. Streaming requires workarounds (SSE, WebSockets, long polling — each a different mechanism).
- **No built-in code generation** — client libraries hand-written or generated from external schema.

## 2. gRPC — What It Is

gRPC is a **Remote Procedure Call framework** built by Google, open-sourced 2015. Runs over **HTTP/2**, uses **Protocol Buffers** as its serialization format.

Mental model shift: instead of "manipulate resources via HTTP verbs," gRPC says "call a function on a remote server." You define a service with methods; the framework generates client and server code in your chosen language.

```protobuf
// user.proto — the contract between client and server
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
  rpc StreamUpdates (stream UpdateRequest) returns (stream UpdateResponse);
}

message GetUserRequest {
  int64 user_id = 1;
}

message User {
  int64  id         = 1;
  string first_name = 2;
  string last_name  = 3;
  string email      = 4;
}
```

Run `protoc` against the `.proto` file → generated client and server stubs in Go, Python, Java, C++, or any of ~10 supported languages. Client stub looks like a local function call:

```python
# No manual HTTP construction, no JSON parsing, no field name strings
user = stub.GetUser(GetUserRequest(user_id=123))
print(user.first_name)
```

## 3. Protocol Buffers — The Serialization Layer

Protobuf is a **binary serialization format** — data encoded as compact binary, not human-readable text.

**How it works:** Each field identified by its **field number** (the integer after `=` in the proto definition), not its name. Wire format encodes: field number + wire type + value. Field names completely absent from serialized bytes.

```
User { id: 123, first_name: "Abhijeet", email: "a@b.com" }

JSON:    {"id":123,"first_name":"Abhijeet","email":"a@b.com"}  → ~47 bytes
Protobuf: [binary]                                              → ~18 bytes
```

**Why binary is faster:** no string parsing, no delimiter scanning, no type inference. Field numbers are integers — looked up in a generated decode table in nanoseconds.

**Schema evolution:** Field numbers are permanent. You can add new fields (new numbers), deprecate old ones (mark `reserved`), rename fields freely (wire format only knows numbers). Old clients skip unknown field numbers. New clients see zero/empty for missing fields. Backward and forward compatibility built into the format.

**Tradeoff:** not human-readable. Can't `curl` a gRPC endpoint and read the response. Need tooling — `grpcurl` is the standard CLI equivalent.

## 4. HTTP/2 as Transport

gRPC uses HTTP/2, giving it capabilities impossible in HTTP/1.1:

**Multiplexing:** HTTP/1.1 = one request per TCP connection (or pipelining with head-of-line blocking). HTTP/2 multiplexes many streams over a single TCP connection — each gRPC call is one stream. Thousands of concurrent RPCs on a small number of TCP connections.

**Binary framing:** HTTP/2 sends data in binary frames. No text parsing overhead.

**Header compression (HPACK):** Repeated headers transmitted as references to a shared table, not full strings.

## 5. The Four gRPC Communication Patterns

**1. Unary RPC** — one request, one response. Equivalent to REST.
```protobuf
rpc GetUser (GetUserRequest) returns (User);
```

**2. Server streaming** — one request, stream of responses.
```protobuf
rpc ListUsers (ListUsersRequest) returns (stream User);
```
Use case: return 1M users in chunks, real-time feeds, log tailing, stock price updates.

**3. Client streaming** — stream of requests, one response.
```protobuf
rpc UploadChunks (stream Chunk) returns (UploadResult);
```
Use case: large file upload in chunks, IoT sensor batching readings.

**4. Bidirectional streaming** — both sides send streams simultaneously over the same HTTP/2 connection.
```protobuf
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```
Use case: real-time chat, multiplayer game state, live collaborative editing.

REST has no native equivalent to patterns 2, 3, or 4. WebSockets approximate bidirectional streaming but are a separate protocol with different semantics and no schema enforcement.

## 6. gRPC vs REST — When to Use Which

| Dimension | REST | gRPC |
|-----------|------|------|
| Client type | Public-facing, browsers, mobile | Internal service-to-service |
| Payload efficiency | JSON — verbose but readable | Protobuf — compact, fast |
| Schema enforcement | External (OpenAPI) | Built-in (.proto contract) |
| Code generation | Optional | First-class (.proto → stubs) |
| Streaming | Workarounds (SSE, WS) | Native (4 patterns) |
| Browser support | Native | Needs gRPC-Web proxy |
| Debugging | curl, Postman, browser | grpcurl, specialized tooling |
| Load balancing | Standard L7 HTTP LB | Needs gRPC-aware L7 LB |
| Caching | HTTP cache naturally | No HTTP cache |

**Use REST when:** public API, external developers or browsers calling it, HTTP caching valuable, discoverability and human-readability matter.

**Use gRPC when:** internal microservices (both ends controlled), high throughput / low latency, streaming needed, polyglot environment, strong contract enforcement between teams.

**The common real-world pattern:** REST at the edge (public API gateway) + gRPC internally (service-to-service). Browser/mobile talks REST to the gateway. Behind the gateway, microservices communicate via gRPC. Best of both: broad compatibility externally, efficiency and type safety internally. This is Google's internal architecture.

**Why Google uses gRPC internally:** At Google's scale, even a 10% payload reduction = hundreds of terabytes of bandwidth saved per day. gRPC was based on Google's internal Stubby RPC framework.

**The browser limitation:** Browsers control TCP connections at the OS level and don't expose low-level HTTP/2 stream control required by gRPC. Specifically, browsers don't allow reading HTTP/2 trailers (where gRPC sends its status code) and don't support client streaming. **gRPC-Web** is a proxy (typically Envoy or nginx) that translates between the browser's HTTP/1.1 requests and the backend's gRPC protocol. Without this proxy, a browser cannot call a gRPC endpoint directly.

---

## PART B: K8s Pods, Deployments, ReplicaSets

## 7. Pod — The Atomic Unit

The smallest deployable unit in Kubernetes. Contains one or more containers sharing:
- Same **network namespace** (same IP, same port space)
- Same **IPC namespace**
- Optionally shared **volumes**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: search-api
  labels:
    app: search-api
spec:
  containers:
  - name: search-api
    image: search-api:v1
    ports:
    - containerPort: 8000
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"
```

**Almost never create Pods directly in production.** A Pod has no self-healing — if it crashes or its node dies, it's gone. You use controllers that manage Pods.

## 8. ReplicaSet — Maintaining a Count

Ensures a specified number of identical Pods are running at all times. Watches Pods matching its label selector, creates or deletes Pods to match desired count.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: search-api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: search-api
  template:             # Pod template — IS a Pod spec
    metadata:
      labels:
        app: search-api
    spec:
      containers:
      - name: search-api
        image: search-api:v1
```

If you delete a Pod → ReplicaSet creates a replacement. Node dies → creates new Pods on surviving nodes. But no concept of rolling updates — updating the image requires deleting and recreating Pods manually. This is why you rarely create ReplicaSets directly.

## 9. Deployment — The Standard Workload

Manages ReplicaSets and adds rolling update capability. When you change the Pod template (new image, env var, resource limits), Deployment creates a new ReplicaSet and orchestrates the transition.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: search-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: search-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # max pods above desired during update
      maxUnavailable: 1    # max pods below desired during update
  template:
    metadata:
      labels:
        app: search-api
    spec:
      containers:
      - name: search-api
        image: search-api:v2    # changed from v1
```

## 10. Rolling Update Sequence

**With `maxSurge: 1`, `maxUnavailable: 1`, desired 3 replicas:**
1. New RS created with v2 template
2. Scale new RS to 1 → now 4 pods total (3 old + 1 new)
3. Wait for new pod to be Ready
4. Scale old RS to 2 → now 3 pods (2 old + 1 new)
5. Scale new RS to 2, wait for Ready
6. Scale old RS to 1
7. Scale new RS to 3, wait for Ready
8. Scale old RS to 0 — kept at 0 for rollback

**With `maxSurge: 1`, `maxUnavailable: 0` (zero-downtime, safe):**
- Never goes below desired 3 replicas
- Scale new RS to 1 first (now 4 pods), wait for Ready, then scale old RS to 2 (back to 3)
- Slower but guaranteed no unavailability
- Note: requires PDB headroom — if PDB says `minAvailable: 3` this blocks

**Rollback:**
```bash
kubectl rollout undo deployment/search-api
```
Scales old RS back up, new RS back down. Old RS kept at 0 exactly for this purpose.

## 11. The Object Hierarchy

```
Deployment
  └── ReplicaSet (v1 image) — kept at 0 after update, for rollback
  └── ReplicaSet (v2 image) — active
        ├── Pod (v2)
        ├── Pod (v2)
        └── Pod (v2)
```

Deployment controller → manages ReplicaSets.
ReplicaSet controller → manages Pods.
Two separate reconciliation loops, two separate controllers.

## 12. Essential kubectl Commands

```bash
kubectl get deployments
kubectl get replicasets
kubectl describe deployment search-api
kubectl rollout status deployment/search-api     # watch rollout progress
kubectl rollout history deployment/search-api    # see past revisions
kubectl rollout undo deployment/search-api       # rollback one version
kubectl rollout undo deployment/search-api --to-revision=2  # rollback to specific
kubectl scale deployment search-api --replicas=5
kubectl set image deployment/search-api search-api=search-api:v3  # trigger update
```

---

## PART C: ConfigMaps vs Secrets

## 13. ConfigMap — Non-Sensitive Config

Decouples non-sensitive configuration from container images.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: search-api-config
data:
  MODEL_NAME: "all-MiniLM-L6-v2"
  LOG_LEVEL: "info"
  MAX_RESULTS: "10"
  config.yaml: |           # can store whole config files
    model:
      name: all-MiniLM-L6-v2
    server:
      port: 8000
```

## 14. Secret — Sensitive Data

Passwords, API keys, TLS certificates, tokens.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: search-api-secret
type: Opaque
data:
  API_KEY: YWJoaWplZXQ=       # base64 encoded — NOT encrypted
  DB_PASSWORD: c2VjcmV0        # base64 encoded — NOT encrypted
```

**Critical distinction:** Secrets are base64 encoded, **not encrypted by default.** `echo YWJoaWplZXQ= | base64 -d` gives you the original string immediately. The security model relies on:
- RBAC restrictions on who can `get`/`list` Secrets
- etcd encryption at rest (must be explicitly configured — not on by default)
- Audit logging of Secret access

For stronger security: external secret stores (HashiCorp Vault, AWS Secrets Manager) with Kubernetes External Secrets operator, keeping secrets entirely outside etcd.

## 15. Two Ways to Consume Both

**Method 1 — Environment variables:**
```yaml
spec:
  containers:
  - name: search-api
    env:
    - name: MODEL_NAME
      valueFrom:
        configMapKeyRef:
          name: search-api-config
          key: MODEL_NAME
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: search-api-secret
          key: API_KEY
    envFrom:                     # inject ALL keys from ConfigMap as env vars
    - configMapRef:
        name: search-api-config
```

**Method 2 — Volume mount (files):**
```yaml
spec:
  containers:
  - name: search-api
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-vol
    configMap:
      name: search-api-config
  - name: secret-vol
    secret:
      secretName: search-api-secret
```

## 16. Env Var vs Volume Mount — When to Use Which

| Scenario | Env var | Volume mount |
|----------|---------|-------------|
| Simple key/value config | ✅ | Works but overkill |
| Whole config files | ❌ awkward | ✅ natural fit |
| TLS certificates | ❌ | ✅ (`/etc/tls/tls.crt`) |
| Live updates without restart | ❌ | ✅ |
| App uses `os.getenv()` | ✅ | ❌ needs file reading |
| App expects config files | ❌ | ✅ |

**The live update behavior — exam-critical:** Update a ConfigMap → Pods consuming it via **volume mount** see updated files within ~60 seconds, no restart needed. Pods consuming it via **env var** do NOT see the update — env vars are set at container startup and never change. Must restart Pods (rolling update) to pick up env var changes.

This asymmetry is a common production gotcha: operator updates a ConfigMap, checks the app, sees no change, assumes the update didn't apply — but it's actually that env var injection only happens at container start.

## 17. ConfigMap vs Secret — Practical Comparison

| | ConfigMap | Secret |
|--|-----------|--------|
| Data encoding | Plain text | base64 |
| Encryption at rest | No | Optional (requires etcd config) |
| RBAC | Standard | Same, but convention = tighter |
| Use for | Non-sensitive config, feature flags | Passwords, API keys, certs, tokens |
| Max size | 1MB | 1MB |
| Immutable option | Yes | Yes |

`immutable: true` on either prevents accidental changes and improves performance (kubelet stops watching for changes).

---

## Retention Questions and Answers

**Q1: A team uses REST for internal service-to-service calls between 12 microservices. They're experiencing high latency and bandwidth costs. What would switching to gRPC concretely change, and what would it cost?**

Switching to gRPC would concretely change:
- **Payload size:** Protobuf binary encoding is typically 3–10× smaller than equivalent JSON. A response with many repeated field names (e.g. 1000 user objects) benefits most. This directly reduces bandwidth costs and reduces serialization/deserialization CPU time.
- **Connection overhead:** HTTP/2 multiplexing means multiple concurrent service-to-service calls share one TCP connection per service pair instead of opening new connections per request. Eliminates TCP handshake and TLS handshake overhead for high-frequency calls.
- **Type safety:** Generated client stubs eliminate an entire class of bugs at service boundaries — wrong field name, wrong type, missing required field. Caught at compile time, not in production.
- **Streaming:** If any of the 12 services needs to stream data (currently implemented as chunked REST or polling), bidirectional streaming becomes native.

What it costs:
- **Tooling shift:** developers can't use curl or Postman. They need grpcurl and proto-aware tooling. Debugging is harder without human-readable payloads.
- **Load balancer changes:** standard L4/L7 HTTP LBs may not handle gRPC correctly because gRPC uses long-lived HTTP/2 connections with multiple streams. Need an L7 LB that understands gRPC framing (Envoy, nginx with gRPC support) or service-mesh-level load balancing.
- **Browser access:** if any of these services are ever called directly from a browser, they need a gRPC-Web proxy layer.
- **Migration cost:** every service needs .proto files, generated stubs, and rewritten client code. A 12-service migration is significant engineering work.

---

**Q2: What does `maxSurge: 1, maxUnavailable: 0` mean during a rolling update — walk through the exact sequence for a 3-replica Deployment?**

`maxSurge: 1` means at most 1 extra pod above the desired count (3) can exist at any time. `maxUnavailable: 0` means zero pods below the desired count can be unavailable at any time — you always have at least 3 available pods.

Sequence for 3 replicas, updating from v1 to v2:

1. New RS (v2) created. Old RS (v1) at 3.
2. Scale new RS to 1 → now 4 pods total (3 old v1, 1 new v2). This is the surge — 1 above desired.
3. Wait for the new v2 pod to pass readiness probe and be Ready.
4. Scale old RS down to 2 → now 3 pods total (2 old v1, 1 new v2). Never dropped below 3 — `maxUnavailable: 0` satisfied.
5. Scale new RS to 2 → 4 total (2 old v1, 2 new v2).
6. Wait for second v2 pod to be Ready.
7. Scale old RS to 1 → 3 total (1 old v1, 2 new v2).
8. Scale new RS to 3 → 4 total (1 old v1, 3 new v2).
9. Wait for third v2 pod to be Ready.
10. Scale old RS to 0 → 3 total (3 new v2). Old RS kept at 0 for rollback.

This is slower than allowing `maxUnavailable: 1` (which can scale down and up simultaneously) but guarantees zero reduction in serving capacity at any point — important for latency-sensitive or traffic-sensitive services.

---

**Q3: You updated a ConfigMap that provides `LOG_LEVEL` to your app via environment variable. Pods are still logging at the old level. Why, and what's the fix?**

Environment variables are injected into a container's process environment at startup, using the values from the ConfigMap at the time the container was created. They are not dynamic — the process receives a copy of the environment at launch and that copy never changes regardless of what happens to the ConfigMap afterward.

Updating the ConfigMap writes new data to etcd. Kubernetes propagates this to kubelet's local cache. But the running container's environment is already set — the kubelet doesn't signal the container or update its environment in place. The change simply doesn't reach the running process.

Fix: trigger a rolling restart of the Deployment so each container starts fresh with the updated ConfigMap values:
```bash
kubectl rollout restart deployment/search-api
```
This performs a rolling update with the same Pod template, but each new container reads the ConfigMap values fresh at startup — picking up `LOG_LEVEL: debug` or whatever was changed.

If you had used volume mounts instead of env vars, the updated ConfigMap would have propagated to the file inside the running container within ~60 seconds without any restart. This is a key reason to prefer volume mounts for configuration that changes more frequently.

---

**Q4: What's the difference between a ReplicaSet and a Deployment — why do you almost never create a ReplicaSet directly?**

A ReplicaSet does exactly one thing: ensures a specified number of Pods matching its selector are running. It creates Pods when count is too low, deletes them when too high. If you change its Pod template (e.g. new image tag), it does nothing to existing Pods — it only acts when the count is wrong. To update the image you'd have to delete existing Pods manually so the RS recreates them with the new template, causing downtime.

A Deployment wraps one or more ReplicaSets and adds rolling update orchestration. When you change the Deployment's Pod template, the Deployment controller creates a **new ReplicaSet** with the new template and gradually scales it up while scaling the old RS down, ensuring the configured `maxUnavailable`/`maxSurge` constraints are respected. The old RS is kept at 0 replicas to enable `kubectl rollout undo`. The Deployment also provides rollout history and pause/resume capability.

You don't create ReplicaSets directly because the moment you need to update your application (which is constantly), you'd be doing manual rolling updates yourself. The Deployment controller automates exactly that. The ReplicaSet is an implementation detail of how Deployment achieves rolling updates — you interact with the Deployment and let it manage the ReplicaSets.

---

**Q5: Why are Kubernetes Secrets not actually secret by default, and what makes them more secure than ConfigMaps in practice?**

Secrets are not encrypted — they are base64 encoded, which is a reversible encoding, not encryption. Anyone who can `kubectl get secret search-api-secret -o yaml` sees the data and can decode it with `echo <value> | base64 -d`. In etcd (where all Kubernetes objects are stored), Secrets are stored as plaintext by default — anyone with direct etcd access reads them unobstructed.

What makes them more secure than ConfigMaps in practice is a combination of convention and configuration:

**Convention:** teams apply tighter RBAC to Secrets — service accounts that need ConfigMaps don't necessarily get access to Secrets. The separate object type creates a clear access control boundary. `kubectl get configmaps` and `kubectl get secrets` are separately RBAC-controlled verbs.

**etcd encryption at rest (requires explicit configuration):** you can configure the Kubernetes API server with an EncryptionConfiguration that encrypts specific resource types (like Secrets) before writing them to etcd. This means an attacker with direct etcd access sees encrypted blobs for Secrets but plaintext for ConfigMaps. This is not enabled by default — you must configure it.

**External secret stores:** the strongest model keeps Secret values entirely outside Kubernetes. HashiCorp Vault or AWS Secrets Manager stores the actual values; the Kubernetes External Secrets operator fetches them at runtime and creates Kubernetes Secrets that are short-lived and auto-rotated. etcd never sees the raw sensitive value for long.

---

**Q6: Explain why gRPC can't be called directly from a browser without a proxy layer, while REST can.**

Browsers implement the HTTP/2 specification at a high level via the Fetch API and XMLHttpRequest, but they deliberately restrict access to low-level HTTP/2 stream control for security reasons. Two specific limitations block native gRPC in browsers:

First, gRPC uses **HTTP/2 trailers** (headers sent after the response body) to transmit the gRPC status code and any error details. Browsers expose response headers (sent before the body) via the Fetch API but do not expose HTTP/2 trailers — the browser's networking layer strips them. Without access to trailers, the browser can't read the gRPC status code and can't determine whether the call succeeded or failed.

Second, **client-side streaming** in gRPC requires sending a stream of data frames on an HTTP/2 request body before the server responds. Browsers don't support request body streaming in the way gRPC requires — the Fetch API doesn't allow sending an HTTP/2 stream of framed messages.

gRPC-Web solves this by defining a protocol that works within browser constraints: it encodes gRPC trailers into the response body itself (since the browser can read the body), and uses standard HTTP/1.1 or HTTP/2 request patterns the browser supports. A proxy (typically Envoy or nginx with the gRPC-Web module) sits between the browser and the gRPC backend, translating gRPC-Web requests into full gRPC and vice versa.

REST works natively in browsers because it uses standard HTTP/1.1 or HTTP/2 request-response patterns with headers and body that the browser's Fetch API fully exposes, with no reliance on trailers or streaming request bodies.
