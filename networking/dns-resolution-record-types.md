# DNS: Resolution Chain and Record Types

> Week 4 — Networking fundamentals, DNS day.

## Recursive vs iterative resolution

**Actors:**
1. **Stub resolver** — OS/app-level resolver (e.g. glibc). Does no real work, just asks the recursive resolver and waits.
2. **Recursive resolver** — ISP resolver, 8.8.8.8, 1.1.1.1, or corporate resolver. Does the actual legwork.
3. **Root servers** — 13 logical (anycast) addresses; know only where TLD servers are.
4. **TLD servers** — e.g. `.com`; know where a domain's authoritative nameservers are.
5. **Authoritative nameservers** — source of truth for a specific zone.

**Iterative query:** the queried server responds with either the final answer or a referral ("ask this other server") — it never chases the chain on your behalf. The *querier* iterates through referrals itself.

Chain for `google.com`:
- Resolver → root server → referral to `.com` TLD servers
- Resolver → `.com` TLD server → referral to `ns1.google.com` (+ friends)
- Resolver → `ns1.google.com` (authoritative) → actual A record, final answer

**Recursive query:** the stub resolver sends one query to the recursive resolver and expects a final answer — the recursive resolver is obligated to chase the full iterative chain itself and return one result.

**Key relationship:** one recursive query in (from stub resolver) → N iterative queries out (to root, TLD, authoritative) — same server plays both roles simultaneously.

**One-liner:** "Recursive queries expect a final answer and put the burden of chasing referrals on the server; iterative queries can return a referral, and it's up to the querier to follow it."

**Caching:** every response carries a TTL; recursive resolvers cache records/delegation info so the full cold-cache root→TLD→authoritative walk is the worst case, not the norm. DNS propagation delay after a change is bounded by TTL.

### Lab: `dig +trace google.com`

Simulates the recursive resolver's iterative walk from your own machine — starts at root, manually follows referrals, shows every hop instead of delegating to your configured resolver.

Order of output:
1. Query to a root server → referral to `com.` TLD nameservers, with **glue records** (A records for those NS names) in ADDITIONAL section
2. Query to a `.com` TLD server → referral to `google.com`'s authoritative nameservers, again with glue
3. Query to an authoritative server → actual `A` record, response has the `aa` (authoritative answer) flag set

**Glue records:** TLD/parent servers include the IP of the referred nameserver directly (not just its name) to avoid a chicken-and-egg lookup problem. Only exist for in-bailiwick nameservers (nameserver's own name is inside the zone it serves).

`aa` flag appears only on the final, authoritative response — root/TLD responses are referrals, not answers. Each hop shows real RTT — use `+trace` to debug "DNS is slow" or broken/missing-glue delegation issues in production.

## DNS record types

| Type | Purpose | Notes |
|---|---|---|
| **A** | Hostname → IPv4 | Most common record |
| **AAAA** | Hostname → IPv6 | Same role as A, different address family |
| **CNAME** | Alias one hostname to another hostname (not an IP) | Cannot coexist with any other record type at the same name (RFC restriction) |
| **MX** | Mail servers for a domain, with priority | `priority hostname`, lower = preferred; target must itself resolve via A/AAAA, can't point directly to an IP |
| **TXT** | Arbitrary text on a name | SPF/DKIM/DMARC, domain ownership verification (Search Console, ACME), general metadata |
| **PTR** | IP → hostname (reverse) | Lives in `in-addr.arpa` (v4) / `ip6.arpa` (v6) reverse zones; used for reverse lookups, e.g. mail server reputation checks |
| **NS** | Delegates a zone to authoritative nameservers | This is the delegation mechanism seen at every `+trace` hop |
| **SOA** | Start of Authority — one per zone | Primary NS, admin contact, serial number (incremented per zone change, drives secondary zone transfers), refresh/retry/expire intervals, negative-caching TTL |

### CNAME vs A

- **A/AAAA**: name points directly to an IP you control.
- **CNAME**: name should always mirror another name's resolution — e.g. `www.example.com` → `example.com`, or pointing at a third-party hostname whose IPs may change (e.g. `blog.example.com` → `ghs.googlehosted.com`).

### Can you CNAME the zone apex? — No

The **zone apex** (`example.com`, not `www.example.com`) cannot have a CNAME. RFC 1034/1035: the apex must have SOA (and typically NS) records, and CNAME can't coexist with any other record type at the same name. Since SOA/NS are mandatory at the apex, CNAME is structurally impossible there.

**Workaround — ALIAS / ANAME (non-standard, provider-specific):**
- Configured like a CNAME (points at a hostname), behaves like an A record externally
- The DNS provider resolves the target server-side and returns A/AAAA records to the client — legal at the apex because it's not actually a CNAME RRTYPE on the wire
- Not standardized — behavior varies per provider (Cloudflare/DNSimple: ALIAS/ANAME; AWS Route 53: ALIAS record for pointing apex at ELB/CloudFront/S3)

**One-liner:** "You can't CNAME the zone apex because CNAME can't coexist with the mandatory SOA/NS records that must live there — the workaround is a provider-specific ALIAS/ANAME record, which looks like a CNAME in config but is resolved server-side into A records so it's legal at the apex."

---
*DNS day complete. Next: TLS 1.3 / mTLS deep dive (already covered at theory level in Week 3-4 background — check tracker for whether a dedicated hands-on day remains) or continue per tracker sequence.*
