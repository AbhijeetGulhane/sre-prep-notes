# TCP Connection Management (K&R Ch. 3.5)

Covers: 3-way handshake, FIN teardown, RTO estimation & Karn's algorithm.

---

## 1. Three-Way Handshake (SYN / SYN-ACK / ACK)

Purpose: synchronize initial sequence numbers (ISNs) on both sides before any
data flows. "Synchronization," not just "connecting."

1. **SYN** — client sends `seq = client_isn`, SYN flag set, no payload.
2. **SYN-ACK** — server allocates state (half-open, `SYN_RCVD`), replies
   `seq = server_isn`, `ack = client_isn + 1`, SYN+ACK flags set.
3. **ACK** — client replies `seq = client_isn + 1`, `ack = server_isn + 1`.
   Can carry first byte of application data.

Both SYN and FIN consume one sequence number each even with no payload —
hence the `+1` on ACK numbers.

**Why not 2-way?** A delayed duplicate SYN from an old, already-closed
connection could cause the server to half-open a connection nobody
requested. The third ACK confirms the client actually received the
server's ISN and is live.

**Security note:** server commits real resources only after the final ACK.
The `SYN_RCVD` half-open state + backlog queue is exactly what SYN flood
attacks exploit.

ISNs are assigned pseudo-randomly (RFC 6528) to prevent sequence-number
prediction/injection attacks.

**Reading:** K&R §3.5.6 (Connection Management); RFC 793 / RFC 9293
(formal TCP state machine).

---

## 2. FIN Teardown (Four-Way Close)

Closing is asymmetric — each direction of the full-duplex connection closes
independently. This is **half-close**.

1. Client sends **FIN** — "no more data to send," not "done receiving."
2. Server **ACKs** it. Client → `FIN_WAIT_2`. Server can still send data
   (connection is half-closed).
3. When server has no more data, it sends its own **FIN**.
4. Client **ACKs** the server's FIN, enters **`TIME_WAIT`** for 2×MSL
   before fully closing (guards against delayed-duplicate segments and
   ensures the final ACK actually landed — see prior TIME_WAIT notes).

In practice, steps 2 and 3 often collapse into a single FIN+ACK segment if
the server has nothing queued — a 3-segment close. The 4-message model is
the general case (either side can still be sending when the other's FIN
arrives), which is how K&R draws it (§3.5.6, Fig. 3.31, client-active-close).

**Simultaneous close:** both sides send FIN before receiving the other's —
handled via `FIN_WAIT_1 → CLOSING → TIME_WAIT`. Less commonly asked, but
know it exists.

**Interview follow-up:** walk `netstat`/`ss` output and identify which
close-state a socket is in — good live-debugging material for Wk8-9
troubleshooting prep.

---

## 3. RTO Estimation & Karn's Algorithm (K&R §3.5.3)

TCP needs a retransmission timeout (RTO): how long to wait for an ACK
before assuming loss and resending. Too short → spurious retransmissions.
Too long → slow loss recovery. TCP adapts RTO continuously from measured
RTT.

**Jacobson/Karels algorithm (RFC 6298):**

```
EstimatedRTT = (1 - α) * EstimatedRTT + α * SampleRTT     (α = 1/8)
DevRTT       = (1 - β) * DevRTT + β * |SampleRTT - EstimatedRTT|   (β = 1/4)
RTO          = EstimatedRTT + 4 * DevRTT
```

- `EstimatedRTT` is an EWMA — recent samples weighted more, old ones not
  discarded outright, smooths a single slow sample.
- `DevRTT` approximates RTT variance without computing true stddev. High
  variance (bursty path) → wider margin → larger RTO; self-tuning instead
  of a fixed timeout.
- The `4x` multiplier is empirical, chosen to keep spurious timeouts rare
  under typical variance.

**Karn's algorithm** (the interview trap): never use a retransmitted
segment's ACK to update SampleRTT. If a segment times out, gets
retransmitted, and then an ACK arrives, you can't tell if the ACK is for
the original or the retransmission — the **retransmission ambiguity
problem**. Sampling it would corrupt the estimate. Instead:

- Skip RTT sampling for that segment entirely.
- Apply **exponential backoff**: `RTO = 2 * RTO` after each consecutive
  timeout, until finally ACKed — then resume normal EWMA sampling on the
  next fresh (non-retransmitted) segment.

Same backoff pattern recurs in gRPC/HTTP retry logic and
exponential-backoff-with-jitter in distributed systems generally — a
recurring interview motif, not just a TCP quirk.

**Reading:** K&R §3.5.3 (derivation); RFC 6298 (canonical formula + Karn's
algorithm); Van Jacobson's 1988 SIGCOMM paper "Congestion Avoidance and
Control" (original reasoning, bridges into §3.7 congestion control).

---

**Next topic:** K&R §3.7 — Congestion Control (AIMD, slow start, congestion
avoidance, fast retransmit/fast recovery).
