# DNS Lab: `dig +trace` walkthrough + WSL2 quirk

> Week 4 — DNS day, hands-on lab companion to `dns-resolution-record-types.md`.

## WSL2 quirk: `dig +trace google.com` fails, `dig +trace @8.8.8.8 google.com` works

**Symptom:**
```
$ dig +trace google.com
;; communications error to 10.255.255.254#53: timed out  (x3)
;; no servers could be reached
```
...while plain `dig google.com` (no +trace) works fine through the same server.

**Why:** `+trace` starts by querying whatever's in `/etc/resolv.conf` for the root zone's NS set — a different query pattern than an ordinary A-record lookup. WSL2's default NAT networking mode runs a lightweight DNS proxy on `10.255.255.254` that forwards normal lookups to Windows fine, but doesn't reliably handle this specific root-hint query. Known WSL2 quirk, not a broken host.

**Workaround (used for this lab):**
```
dig +trace @8.8.8.8 google.com
```
`@8.8.8.8` only redirects the *first* query (root hints) to a real public resolver — every subsequent hop still queries the actual root, `.com` TLD, and `ns*.google.com` authoritative servers directly. This is normal practice, not a shortcut that invalidates the lab.

**Real fix (optional, broader change):** switch WSL2 from NAT to mirrored networking (Windows 11 22H2+), which removes the DNS-proxy layer entirely:
```
# %UserProfile%\.wslconfig
[wsl2]
networkingMode=mirrored
```
Then `wsl --shutdown` and reopen. Only worth doing if hitting other WSL2 networking issues too — not solely for this.

**Other noise to ignore in trace output:** `UDP setup with ... failed: network unreachable` lines are `dig` trying IPv6 addresses for root/TLD/auth servers first; WSL2 NAT has no real IPv6 route out, so it falls back to IPv4 automatically. Not a DNS chain problem, just local networking.

## Trace walkthrough (against `dig +trace @8.8.8.8 google.com`)

| Hop | Server | What it returned | Maps to theory |
|---|---|---|---|
| 1 | 8.8.8.8 itself | All 13 root server NS records, + RRSIG | Answered directly from 8.8.8.8's own root hints/cache — not yet a real referral |
| 2 | `b.root-servers.net` (real root server) | `.com` TLD NS records (all 13), + DS record for `com.` | The actual iterative referral: root → TLD delegation |
| 3 | `b.gtld-servers.net` (`.com` TLD server) | `google.com`'s NS records (ns1-4.google.com), + NSEC3 records | TLD → authoritative delegation |
| 4 | `ns2.google.com` (authoritative) | `google.com. A 142.251.211.78` | Final authoritative answer (aa flag) — end of chain |

## Bonus: DNSSEC artifacts seen in the trace (adjacent topic, not on plan yet)

Because 8.8.8.8 does DNSSEC validation, the trace surfaced real DNSSEC record types:

- **RRSIG** — signature over a record set, proves it wasn't tampered with in transit (seen on the root NS set)
- **DS** (Delegation Signer) — at the root→`.com` hop, cryptographically vouches that `.com`'s zone-signing key is legitimate, chaining trust from root's key down
- **NSEC3** — appeared instead of a DS record at the `.com`→`google.com` hop; proves a *negative* ("`google.com` has no DS record, and here's cryptographic proof this absence is real, not stripped by an attacker"). Confirms `google.com` itself isn't DNSSEC-signed, but the proof-of-absence is.

Worth a dedicated DNSSEC session later if targeting roles with heavy DNS/security surface area — not required for the current plan.

## Tip for next time
Run `dig +trace +additional @8.8.8.8 google.com` to see glue A/AAAA records explicitly in the TLD referral step (they were present but not printed in the default trace output above).

---
*DNS day fully complete: resolution chain, record types, zone-apex CNAME restriction, and hands-on trace lab. Ready for whatever's next on the tracker.*
