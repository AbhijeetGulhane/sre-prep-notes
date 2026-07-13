# Container Networking from Scratch: Network Namespaces, veth Pairs, Bridges

> Networking gap remediation #3 (iximiuz.com-style lab). Hands-on rebuild of what CNI plugins automate, done manually with raw `ip` commands on WSL2.

## The three core primitives

**Network namespace (netns)** — kernel feature giving a process its own isolated interfaces, routing tables, iptables rules, own loopback — fully separate from the host. The actual mechanism containers use for network isolation; Docker/containerd/CRI-O create one per container, Kubernetes just orchestrates it at scale. Default: everything shares the root netns. A new netns starts genuinely empty — no `eth0`, no internet reachability — until wired up.

**veth pair** — two virtual interfaces always created together, always connected like a virtual ethernet cable (whatever enters one end exits the other). Standard pattern: one end stays in root namespace (or attaches to a bridge), other end moves into a container's netns, renamed `eth0` from inside. This is the actual mechanism behind every Pod's `eth0` seen in `ss -tnp` output — one end of a veth pair whose other end lives on the host.

**Linux bridge** — a virtual switch, kernel-level L2 device that multiple veth ends (or physical interfaces) attach to; forwards traffic between them like a physical Ethernet switch, using MAC addresses. How same-node containers/Pods reach each other — this is the `cni0`/`cbr0` bridge referenced in basic CNI setups.

**Maps directly to a single node's real Pod networking:** every Pod = a netns, connected via a veth pair to a bridge; cross-node traffic then uses whatever the CNI's cross-node mechanism is (VXLAN encap or BGP routing — from the CNI plugins topic).

## Lab: built it by hand on WSL2

```bash
# Create two namespaces ("containers")
sudo ip netns add container1
sudo ip netns add container2

# Create a bridge (the virtual switch)
sudo ip link add name br0 type bridge
sudo ip link set br0 up

# veth pair for container1: one end to bridge, one end into the namespace
sudo ip link add veth1-host type veth peer name veth1-ns
sudo ip link set veth1-host master br0
sudo ip link set veth1-host up
sudo ip link set veth1-ns netns container1

# Inside container1: rename to eth0, assign IP, bring up
sudo ip netns exec container1 ip link set veth1-ns name eth0
sudo ip netns exec container1 ip addr add 10.10.10.11/24 dev eth0
sudo ip netns exec container1 ip link set eth0 up
sudo ip netns exec container1 ip link set lo up

# Repeat for container2 with 10.10.10.12/24
```

**Result — ping across the bridge:**
```
sudo ip netns exec container1 ping -c 3 10.10.10.12
```
```
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 0.105/0.181/0.323/0.100 ms
```
Sub-millisecond RTT as expected — entirely virtual/in-kernel, no physical wire.

**Isolation confirmed:**
```
sudo ip netns exec container1 ip addr show
```
Only shows `lo` and its own `eth0` (`10.10.10.11`) — zero visibility into `container2`. Genuinely separate network stacks, not just separate IPs on a shared view.

**Watched the actual L2 forwarding on the bridge:**
```
sudo tcpdump -i br0 -n
```
```
IP 10.10.10.11 > 10.10.10.12: ICMP echo request
IP 10.10.10.12 > 10.10.10.11: ICMP echo reply
```
Caught the real packets crossing the bridge — the exact flow the CNI notes describe abstractly, this time wired by hand instead of automated by Calico/Flannel.

## Observation worth remembering: `tunl0@NONE` appeared unprompted

`container1`'s `ip addr show` included a `tunl0@NONE` (IPIP tunnel device) interface that was never explicitly created. Explanation: the kernel auto-creates a default `tunl0` in *every new* network namespace whenever the `ipip` kernel module is already loaded system-wide — which it was here, because **Calico uses IPIP encapsulation mode** on this same minikube cluster. Genuine proof the hand-built lab and the running Calico CNI share the same underlying kernel machinery, not just a conceptual analogy.

**Interview-relevant takeaway:** if `tunl0` (or similar unexpected interfaces) show up in a namespace/Pod you didn't explicitly configure, check what kernel modules are loaded system-wide and what CNI/networking tooling triggered them — it's a real diagnostic signal, not noise, when troubleshooting unexpected interfaces in production.

## Cleanup
```bash
sudo ip netns delete container1
sudo ip netns delete container2
sudo ip link delete br0
```

## Interview one-liners

- "A network namespace is what actually isolates a container's network stack — Docker/containerd create one per container, Kubernetes just orchestrates that at scale."
- "Every Pod's eth0 is one end of a veth pair — the other end lives on the host, attached to a bridge or the CNI's equivalent construct."
- "A Linux bridge is a virtual L2 switch — same-node Pods reach each other by having their veth pairs' host-side ends plugged into a shared bridge, exactly like physical switch hardware."
- "This is literally what every CNI plugin automates on Pod creation — netns + veth pair + bridge — the plugin just does it faster and ties it into Kubernetes' Pod lifecycle instead of you running `ip` commands by hand."

---
*Networking gap remediation block: Cloudflare Learning Center ✓, Ferrill chapters ✓, DNS trace lab ✓, this container networking lab ✓. Remaining: "Life of a Packet" KubeCon talk.*
