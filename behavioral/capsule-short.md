# capsule — Short Behavioral Answer

Use this when asked: "Tell me about a project you built" or "What have you
done hands-on recently with Linux internals?"

---

## The 60-Second Version

"I built a minimal container runtime from scratch in Go — called capsule —
to understand what Docker and the kubelet actually do at the kernel level.

It uses Linux namespaces for isolation: UTS namespace for hostname, mount
namespace with pivot_root to swap the root filesystem into an Alpine rootfs,
PID namespace so the workload becomes PID 1 with its own process view. It
wires up cgroups v2 for CPU and memory limits — the same mechanism Kubernetes
uses. It has a PID-1 init that reaps orphaned zombie processes and forwards
signals, exactly what tini does in Docker. And there's a user namespace flag
for rootless operation — container root maps to my unprivileged UID, no sudo.

There's also a raw C companion that does the same isolation with a direct
clone(2) syscall so you can strace it and see exactly what the kernel does.

The whole point was to be able to answer 'what is a container, actually?' by
having written one."

---

## The 30-Second Version

"I wrote a container runtime in Go from scratch — namespaces, cgroups v2,
pivot_root, PID-1 zombie reaping and signal forwarding, user namespace for
rootless. Plus a C companion using raw clone() you can strace. The goal was
to understand what Docker does at the kernel level, not to replace it."

---

## One-Line Version

"A container runtime from scratch in Go — namespaces, cgroups v2, pivot_root,
PID-1 init — to understand what Docker does underneath."

---

## Follow-up Hooks

| What you say | Follow-up they'll ask |
|---|---|
| "pivot_root to swap the root filesystem" | Why pivot_root and not chroot? |
| "PID namespace so workload becomes PID 1" | What's special about PID 1? |
| "reaps orphaned zombie processes" | Show me the zombie demo |
| "cgroups v2 for CPU and memory limits" | How does cpu.max work? |
| "rootless — container root maps to my UID" | How does user namespace mapping work? |
| "raw C companion you can strace" | What does the strace output show? |
| "re-exec pattern" | Why can't you just call unshare() in Go? |

---

## Key Numbers to Drop Naturally

- Phases: 6 (UTS+Mount, PID, PID-1 init, cgroups v2, user+net namespace, C companion)
- Namespaces used: 6 (UTS, Mount, PID, IPC, User, Net)
- Zombie demo: 5 orphans → STAT:Z without init, 0 zombies with init
- cgroup CPU format: `"<quota_us> <period_us>"` e.g. "50000 100000" for 0.5 CPU
- cgroup memory: raw bytes, no "M" suffix
- Rootfs: Alpine minirootfs, ~2.7MB
- Commits: 5 (one per phase)

---

## The Zombie Demo — Know This Cold

This is the most concrete story in the project:

"I wrote a C program — `orphan_maker` — that spawns 5 orphaned grandchildren
using a double-fork. Middle children exit immediately, orphaning the grandchildren
to PID 1. Without my init, PID 1 was the workload, it never called wait(), and
all 5 showed as Z-stat — zombie — in ps. With my PID-1 init running a Wait4 loop
on SIGCHLD, all 5 were reaped immediately. Zero zombies. The before/after ps
output is the demo — 5 defunct processes disappear when you add the reaper.

This is exactly the problem tini solves in Docker. And it's why every K8s pod
without a SIGTERM handler in its PID 1 burns the full 30-second grace period —
PID 1 ignores signals by default unless a handler is registered."

---

## The Re-exec Pattern — Know This Cold

"Go's runtime is multi-threaded before main() starts. Calling unshare() in Go
would only affect one OS thread — other threads stay in the old namespace. So
instead of unshare(), I use exec.Command with SysProcAttr.Cloneflags. This
triggers clone(2) at the kernel level when creating the child process — the new
process starts in fresh namespaces before Go's runtime initializes. The binary
re-execs itself with a sentinel argument to know it's the child path. Config
travels via environment variables. This is the same pattern runc uses."

---

## WSL2 Caveats — Name Them Proactively

Shows operational awareness:

"Three things behaved differently on WSL2 than on bare metal. Memory OOM
enforcement didn't fire — Microsoft patched it out — but memory.events still
showed the limit engaging so I could verify the cgroup was working. The cpu
controller wasn't pre-delegated in subtree_control so I had to write '+cpu'
explicitly. And network namespace isolation for backgrounded processes was
incomplete. All three are documented in the README as known caveats. On a
real Linux host or production K8s node, all three work correctly."

---

## If They Ask "What Would You Add Next?"

"Three natural next phases. First, real container networking — a veth pair to
the host, bridge, NAT via iptables. This is what Flannel and Calico do as CNI
plugins, and I understand the steps. Second, overlayfs for copy-on-write image
layers — so you could run multiple containers from the same base image without
duplicating the rootfs. Third, seccomp filtering and capability dropping — the
security layer Docker adds on top of namespace isolation."

This shows you understand the boundaries you drew and what lies beyond them.

---

## The K8s Connection — Use This

If the interviewer is focused on Kubernetes:

"Every K8s concept has a capsule primitive underneath it. CPU limits use
cpu.max in cgroups v2 — CFS bandwidth throttling, same mechanism. Memory
limits use memory.max — cgroup-scoped OOM killer, exit code 137, same
mechanism. The tini/dumb-init pattern is my reaper.go. The network namespace
CNI sets up is exactly what CLONE_NEWNET creates — just with a veth pair added.
The user namespace is how rootless containers in Podman work. Building capsule
means I can explain what K8s does at the kernel level, not just what it does
at the YAML level."
