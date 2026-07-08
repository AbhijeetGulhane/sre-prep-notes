# TLS, mTLS, HTTPS Termination, Load Balancing — Complete Study Notes

**Date:** Week 4, Day 2 (Wednesday)

---

## PART A: TLS 1.3

## 1. Why TLS Exists

HTTP sends everything in plaintext — any device on the network path (ISP, router, Wi-Fi AP) can read or modify data. TLS wraps HTTP in a cryptographic tunnel providing three guarantees:

- **Confidentiality** — data encrypted, unreadable to observers
- **Integrity** — data can't be modified in transit without detection
- **Authentication** — the server is who it claims to be (via certificate)

## 2. TLS 1.3 Handshake — Step by Step

TLS 1.3 completes in **1 round trip (1-RTT)**. Key improvement over TLS 1.2's 2-RTT.

**Client Hello (client → server):**
- Protocol version (TLS 1.3)
- Random nonce (`client_random`)
- Supported cipher suites (e.g. `TLS_AES_256_GCM_SHA384`)
- **Key share** — client's ECDHE public key. In TLS 1.3, client proactively sends this for its most likely cipher suite without waiting for server to negotiate first.

**Server Hello (server → client, same round trip):**
- Selected cipher suite
- Server's own ECDHE public key share
- Server's **certificate** (public key + CA signature)
- **Certificate Verify** — digital signature over the handshake transcript proving server holds the matching private key
- **Finished** — MAC over entire handshake proving integrity

At this point, both sides independently derive the same symmetric session keys from the ECDHE exchange (using HKDF). Server can start sending encrypted application data immediately.

**Client Finished:**
- Verifies certificate chain back to trusted CA
- Verifies server's signature
- Sends its own `Finished` MAC
- Starts sending encrypted application data

**Session key derivation:** Both sides use the ECDHE shared secret + both random nonces → HKDF → separate keys for each direction. The actual session key is never transmitted — independently computed.

## 3. TLS 1.2 vs TLS 1.3

| Feature | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| Round trips to first byte | 2-RTT | 1-RTT |
| Key exchange | RSA or DHE/ECDHE | ECDHE only (mandatory) |
| Cipher suites | Many including weak | 5 strong only |
| Forward secrecy | Optional (RSA = no FS) | Mandatory (ECDHE always) |
| Certificate in handshake | Plaintext | Encrypted |
| 0-RTT resumption | Session tickets (limited) | 0-RTT PSK |

**Forward secrecy precisely:** In TLS 1.2 with RSA key exchange, if an attacker records today's encrypted traffic and later steals the server's private key, they can decrypt all past sessions. With ECDHE (mandatory in TLS 1.3), each session uses a fresh ephemeral key pair discarded after the session. Stealing the private key later gains nothing — past sessions are protected.

## 4. 0-RTT Resumption

If client and server have communicated before, TLS 1.3 allows the client to send encrypted application data in the very first message — zero additional round trips. Uses a **Pre-Shared Key (PSK)** derived from the previous session.

**The replay attack problem:** 0-RTT data has no replay protection. An attacker who captures a 0-RTT request can re-send it and the server can't distinguish it from the original. Makes 0-RTT unsafe for non-idempotent operations (POST, DELETE, payment processing). Safe only for idempotent reads (GET with no side effects). CDNs like Cloudflare support it but restrict to safe methods.

---

## PART B: mTLS — Mutual TLS

## 5. What Changes vs Standard TLS

Standard TLS: only the **server** presents a certificate — client verifies server identity. mTLS adds: the **client also presents a certificate** — server verifies client identity too. Both sides authenticate each other.

Extended handshake steps:
1. Server sends `Certificate Request` to client during handshake
2. Client sends its certificate + `Certificate Verify` signature
3. Server verifies client certificate chain against a trusted CA
4. Connection proceeds only if both certificates valid and trusted

## 6. Where mTLS Is Used

| Use case | Why |
|----------|-----|
| Service mesh (primary) | Authenticate service-to-service calls at infrastructure level, no app code changes |
| gRPC internal services | Both sides verify identity; strongly recommended by gRPC |
| API access control | Client certificate instead of API key; revocation via CRL/OCSP |
| IoT devices | Unique cert burned in at manufacture; authenticates to backend |

## 7. How Istio Implements mTLS

Istio achieves mTLS transparently — application containers write zero TLS code.

**Envoy sidecar proxies:** Every pod gets an Envoy proxy injected as a sidecar (via mutating admission webhook at pod creation). All inbound/outbound traffic goes through Envoy, not the application directly.

**Certificate issuance:** Istio's **Istiod** acts as a Certificate Authority. When a pod starts, its Envoy sidecar requests a certificate from Istiod, presenting the pod's Kubernetes Service Account token as identity proof. Istiod issues a short-lived X.509 certificate encoding a SPIFFE identity: `spiffe://cluster.local/ns/default/sa/payment-service`.

**mTLS enforcement:** When `order-service` calls `payment-service`:
1. `order-service`'s Envoy intercepts the outbound call
2. Establishes mTLS connection to `payment-service`'s Envoy
3. Both present certificates, verify each other
4. Application containers on both ends see plain HTTP — mTLS entirely at the sidecar layer

**PeerAuthentication policy:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
spec:
  mtls:
    mode: STRICT      # all traffic must use mTLS; plaintext rejected
    # mode: PERMISSIVE  # allows both mTLS and plaintext (used during migration)
```

**Why short-lived certificates matter:** Istio certificates have ~24hr TTL and auto-rotate. Compromised certificate becomes useless quickly. No explicit revocation infrastructure needed.

---

## PART C: HTTPS Termination — Three Modes

## 8. Mode 1 — Terminate at the Load Balancer (Most Common)

```
Client ──[TLS]──► LB (decrypts) ──[plaintext HTTP]──► Backend pods
```

LB holds the certificate and private key. Decrypts incoming traffic, forwards plaintext to backends.

**Pros:**
- Backend services are simple — no TLS code, no cert management
- LB can inspect HTTP headers, do path routing, add headers, log request details
- Certificate management centralized at one place
- Internal traffic stays in private network — plaintext acceptable

**Cons:**
- Private key on the LB — single high-value target
- Internal traffic is plaintext — fails compliance mandating encryption-in-transit everywhere (PCI-DSS, HIPAA)
- LB sees all decrypted traffic — regulatory concern in some industries

**When to use:** Most web applications, APIs, SaaS. Default for Google Cloud HTTPS LB, AWS ALB.

## 9. Mode 2 — TLS Pass-Through

```
Client ──[TLS]──► LB (forwards encrypted) ──[TLS]──► Backend pods
```

LB never decrypts. Forwards the encrypted TCP stream to a backend based on TCP-level info (IP, port) or SNI (Server Name Indication — hostname in unencrypted Client Hello).

**Pros:**
- True end-to-end encryption — no plaintext anywhere in the path
- Private key never touches the LB
- Required for protocols the LB can't or shouldn't interpret

**Cons:**
- LB can't see HTTP headers, paths, cookies — **no L7 routing possible**
- Can't add headers, log request details, or do auth at LB level
- Backend services must handle TLS — cert management distributed everywhere
- Health checks must be TCP-level only

**When to use:** Database connection poolers (PgBouncer), gRPC with strict mTLS, custom binary protocols, protocols that break if decrypted mid-path.

## 10. Mode 3 — Re-encrypt (TLS Bridging)

```
Client ──[TLS]──► LB (decrypts, re-encrypts) ──[TLS]──► Backend pods
```

LB terminates client's TLS, inspects/modifies at L7, establishes new TLS connection to backend. Two separate TLS sessions.

**Pros:**
- Full L7 visibility and routing at the LB
- End-to-end encryption including internal network segment
- Satisfies compliance requirements for encryption-in-transit everywhere

**Cons:**
- Double TLS overhead (two handshakes per request)
- Backend services still need certificates
- More complex cert management

**When to use:** Financial services and healthcare where compliance mandates encryption everywhere, but L7 routing is still needed. Kubernetes ingress controllers with backend TLS.

---

## PART D: L4 vs L7 Load Balancers

## 11. L4 — Transport Layer LB

Operates on TCP/UDP packets. Sees: source IP, destination IP, source port, destination port. Payload is opaque.

**How it works:** Picks a backend using algorithm, forwards all packets of that TCP connection to the same backend. Maintains connection table (client IP:port → backend IP:port). Essentially NAT at scale.

**Characteristics:**
- Extremely fast — minimal processing per packet, just table lookup
- Protocol agnostic — works for HTTP, gRPC, MySQL, Redis, SMTP, any TCP/UDP
- Can't inspect content — no path routing, no header manipulation, no auth
- One TCP connection from client → one to backend

**Use cases:** Database load balancing (MySQL, Redis, PostgreSQL), any non-HTTP protocol, maximum throughput. AWS NLB, Google Cloud Network LB.

## 12. L7 — Application Layer LB

Operates on HTTP/HTTPS request level. Sees: URL path, HTTP headers, cookies, request body, HTTP method.

**How it works:** Terminates client's TCP connection. Parses HTTP request. Applies routing rules. Opens (or reuses pooled) TCP connection to selected backend. More processing per request, vastly more capability.

**Characteristics:**
- Path-based routing (`/api/*` → service A, `/web/*` → service B)
- Header-based routing (route by `User-Agent`, `X-Country`, `Authorization`)
- Cookie-based sticky sessions
- TLS termination
- Request/response modification (add/strip headers)
- Authentication and authorization at LB level
- Content-based health checks (`GET /healthz` expect 200)
- WebSocket and HTTP/2 multiplexing awareness

**Use cases:** Web applications, REST APIs, microservices routing, anything HTTP. AWS ALB, Google Cloud HTTPS LB, nginx, HAProxy, Envoy, Traefik.

## 13. L4 vs L7 Comparison

| | L4 | L7 |
|--|----|----|
| OSI layer | Transport (TCP/UDP) | Application (HTTP) |
| Sees | IP + port only | Headers, paths, cookies, body |
| TLS termination | No (pass-through) | Yes |
| Path routing | No | Yes |
| Header manipulation | No | Yes |
| Speed | Faster | Slightly slower |
| Protocol support | Any TCP/UDP | HTTP/HTTPS only |

---

## PART E: Load Balancing Algorithms

## 14. Round Robin

Each request goes to the next backend in sequence: Backend 1, 2, 3, 1, 2, 3...

Equal distribution assuming equal-cost requests. Breaks down when requests have wildly different processing times — slow backend accumulates a queue while others are idle.

## 15. Least Connections

New request goes to the backend with fewest active connections currently.

Handles heterogeneous request duration better than round robin — naturally avoids sending more work to an already-busy backend. Requires tracking active connections per backend. Common choice for long-lived connections (gRPC streams, WebSockets, database connections).

## 16. IP Hash (Source IP Affinity)

Hash the client's source IP → always route to the same backend.

Provides **sticky sessions** without cookies — same client always hits same backend. Useful when application maintains server-side session state that isn't shared across backends.

**The NAT proxy gotcha — classic failure mode:** Many clients behind a corporate proxy or NAT gateway share the same public source IP from the LB's perspective. IP hash sends all of them to the same backend → one backend gets 80-90% of traffic, others idle. Completely destroys load distribution.

**Fix:** Cookie-based stickiness (each client gets its own cookie regardless of source IP) or shared session storage (Redis, database-backed sessions) eliminating the need for stickiness at all.

## 17. Random

Pick a backend randomly per request. At scale, law of large numbers produces roughly even distribution. No state to maintain. Less predictable for small backend counts, equivalent to round robin at scale.

## 18. Weighted Variants

Any algorithm can be weighted — assign weight to each backend based on capacity. A backend with 8 CPUs gets 2× the traffic of one with 4. Essential when backends have different capacities.

---

## PART F: Health Checks and Circuit Breaker

## 19. Active Health Checks

LB proactively sends health check requests to each backend on a timer — typically `GET /healthz` every 5–30 seconds. Backend failing N consecutive checks → marked unhealthy, removed from pool. Passes M consecutive checks → re-added.

**Pros:**
- Fast detection — know a backend is down within seconds, regardless of whether real traffic has hit it
- Clean separation: health probes distinct from traffic

**Cons:**
- Generates extra traffic to all backends continuously
- Health check endpoint must accurately reflect real backend health (same DB connection issues, same thread pool status) — not just "process is alive"

## 20. Passive Health Checks (Outlier Detection)

LB observes real request outcomes — too many 5xx responses or timeouts within a window → backend marked unhealthy, temporarily removed. No separate probe traffic.

**Pros:**
- No synthetic traffic overhead
- Reflects actual user-facing behavior

**Cons:**
- Requires real user traffic to detect failure
- Some users receive errors before backend is removed
- Backend can fail silently between observation windows

## 21. Circuit Breaker Pattern

Software-level pattern preventing a struggling service from being overwhelmed with requests it can't handle. Stops cascade failures.

**Three states:**

**Closed (normal operation):**
Requests flow normally. Circuit breaker counts failures. Error rate below threshold → stays Closed.

**Open (failing fast):**
Failures exceed threshold within time window → breaker trips Open. All subsequent requests **fail immediately** without attempting to reach the backend — returns error or fallback response instantly. Gives struggling backend time to recover without continuous hammering.

**Half-Open (probing recovery):**
After a timeout period, allows a small number of test requests through.
- Test requests succeed → transition back to Closed (backend recovered)
- Test requests fail → return to Open (backend still struggling)

```
Closed ──[error threshold exceeded]──► Open ──[timeout expires]──► Half-Open
  ▲                                                                      │
  └──────────────[test requests succeed]────────────────────────────────┘
                                          │
                          [test requests fail]──► back to Open
```

**Why it matters for SRE:** Without circuit breaker, a slow downstream service causes your threads/goroutines to pile up waiting for timeouts → your service exhausts resources → cascade failure. Circuit breaker stops the cascade by failing fast and freeing resources immediately.

**Real implementations:** Resilience4j (Java), Envoy's circuit breaking (configured via Istio DestinationRule), cloud LB outlier detection, Hystrix (Netflix, now maintenance mode).

---

## How All These Connect

```
Client
  │
  ▼ [TLS 1.3 — 1-RTT, ECDHE, forward secrecy, cert auth]
  │
L7 LB — terminates TLS (mode 1 or re-encrypt mode 3)
  │  Inspects HTTP, routes by path/header/host
  │  Active health checks to backends
  │  Circuit breaker if backends failing
  │  LB algorithm: round-robin / least-conn / IP hash / weighted
  │
  ▼ [Plaintext or re-encrypted TLS to backend]
  │
Service Mesh (Istio) — if present
  │  mTLS between every service-to-service call
  │  Envoy sidecars handle it transparently
  │  PeerAuthentication STRICT: plaintext rejected
  │
  ▼
Backend Pod
```

---

## Retention Questions and Answers

**Q1: What is the exact purpose of the ECDHE key exchange in TLS 1.3 — what problem does it solve that RSA didn't?**

RSA key exchange: the client encrypts a pre-master secret with the server's public key, sends it to the server. If an attacker records today's traffic and later steals the server's private RSA key, they can decrypt the pre-master secret from the recording and then derive the session keys — decrypting all past sessions. No forward secrecy.

ECDHE (Elliptic Curve Diffie-Hellman Ephemeral): both sides generate fresh ephemeral key pairs per session. They exchange public keys, each independently computes the same shared secret without ever transmitting it (the mathematical property of Diffie-Hellman). After the session, the ephemeral private keys are discarded. Stealing the server's long-term private key later gains nothing — the ephemeral keys are gone. Past sessions remain secure. ECDHE provides **forward secrecy** (also called perfect forward secrecy). TLS 1.3 mandates ECDHE, making forward secrecy non-optional.

---

**Q2: Why is 0-RTT resumption unsafe for POST requests but safe for GET requests?**

0-RTT data has no replay protection — there is no mechanism for the server to distinguish a legitimate first request from an attacker re-transmitting a captured copy of that same request. An attacker who records a 0-RTT POST to `/api/payment` can re-send that exact packet and the server processes it again.

For GET requests (idempotent reads), replaying the request has no side effect — reading a resource multiple times returns the same result, no state changes. The attacker gains nothing from replaying "fetch my account balance."

For POST/DELETE/PUT (non-idempotent state-changing operations), replaying the request causes real damage — a duplicate payment, a second order, a repeated deletion. The server can't tell them apart. Hence 0-RTT should only be used for safe, idempotent HTTP methods. Cloudflare and major CDNs implementing 0-RTT specifically restrict it to `GET` and `HEAD` requests and reject 0-RTT for any method that changes server state.

---

**Q3: You're debugging a production issue where one backend is getting 80% of traffic while others are idle. The LB algorithm is IP hash. What's the most likely cause?**

The NAT proxy problem. Many clients are behind a corporate proxy, VPN, or NAT gateway — from the LB's perspective, they all share the same public source IP address. IP hash maps a source IP to a specific backend deterministically. If 80% of your users route through two or three corporate proxy IPs, those IPs consistently hash to the same one or two backends, while other backends sit idle.

Verify by checking the LB's access logs — if you see a handful of source IPs generating the vast majority of requests, that confirms the hypothesis.

Fix options: switch to cookie-based sticky sessions (each individual browser/client gets its own session cookie, regardless of source IP, giving the LB something unique per-client to hash on), or if session stickiness isn't actually needed, switch to round-robin or least-connections which don't depend on source IP at all. The cleanest fix is usually to eliminate the need for stickiness by externalizing session state to Redis or a database, then switch to round-robin.

---

**Q4: A client calls payment-service via Istio. Neither the client developer nor the payment-service developer wrote any TLS code. How is the mTLS connection actually established?**

Istio's mutating admission webhook injected an Envoy sidecar container into both pods at creation time. All network traffic is transparently redirected through these sidecars via iptables rules set up by Istio's `istio-init` init container.

When the client pod calls `payment-service`, its Envoy sidecar intercepts the outbound call before it leaves the pod. The Envoy sidecar holds a certificate previously issued by Istiod (Istio's control plane CA) — issued when the pod started, encoding the pod's SPIFFE identity derived from its Kubernetes Service Account.

Envoy initiates an mTLS handshake with the Envoy sidecar on the `payment-service` pod, presenting its certificate. `payment-service`'s Envoy verifies it against Istiod's CA, presents its own certificate, and the client's Envoy verifies that in return. Both sides authenticated. A TLS tunnel is established sidecar-to-sidecar.

The application container in the client pod sent plain HTTP. The application container in the `payment-service` pod receives plain HTTP. mTLS happened entirely in the sidecar layer — neither developer wrote any TLS code. Certificates auto-rotate every ~24 hours without any deployment or application involvement.

---

**Q5: Your LB is in pass-through mode and you need path-based routing (/api → A, /web → B). What's the problem and what's the fix?**

Pass-through mode means the LB forwards the encrypted TCP stream without decrypting it. The LB can only see TCP-level information — source IP, destination IP, port, and the SNI hostname in the unencrypted Client Hello. The LB cannot see the HTTP path `/api` or `/web` because those are inside the encrypted TLS payload.

Path-based routing is an L7 concept — it requires reading HTTP headers, which requires decrypting TLS first. A pass-through LB by definition can't do this.

Fix: switch to TLS termination at the LB (mode 1) or re-encryption (mode 3). Either allows the LB to decrypt the TLS stream, read the HTTP path, apply routing rules, and forward to the correct backend. If end-to-end encryption is required (the reason you were using pass-through), use re-encryption: LB terminates client TLS, applies path routing, re-encrypts to the backend.

The only partial workaround without changing mode is SNI-based routing — routing on the hostname rather than the path (different subdomains for different services: `api.example.com` → service A, `web.example.com` → service B). But this requires different hostnames, not different paths on the same hostname.

---

**Q6: Explain the difference between active and passive health checks — when would you use each, and what's the blind spot of passive-only?**

Active health checks: the LB proactively sends synthetic requests (e.g. `GET /healthz`) to each backend on a timer, independent of real user traffic. Failure → backend removed from pool. This detects failure fast regardless of traffic volume — even a backend with zero real traffic gets checked.

Passive health checks (outlier detection): the LB observes real request outcomes. If a backend returns too many 5xx responses or times out too often within a window, it's temporarily removed. No synthetic traffic generated.

When to use each: active is better for fast detection and for services with low or variable traffic (you can't passively detect failure on a backend that rarely gets requests). Passive is better for high-traffic services where you want health signals to reflect real user experience rather than synthetic probes, and where synthetic traffic would add meaningful overhead.

The blind spot of passive-only: a backend can fail silently and continue receiving traffic for the entire observation window before enough errors accumulate to trigger removal. Real users absorb the errors during that window. If traffic is low, it takes longer to accumulate enough observations to trigger removal — a backend could be failing for minutes before passive detection kicks in. Active health checks catch this immediately on the next probe interval regardless of traffic.

Best practice in production: use both. Active for fast detection, passive as a secondary signal that catches degraded-but-not-totally-failed backends that still pass synthetic health checks.

---

**Q7: A service's circuit breaker has been Open for 30 seconds and just transitioned to Half-Open. The first 3 test requests all time out. What happens next and why?**

The circuit breaker transitions back to **Open** immediately.

Half-Open is a cautious probing state — it allows a limited number of test requests through to check whether the backend has recovered. The threshold for "recovered" requires those test requests to succeed. Three consecutive timeouts signal the backend is still unable to handle requests — it has not recovered during the 30-second Open window.

Returning to Open makes sense because: if the backend is still timing out on test requests, sending full production traffic to it would cause the same cascade failure the circuit breaker was designed to prevent. The requests piling up would exhaust caller threads/goroutines again.

After returning to Open, another timeout period begins — perhaps longer than the first (some implementations use exponential backoff on the Open timeout, e.g. 30s → 60s → 120s) — before the circuit breaker tries Half-Open again. This gives the backend more time to recover before being retried. The pattern continues until either the backend recovers (test requests in Half-Open succeed → Closed) or intervention is required (the backend needs manual recovery — deployment fix, restart, resource increase).
