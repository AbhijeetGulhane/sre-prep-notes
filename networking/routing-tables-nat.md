# Routing Tables and NAT

> Networking gap remediation #2 complete (Ferrill hours 1.5-3: NAT + Routing Concepts). Full theory deep-dive.

## The routing table — how the kernel decides "where next"

Every IP-stack device maintains a **routing table**: destination network ranges → next hop (gateway IP or local interface). Every outbound packet gets a routing table lookup before leaving the machine.

```bash
ip route show
```
```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.42
```
Second line: `192.168.1.0/24` directly reachable on `eth0`, no gateway (local subnet, ARP resolves MAC directly). First line (`default`): catch-all — anything unmatched by a more specific route goes to `192.168.1.1` via `eth0`.

## Longest-prefix match — the core algorithm

When multiple routes match a destination, the kernel picks the **most specific** (longest subnet mask). Given routes to `10.0.0.0/8`, `10.1.0.0/16`, `10.1.1.0/24`, a packet to `10.1.1.5` matches all three — kernel picks `/24` since it's most specific. Lets networks have broad routes for general reachability plus specific carve-outs (a VLAN, VPN range) without needing explicit priority ordering — longest-prefix-match handles precedence automatically.

**Default route (`0.0.0.0/0`)** matches everything (0-length prefix = "don't care" on all 32 bits) — only wins when nothing more specific exists. "Default gateway" and "default route" are the same concept: the route entry for `0.0.0.0/0`.

## Direct delivery vs routed delivery

1. **Directly connected** — destination in a subnet the machine has an interface on (`scope link` route, no gateway). ARP for destination's MAC directly, send frame straight there — pure L2 delivery, no router involved.
2. **Needs a gateway** — destination elsewhere. ARP for the *gateway's* MAC (never the final destination's — could be anywhere), send the frame to the gateway. Gateway repeats its own routing lookup. This hop-by-hop repetition — each router only knowing "next hop," not the full path — is the actual mechanism of internet routing, and why it scales (no router needs the complete path to every destination).

## NAT — the problem it solves

**Private IP ranges (RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) aren't routable on the public internet** — no backbone router forwards to `192.168.1.42` (meaningless outside your own network, millions of networks reuse these ranges). Nearly every network uses them internally because IPv4's ~4 billion addresses ran out decades ago.

**NAT** rewrites source IP/port of outbound packets from private → router's public IP, tracks the mapping, rewrites incoming replies back before internal delivery. From the internet's view, every device behind the router looks like one public IP.

## SNAT vs DNAT — direction determines the name

**SNAT (Source NAT)** — rewrites the **source** address of outbound packets. Normal home router behavior. Most common form: **PAT (Port Address Translation)** / "NAT overload" — since many devices share one public IP, source **port** also gets rewritten so each device's connections stay distinguishable. The (public IP, source port) pair uniquely identifies which internal device+connection a reply belongs to.

**DNAT (Destination NAT)** — rewrites the **destination** address of incoming packets. Exactly the ClusterIP mechanism from the kube-proxy lab: packet arrives addressed to ClusterIP, DNAT rewrites destination to a real Pod IP. Home router port forwarding is the same mechanism — packets to public IP:port get destination rewritten to internal IP:port.

**Unifying insight:** SNAT and DNAT are the same rewriting mechanism applied to different packet fields depending on direction — the exact same conntrack-based mechanism as the ClusterIP lab, just running on a home router instead of a K8s node.

## Connection tracking — makes NAT bidirectional

Router SNATs an outbound packet → records an entry in **conntrack** (same kernel subsystem from the kube-proxy lab): original (private IP, private port) ↔ translated (public IP, public port), keyed by the full 5-tuple.

Reply arrives addressed to (public IP, public port) → router looks up conntrack, finds the entry, **un-translates** destination back to original private IP:port before forwarding internally. Same "original direction / reply direction" pairing seen in the earlier `conntrack -L` ClusterIP lab output — home router NAT and K8s ClusterIP NAT are mechanically the same feature.

**Why unsolicited inbound connections get dropped:** no existing conntrack entry (no prior outbound connection) means no mapping to reverse-translate against — dropped by default. NAT incidentally acts like a firewall even though its actual purpose is address conservation. Inbound connections need explicit port-forwarding (static DNAT rules) precisely because there's no conntrack entry otherwise.

## Full path: laptop → internet server → back

1. Laptop's routing table lookup finds no directly-connected route → matches `default` → packet to home router via ARP-resolved MAC
2. Router does its own lookup → not locally connected → matches its own default route toward ISP
3. **Before forwarding to ISP, NAT does SNAT+PAT**: rewrites source (private IP, random high port) → (public IP, translated port), records in conntrack
4. Packet traverses internet hop-by-hop, each router doing its own longest-prefix-match lookup — no NAT at any of those hops (NAT only at network edges)
5. Server replies to (public IP, translated port)
6. Router looks up conntrack, **DNATs back** to (laptop's private IP, original port), forwards internally

**One-liner:** "Routing decides where a packet goes next using longest-prefix match against the routing table; NAT decides what address the packet appears to come from or go to, using conntrack to remember the original mapping so reply traffic can be un-translated correctly — independent mechanisms that happen to run together at every network edge, one deciding the path, the other rewriting addresses along it."

## Interview one-liners, compiled

- "The routing table lookup happens on every outbound packet without exception — the kernel picks the longest-prefix (most specific) matching route, falling back to the default route (0.0.0.0/0) only when nothing more specific matches."
- "Directly connected destinations get ARP'd and delivered at L2 directly; everything else gets forwarded to a gateway, which repeats its own routing decision — no router needs to know the full path, just the next hop."
- "NAT exists because private IP ranges aren't internet-routable and IPv4 doesn't have enough addresses for every device — SNAT rewrites source (normal outbound, usually with PAT), DNAT rewrites destination (port forwarding, or K8s ClusterIP translation)."
- "Conntrack is what makes NAT bidirectional — it remembers the original-to-translated mapping so reply packets can be un-translated back to the right internal device, and it's the exact same mechanism whether it's a home router or a Kubernetes node's kube-proxy."

---
*Networking gap remediation #2 complete (Ferrill 1.5-3hrs, NAT + Routing). Remaining from the original remediation plan: Cloudflare Learning Center reading (if not already covered), Life of a Packet KubeCon talk — already completed per earlier session.*
