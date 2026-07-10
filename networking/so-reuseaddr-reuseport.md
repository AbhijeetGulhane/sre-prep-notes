# SO_REUSEADDR vs SO_REUSEPORT

> Week 4 — TCP/IP deep dive (final topic for this day). Follows directly from TIME_WAIT.

## SO_REUSEADDR

**Problem it solves:** restarting a server fails with `EADDRINUSE` because the old listening socket's local (addr, port) tuple is still tied up by a connection sitting in TIME_WAIT. By default the kernel refuses to bind a new socket to a local address that has *any* socket — including TIME_WAIT ones — associated with it.

**What it does:** tells the kernel to allow binding to a local port even if a socket in TIME_WAIT is using it. Scoped to the *local* bind check only — does not weaken TIME_WAIT's correctness guarantees for the specific 4-tuples it's protecting.

**Practical takeaway:** every production server framework sets this unconditionally before `bind()`. Without it, every restart of a busy server would block on TIME_WAIT expiry (up to 60s+ on Linux). Treat it as a correctness fix for restarts, not a performance knob.

**What it does NOT do:** does not let two independent processes both `listen()` on the same port and both receive connections.

## SO_REUSEPORT

**Problem it solves:** want multiple independent processes/threads to bind the *same* port and have the kernel load-balance incoming connections across them, instead of funneling through one shared accept queue.

**What it does:** allows multiple sockets to bind the identical (addr, port), provided every one sets the flag. Kernel hashes each incoming SYN's (or UDP datagram's) 4-tuple to pick which listener's accept queue receives it.

**Why it matters (Linux 3.9+, 2013):** pre-REUSEPORT multi-worker model = one shared listening socket, multiple workers calling `accept()` → shared accept-queue lock contention + thundering herd on new connections. REUSEPORT gives each worker its own listening socket + accept queue:
- No shared lock contention
- Kernel-level consistent-hash load balancing (a given flow always lands on the same socket)
- Enables per-core pinning — pairs with RSS/RPS on the NIC for cache locality and real throughput gains

Used by nginx (`reuseport` directive), HAProxy, and custom high-connection-rate services to scale `accept()` across cores.

## Side-by-side

| | SO_REUSEADDR | SO_REUSEPORT |
|---|---|---|
| Solves | Rebind blocked by TIME_WAIT | Load-balanced accept across multiple sockets |
| Who uses it | Single process, restarting | Multiple processes/threads, running concurrently |
| Kernel behavior | Relaxes local-address-in-use check at bind time | Hashes/fans out new connections across bound sockets |
| Typical use | Every server, always on | Multi-process/multi-core servers (nginx, HAProxy, custom high-throughput services) |
| Linux availability | Always been there | 3.9+ (2013) |

**Interview one-liner:**
> "REUSEADDR is about surviving your own restart without waiting out TIME_WAIT; REUSEPORT is about scaling accept() across multiple processes or cores by giving each one its own listening socket, with the kernel load-balancing new connections across them by hashing the 4-tuple."

---
*Next up: DNS — recursive vs iterative resolution chain (`dig +trace google.com` lab), and record types (A, AAAA, CNAME, MX, TXT, PTR, NS, SOA — including why you can't CNAME at the zone apex).*
