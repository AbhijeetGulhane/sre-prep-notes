# TCP: Flow Control, Congestion Control, and TIME_WAIT

> Week 4 — TCP/IP deep dive. Networking fundamentals block.

## 1. Flow control vs congestion control

Two different problems, two different mechanisms — don't conflate them in an interview.

| | Flow control (rwnd) | Congestion control (cwnd) |
|---|---|---|
| Protects | The receiver | The network |
| Signal | Explicit — receiver advertises free buffer space in every ACK's `window` field | Implicit — inferred from loss (or ECN/delay, depending on algorithm) |
| Scope | End-to-end (sender ↔ receiver only) | Path-wide (shared with all competing flows) |
| Failure mode | Zero window — receiver buffer full, sender pauses until window update | Loss triggers backoff |

Effective sending rate is always:

```
effective window = min(cwnd, rwnd)
```

A fast network with a slow-reading receiver app is flow-control-limited even with huge cwnd, and vice versa.

## 2. Congestion control phases (AIMD)

**Slow start**
- cwnd starts at 1 MSS (up to ~10 with RFC 6928 initial window)
- Roughly doubles every RTT (cwnd += 1 MSS per ACK received → compounds to ~2x/RTT)
- Exponential growth until cwnd reaches `ssthresh` or a loss occurs

**Congestion avoidance**
- Once cwnd ≥ ssthresh, growth becomes linear: ~+1 MSS per RTT
- This is the "additive increase" half of AIMD — cautious probing for more bandwidth

**Loss event → multiplicative decrease** (the two loss signals are NOT treated the same)

| Signal | Meaning | Response |
|---|---|---|
| Triple duplicate ACK | Packets still getting through; one segment lost, later ones arrived out of order | Fast retransmit + fast recovery: `ssthresh = cwnd/2`, `cwnd = ssthresh` (not reset to 1), resume congestion avoidance |
| Timeout (no ACK before retransmit timer fires) | Serious congestion / possible full stop; all RTT timing info is lost | `ssthresh = cwnd/2`, `cwnd = 1`, restart slow start from scratch |

This asymmetry is the detail worth stating explicitly: a timeout means TCP can no longer trust anything it knew about the path, so the reset is much more aggressive than a dup-ACK loss.

**Why AIMD specifically:** additive increase + multiplicative decrease is the specific combination proven to converge multiple competing flows toward a fair, equal bandwidth share — not an arbitrary design choice.

**BBR (modern departure):** used by Google/YouTube/gRPC. Doesn't wait for loss — models bottleneck bandwidth and RTT directly. Replaces loss-based CC because loss-based algorithms underutilize high-BDP, lossy, or bufferbloated links. Worth naming this at Google specifically.

### cwnd vs time — the shape to redraw from memory
Sawtooth: exponential climb (slow start) → linear climb (congestion avoidance) → drop to ~half on dup-ACK loss → linear climb again → deep drop to 1 on timeout → slow start restarts. Plot cwnd (y) against RTTs (x).

## 3. TIME_WAIT

**What it is:** after the 4-way FIN/ACK teardown, the side that sent the final ACK (usually the active closer) holds the connection state in TIME_WAIT for a fixed duration before freeing the socket tuple.

**Why it exists:**
1. **Lost final ACK protection** — if the last ACK is lost, the peer (in LAST_ACK) retransmits its FIN. Without TIME_WAIT state, that retransmitted FIN gets a RST instead of an ACK, and the peer's `close()` errors out.
2. **Prevents old duplicate segments from a previous connection** being misdelivered into a new connection that reuses the same 4-tuple (src IP, src port, dst IP, dst port) too soon.

**Why 2×MSL:** MSL = Maximum Segment Lifetime, the assumed max time a packet survives in the network. 2×MSL = one MSL for a stray packet to reach the peer + one MSL for a response back — guarantees any old duplicate has died out before the tuple is reused. RFC assumes MSL = 2 min (TIME_WAIT = 4 min); Linux hardcodes TIME_WAIT to **60 seconds** in practice.

**Why it's an operational problem at scale:** high connection-churn servers (proxies, LBs, anything with many short-lived outbound connections) accumulate thousands of TIME_WAIT sockets, each holding an ephemeral port (`net.ipv4.ip_local_port_range`, ~28K ports by default). If churn outpaces TIME_WAIT expiry → ephemeral port exhaustion → `connect()` fails with `EADDRNOTAVAIL`. Classic incident pattern at high-throughput fleets (e.g. Barclays' ~60K server environment).

**Tuning, safest → riskiest:**

| Setting | Effect | Caveat |
|---|---|---|
| `net.ipv4.tcp_tw_reuse` | Reuse a TIME_WAIT socket for a new *outgoing* connection when TCP timestamps prove safety | Client-side only; requires `tcp_timestamps` enabled; generally safe |
| Widen `net.ipv4.ip_local_port_range` | More ephemeral ports before exhaustion | Delays the problem, doesn't fix root cause |
| Connection pooling / keep-alive | Reduces the *rate* of connection open/close | Best real fix |
| `net.ipv4.tcp_tw_recycle` | Aggressive TIME_WAIT recycling | **Removed in Linux 4.12** — broke NAT'd clients (relied on per-IP monotonic timestamps, which NAT violates). Do not recommend this; if mentioned, immediately note it was removed. |
| `SO_REUSEADDR` | Lets a *listening* socket rebind to a port with TIME_WAIT sockets present | Different problem (server restart binding), not client-side port exhaustion |

**Interview one-liner:**
> "TIME_WAIT exists to safely absorb a lost final ACK and prevent old segments bleeding into a new connection on the same tuple; at high connection churn it exhausts ephemeral ports, and the standard fix is `tcp_tw_reuse` plus reducing connection churn via pooling — not `tcp_tw_recycle`, which was removed for breaking NAT."

---
*Next up: SO_REUSEADDR / SO_REUSEPORT, then DNS resolution (recursive vs iterative, record types).*
