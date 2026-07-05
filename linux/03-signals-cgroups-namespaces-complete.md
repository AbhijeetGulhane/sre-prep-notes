# Signals, Cgroups v2, and Namespaces — Complete Study Notes

---

## PART A: Signals — Deep Internals

## 1. Generation vs. Delivery — Two Separate Events

A signal can be **generated** and then sit **pending** until it is **delivered**.

**Generation sources:**
- Hardware exception: SIGSEGV from a bad memory access, SIGFPE from divide-by-zero
- Syscall: `kill()`, `tgkill()`, `raise()`, `sigqueue()`
- Terminal control: Ctrl+C → SIGINT, Ctrl+\ → SIGQUIT, Ctrl+Z → SIGTSTP
- Kernel-generated: SIGCHLD when a child exits, SIGPIPE when writing to a broken pipe
- Timer: SIGALRM from `alarm()`, SIGVTALRM, SIGPROF

**Delivery:** Happens the next time the kernel checks pending signals — typically on return from a syscall or interrupt, when the process is about to resume execution in user space.

## 2. Signal Disposition — Three Options Per Signal

Every signal has exactly one disposition, set per-signal per-process:

| Disposition | Meaning |
|-------------|---------|
| `SIG_DFL` | Default action (varies by signal: Term/Core/Ign/Stop/Cont) |
| `SIG_IGN` | Explicitly ignore |
| Custom handler | Function installed via `sigaction()` |

**Two signals exempt from all three:** SIGKILL and SIGSTOP **cannot** be caught, blocked, or ignored — by kernel design, so there's always a guaranteed way to control any process.

## 3. Signal Reference Table

| Signal | # | Default | Common Use |
|--------|---|---------|-----------|
| SIGHUP | 1 | Term | Terminal disconnect; daemons repurpose as "reload config" |
| SIGINT | 2 | Term | Ctrl+C — keyboard interrupt |
| SIGQUIT | 3 | Core | Ctrl+\ — terminate with core dump |
| SIGILL | 4 | Core | Illegal instruction |
| SIGTRAP | 5 | Core | Debugger breakpoints |
| SIGABRT | 6 | Core | `abort()` call |
| SIGBUS | 7 | Core | Bus error (misaligned memory access) |
| SIGFPE | 8 | Core | Floating point / arithmetic exception |
| SIGKILL | 9 | Term | **Uncatchable, unblockable, unignorable** |
| SIGSEGV | 11 | Core | Invalid memory access |
| SIGPIPE | 13 | Term | Write to pipe/socket with no reader |
| SIGALRM | 14 | Term | `alarm()` timer expiry |
| SIGTERM | 15 | Term | Polite shutdown — what `kill` sends by default, what kubelet sends |
| SIGCHLD | 17 | Ign | Child state change (exit/stop/continue) |
| SIGCONT | 18 | Cont | Resume a stopped process |
| SIGSTOP | 19 | Stop | **Uncatchable, unblockable, unignorable** |
| SIGUSR1 | 10 | Term | Application-defined |
| SIGUSR2 | 12 | Term | Application-defined |

## 4. Masking and Pending State

A process can **block** a signal via `sigprocmask()` (single-threaded) or `pthread_sigmask()` (multi-threaded). A blocked signal that is generated becomes **pending** and is delivered once unblocked.

**Standard signals** track pending-ness as a **single bit** — sending SIGUSR1 three times while blocked delivers it exactly once when unblocked. The extra two are silently dropped.

**Real-time signals** (`SIGRTMIN` through `SIGRTMAX`, typically 34–64) use a **queue** — every send is preserved. All three deliveries happen when unblocked.

**Inspection:** `/proc/[pid]/status` exposes four hex bitmasks:
```
SigPnd: 0000000000000000   # pending signals
SigBlk: 0000000000000000   # blocked signals
SigIgn: 0000000000000004   # ignored signals (bit 2 = SIGQUIT in this example)
SigCgt: 0000000000010000   # caught signals (have custom handlers)
```
Decode: `kill -l` lists signals in order (signal N = bit N-1).

## 5. Async-Signal-Safety — The Interview-Critical Concept

A signal handler **interrupts whatever the process was doing at an arbitrary point** — potentially mid-way through a non-reentrant library call.

**The malloc deadlock scenario:**
1. Main thread is inside `malloc()`, has acquired the internal heap lock, is manipulating heap metadata
2. Signal arrives → kernel suspends main thread → jumps to signal handler
3. Handler calls `printf()` → `printf()` internally calls `malloc()` → tries to acquire the heap lock
4. The heap lock is **already held by the same thread** (sitting in the handler)
5. Single-threaded deadlock: waiting forever on a lock it already holds and will never release

**The rule:** A signal handler may only call functions on POSIX's async-signal-safe list:
- `write()`, `_exit()`, `signal()`, `sigaction()`, `kill()`
- A handful of other low-level, fully reentrant primitives
- **NOT safe:** `malloc`, `free`, `printf`, `fprintf`, `syslog`, `openlog`, essentially all of libc

**What's safe to do in a handler:**
- Set a `volatile sig_atomic_t` flag (let the main loop notice and act)
- Write a single byte to a pipe (the self-pipe trick)
- Call `_exit()` for fatal signals

## 6. The Self-Pipe Trick

Converts an asynchronous, restrictive signal delivery into an ordinary, safe fd-readiness event.

**Setup:**
```c
int pipefd[2];
pipe(pipefd);
fcntl(pipefd[1], F_SETFL, O_NONBLOCK);  // non-blocking write end
// Add pipefd[0] to epoll set as readable
```

**Signal handler:**
```c
void handler(int sig) {
    char byte = sig;
    write(pipefd[1], &byte, 1);  // async-signal-safe
}
```

**Main event loop:**
```c
// epoll_wait() returns when pipefd[0] becomes readable
// Read the byte, identify which signal, handle it fully in normal context
// No async-signal-safety constraints here
```

**Why it's necessary:** The signal is inherently asynchronous — can interrupt the main loop at any point, no control over when. The handler can't do real work safely. The pipe converts it into a synchronous, pollable readiness event that fits naturally into the existing epoll model.

**Modern alternative: `signalfd()`** — tells the kernel to block signal delivery and instead deliver signal data as readable data on a file descriptor. Same concept, kernel-native: block the signal via `sigprocmask()`, open `signalfd`, add to `epoll` set, read structured `signalfd_siginfo` structs directly.

## 7. SIGCHLD, Zombies, and Orphans

**When a child exits:**
1. Child transitions to **zombie** (Z state) — all resources freed except the `task_struct` holding the exit status
2. Kernel sends **SIGCHLD** to the parent
3. Parent must call `wait()`/`waitpid()` to **reap** the child — read its exit status and let the kernel free the `task_struct`

**If the parent never reaps:** Zombie lingers indefinitely, consuming a PID slot. A system accumulating thousands of zombies can exhaust the PID namespace.

**If the parent exits before the child:** Child becomes an **orphan** — re-parented to PID 1 (init or, in a container, whatever is PID 1 in that PID namespace). PID 1 is responsible for reaping orphaned children.

**Why this matters for containers:** If your application binary runs directly as PID 1 (the container's `ENTRYPOINT`), it inherits the responsibility of reaping all orphaned processes in the container. If it doesn't call `wait()`, zombies accumulate. `tini` and `dumb-init` handle this correctly — they're designed to be proper init processes.

## 8. Kubernetes SIGTERM → Grace Period → SIGKILL — Full Sequence

When a pod is deleted (via `kubectl delete`, a Deployment scale-down, a rolling update, a node drain, etc.):

**Step 1:** Pod is set to `Terminating` phase. Its IP is removed from the Service's Endpoints/EndpointSlice (in parallel with the rest of the sequence — there is a brief race window where traffic can still arrive).

**Step 2:** **`preStop` hook runs first** — kubelet executes the preStop hook and waits for it to complete or time out. SIGTERM is **not sent** until preStop finishes.

**Step 3:** **SIGTERM sent** to PID 1 in each container simultaneously.

**Step 4:** The application should:
- Stop accepting new incoming connections
- Drain in-flight requests (with a bounded timeout — can't wait forever)
- Flush buffered data (logs, metrics, writes)
- Close database connections cleanly
- Call `exit(0)` or equivalent

**Step 5:** Either:
- Application exits within `terminationGracePeriodSeconds` (default 30s, configurable) → clean shutdown ✅
- Grace period expires → kubelet sends unconditional **SIGKILL** → container dies immediately, no cleanup possible ❌

**The preStop sleep pattern:** A common real-world preStop hook is simply:
```yaml
preStop:
  exec:
    command: ["/bin/sleep", "15"]
```
Not because the app needs 15 seconds — but because Service endpoint removal takes time to propagate through kube-proxy to all nodes. A sleep buys time for traffic to stop arriving before the app actually starts shutting down, preventing connection resets to a pod that's in the middle of graceful shutdown.

## 9. The PID 1 Signal Quirk — Famous Production Gotcha

**The rule:** The Linux kernel does **not** apply the default action for a signal to PID 1 (the init process of any PID namespace) unless a custom handler for that signal has been explicitly installed.

**Why this exists:** The root PID namespace's PID 1 (the system init — systemd, SysV init) must survive — accidentally killing it with a stray SIGTERM would be catastrophic. This protection was extended to PID 1 in any PID namespace for consistency.

**The container manifestation:**
- Your application binary is directly the `ENTRYPOINT` → it is PID 1 in the container's PID namespace
- Your application has SIGTERM-handling code — but if the handler registration is somehow not running (race condition, early startup, or the framework handles it differently), or if the framework doesn't explicitly register a handler...
- SIGTERM arrives at PID 1 → no handler registered → kernel silently ignores it (does NOT apply default "terminate" action)
- Application keeps running, not reacting to SIGTERM
- Every pod termination burns the full 30-second grace period before SIGKILL fires
- SIGKILL always works (it's exempt from this protection)

**Symptoms:** Pod always takes exactly `terminationGracePeriodSeconds` to terminate. No evidence the application tried to shut down gracefully. Kubernetes reports `OOMKilled` vs `Completed` shows the SIGKILL exit code (137 = 128 + 9).

**Also common:** Shell-script as ENTRYPOINT. The shell becomes PID 1, launches your app as a child. Shell may or may not forward SIGTERM to its child process. Even if it does, the child must also handle it.

**The fix:** `tini` or `dumb-init` as PID 1:
```dockerfile
ENTRYPOINT ["tini", "--", "myapp"]
# or
ENTRYPOINT ["/usr/bin/dumb-init", "--", "myapp"]
```
These are minimal init processes designed to:
1. Run as PID 1
2. Forward signals to their child processes
3. Reap zombie children

Your application now runs as PID 2+, subject to normal signal semantics.

## 10. Signals Debugging Toolkit

```bash
kill -l                          # list all signals with numbers
kill -SIGTERM <pid>              # send SIGTERM (same as kill -15)
kill -9 <pid>                    # send SIGKILL (always works)
strace -p <pid> -e trace=signal  # trace signal delivery live
/proc/[pid]/status               # SigPnd/SigBlk/SigIgn/SigCgt bitmasks
```

---

## PART B: Cgroups v2 — Unified Hierarchy

## 11. What Cgroups Solve

Cgroups (control groups) limit, account for, and isolate resource usage of a collection of processes. Without cgroups, any process can consume all CPU, memory, disk I/O, or network bandwidth on a system.

## 12. v1 vs v2 — Key Differences

**Cgroup v1 (legacy):**
- Each resource controller (cpu, memory, blkio, etc.) has its **own independent hierarchy**
- A process can be in different cgroups for different controllers simultaneously
- This leads to inconsistencies and complexity — e.g., cpu and memory controllers can disagree about which group a process belongs to
- Still widely used (RHEL 7, older K8s)

**Cgroup v2 (unified hierarchy):**
- Single tree mounted at `/sys/fs/cgroup`
- All controllers share one hierarchy
- A process belongs to exactly one cgroup which covers all controllers
- Structural rule: a non-root cgroup either holds processes directly OR delegates to child cgroups with controllers — not an arbitrary mix
- Required for full Kubernetes feature support (memory QoS, PSI pressure stall info, etc.)

## 13. The v2 Tree Structure

```
/sys/fs/cgroup/                   ← root cgroup
├── cgroup.controllers            ← controllers available system-wide
├── cgroup.subtree_control        ← controllers enabled for children
├── memory.max                    ← root memory limit (usually "max")
│
├── system.slice/                 ← systemd-managed system services
│   ├── sshd.service/
│   └── ...
│
└── kubepods/                     ← kubelet's cgroup root
    ├── besteffort/               ← BestEffort pods
    ├── burstable/                ← Burstable pods
    │   └── pod-<uid>/           ← Pod-level cgroup
    │       ├── container-<id>/  ← Per-container cgroup
    │       └── ...
    └── guaranteed/              ← Guaranteed pods
```

Every directory has:
- `cgroup.procs` — PIDs of processes directly in this cgroup
- `cgroup.subtree_control` — which controllers are active for child cgroups

## 14. Controllers — The Important Ones

### Memory Controller

| File | Purpose |
|------|---------|
| `memory.current` | Current memory usage of this cgroup (including descendants) |
| `memory.max` | Hard limit → OOM kill if exceeded |
| `memory.high` | Soft throttle point → aggressive reclaim before hitting max |
| `memory.low` | Protection floor → resist reclaim under system pressure |
| `memory.min` | Hard protection floor → never reclaim below this |
| `memory.events` | Counters: low, high, max, oom_kill events |
| `memory.stat` | Detailed breakdown: anon, file, slab, sock, etc. |

**memory.max vs memory.high — why you want both:**
- `memory.max` alone = cliff edge: usage is fine → spike → OOMKilled, with nothing in between
- `memory.high` = pressure zone below max: kernel applies aggressive reclaim and throttles allocating processes, slowing the workload and giving it a chance to shed memory before hitting the cliff
- Set `memory.high` meaningfully below `memory.max` to create a buffer zone

### CPU Controller

| File | Purpose |
|------|---------|
| `cpu.max` | "quota period" — e.g. "50000 100000" = 50ms of CPU per 100ms period = 0.5 CPUs |
| `cpu.weight` | Proportional share (default 100, range 1-10000) — like nice values for cgroups |
| `cpu.stat` | Usage stats: nr_periods, nr_throttled, throttled_usec |
| `cpu.pressure` | PSI (Pressure Stall Information) for CPU pressure |

### IO Controller

| File | Purpose |
|------|---------|
| `io.max` | Per-device read/write bandwidth and IOPS limits |
| `io.weight` | Proportional weight for I/O scheduling |
| `io.stat` | Per-device read/write bytes and IOPS |

### Other Controllers

| Controller | File | Purpose |
|-----------|------|---------|
| pids | `pids.max` | Maximum number of processes/threads — fork bomb defense |
| cpuset | `cpuset.cpus` | Pin to specific CPU cores |
| cpuset | `cpuset.mems` | Pin to specific NUMA nodes |

## 15. Cgroup Driver Consistency

The **cgroup driver** setting — `systemd` vs `cgroupfs` — controls how the container runtime creates and names cgroups. This must match between:
- The container runtime (containerd, CRI-O)
- kubelet (`--cgroup-driver` flag)

A mismatch causes node-provisioning failures and is a common real-world operational problem. Both should be set to `systemd` on modern systems that use systemd as the init system.

---

## PART C: Namespaces — The Isolation Layer

## 16. "A Container Is Just a Process" — The Full Picture

There is no `struct container` anywhere in the Linux kernel. A container is an ordinary Linux process that happens to have been started with a specific combination of three things:

1. **Namespaces** — what the process can *see*
2. **Cgroup placement** — what the process can *use*
3. **Pivot-rooted filesystem** — what files the process can *access*

All three are standard Linux primitives that predate "containers" as a concept. The container runtime (containerd, CRI-O) is just tooling that wires them together in a specific way at process creation via `clone()` flags.

## 17. The Seven Namespace Types

| Namespace | Flag | Isolates |
|-----------|------|---------|
| PID | `CLONE_NEWPID` | Process tree — container's main process is PID 1 within its namespace even if PID 34982 on the host |
| Mount | `CLONE_NEWNS` | Filesystem view — independent mount table |
| Network | `CLONE_NEWNET` | Network interfaces, routing table, iptables rules — own loopback, own eth0 |
| UTS | `CLONE_NEWUTS` | Hostname and domain name |
| IPC | `CLONE_NEWIPC` | SysV IPC objects, POSIX message queues |
| User | `CLONE_NEWUSER` | UID/GID mapping — "root" inside (UID 0) maps to unprivileged UID on host |
| Cgroup | `CLONE_NEWCGROUP` | View of the cgroup hierarchy — paths relative to container's own cgroup root |

## 18. How a Container Process Is Created

```
runtime (e.g. containerd)
    └── calls runc with OCI runtime spec (JSON)
         └── runc calls clone() with namespace flags:
               CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS |
               CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER
             └── In the new process:
                   1. pivot_root() into the overlayfs-backed rootfs
                   2. Mount /proc, /dev, /sys inside the new mount namespace
                   3. Set hostname (UTS namespace)
                   4. The process is placed into the cgroup kubelet created
                   5. execve() the container's ENTRYPOINT
```

The container's process is PID 1 within its own PID namespace. From the host, `ps aux` shows it as some large PID (e.g. 45821). Inside the container, `ps aux` shows it as PID 1.

## 19. Kubernetes Pod Namespace Sharing Model

**Shared across all containers in a pod (by default):**
- Network namespace — same IP, same port space, reach each other via `localhost`
- IPC namespace — shared SysV IPC and POSIX message queues
- UTS namespace — same hostname

**Per-container (isolated):**
- Mount namespace — each container has its own filesystem view
- Cgroup — each container has its own resource accounting and limits
- PID namespace — isolated by default

**`shareProcessNamespace: true` on the pod spec:** Shares the PID namespace across all containers — one container can see and signal processes in another. Used for sidecar-based debugging (e.g., a debug sidecar that can inspect the main container's process tree) and process-supervision sidecars.

**Why network namespace is pod-scoped:** This is the fundamental reason containers in a pod are always co-located on one node and can communicate cheaply (localhost). The pause container holds this shared network namespace open even when app containers restart.

## 20. The Pause Container

Every Kubernetes pod has a hidden "infra" container — the `pause` binary — running as PID 1 in the pod's namespaces before any app containers start.

**Why it exists:** Namespaces are tied to processes — they exist as long as at least one process holds them open. If your app container is the only process in the network namespace and it crashes, the network namespace (and the Pod's IP) would be destroyed. Any replacement container would get a new IP, disrupting everything routing to the pod.

**The `pause` solution:** pause holds the network and IPC namespaces open. It is a minimal binary that does nothing in a loop. App containers join its namespaces. When an app container crashes and kubelet restarts it, pause is still running, the namespace persists, the Pod's IP is preserved.

**Visible on a node:** `crictl ps` or `docker ps` on a node always shows one pause container per pod, alongside the app containers.

## 21. Namespace Debugging Toolkit

```bash
# See which namespaces a process is in
ls -la /proc/[pid]/ns/
# Each file is a symlink like: net -> net:[4026531992]
# The number is the namespace inode — processes sharing a namespace have the same inode

# Enter a running process's namespaces from the host
nsenter --target <pid> --net --pid --mount
# Essential when a container has no shell or debugging tools

# Create a new namespace for testing
unshare --net --pid --mount bash

# View the live cgroup tree
systemd-cgls
systemd-cgtop     # live resource usage by cgroup

# Find which cgroup a process is in
cat /proc/[pid]/cgroup

# Inspect a container's network from the host
PID=$(docker inspect --format '{{.State.Pid}}' <container>)
nsenter --target $PID --net ip addr
nsenter --target $PID --net ss -tulpn
```

---

## Retention Questions and Answers

**Q1: Why can't you safely call printf() inside a signal handler, mechanically — what's the actual failure mode?**

`printf()` calls `malloc()` to manage formatting buffers. `malloc()` protects its heap metadata with an internal lock. If the main thread is interrupted mid-way through a `malloc()` call — while holding that lock — and the signal handler calls `printf()` (which calls `malloc()`), the handler tries to acquire the same lock. That lock is already held by the same thread (sitting in the handler). Single-threaded deadlock: the thread waits forever on a lock it already holds and will never release. This is why POSIX defines a narrow list of async-signal-safe functions and `printf`/`malloc`/`free` are explicitly not on it. The only safe operations in a handler are fully reentrant, lock-free primitives like `write()` and `_exit()`.

**Q2: Walk through the self-pipe trick and explain why it's necessary given async-signal-safety constraints.**

Setup: call `pipe()`, set the write end `O_NONBLOCK`, add the read end to your epoll set.

Signal handler: `write(write_fd, &byte, 1)` — a single byte. `write()` is async-signal-safe, so this is legal.

Main loop: `epoll_wait()` returns when the read end becomes readable. Read the byte, identify the signal, handle it fully in normal context with no async-signal-safety constraints.

Why necessary: the signal interrupts the main loop at an arbitrary point. The handler has almost no safe operations available. The pipe converts the asynchronous, unpredictable signal delivery into a synchronous, pollable readiness event — the same kind of event the main epoll loop already handles naturally. Modern alternative: `signalfd()` — block the signal via `sigprocmask()`, open a signalfd, add to epoll, read `signalfd_siginfo` structs directly.

**Q3: A pod takes the full 30 seconds to terminate every single time, even though the app has SIGTERM-handling code. What's your first hypothesis?**

The application is running as PID 1 in its container's PID namespace without a properly registered SIGTERM handler. The Linux kernel does not apply the default action (terminate) for a signal to PID 1 in any PID namespace unless a handler for that signal is explicitly installed. If the handler registration isn't happening (or the app binary is running inside a shell script that becomes PID 1 and doesn't forward signals), SIGTERM is silently swallowed. SIGKILL still fires when the grace period expires (it's exempt from this protection), which is why the pod does eventually die — just always the slow, ungraceful way. Fix: run `tini` or `dumb-init` as PID 1 to properly forward signals to the app process.

**Q4: What's the difference between memory.max and memory.high in cgroup v2, and why would you want both?**

`memory.max` is the hard limit — if the cgroup's usage reaches it, the kernel runs direct reclaim, and if that fails to bring usage under the limit, the OOM killer fires and kills a process in that cgroup. Abrupt, no warning, no graceful handling.

`memory.high` is a soft throttle point below `memory.max` — when usage crosses it, the kernel applies increasingly aggressive reclaim and slows down processes doing allocation inside that cgroup. Not a kill — a "slow down" signal.

Why both: `memory.high` creates a pressure zone below the cliff of `memory.max`. A memory spike crosses `memory.high` → kernel reclaims aggressively → workload slows and sheds memory → usage drops back below `memory.high` before reaching `memory.max`. Without `memory.high`, usage goes from normal to OOMKilled with nothing in between.

**Q5: Explain precisely what "a container is just a process" means — what three things, concretely, differentiate a containerized process from an ordinary one?**

There is no kernel container object. A container is an ordinary process started with three things:

1. **Namespace isolation** — what it can see. The process is started with `clone()` namespace flags (CLONE_NEWPID, CLONE_NEWNET, CLONE_NEWNS, CLONE_NEWUTS, CLONE_NEWIPC, possibly CLONE_NEWUSER). It has its own process tree (PID 1 in namespace), its own network stack, its own filesystem view, its own hostname. These are generic Linux primitives, not container-specific kernel features.

2. **Cgroup placement** — what it can use. The runtime creates a cgroup and moves the container's init process into it, translating resource limits into controller files (`memory.max`, `cpu.max`). The kernel enforces these limits regardless of how the cgroup was created — it knows nothing about "containers."

3. **Pivot-rooted filesystem** — what files it can access. An overlayfs filesystem is set up (layering image layers with a writable top layer) and `pivot_root()` makes it the process's `/`. The process sees a complete Linux filesystem layout from its perspective, with no access to the host's actual root filesystem.

**Q6: Two containers in the same pod can reach each other via localhost but have independent CPU limits. What does that tell you about which namespaces and cgroups are shared versus per-container?**

`localhost` connectivity means they share the **network namespace** — same IP address, same loopback interface, same port space. By extension, they also share the IPC and UTS namespaces by default (pod-level sharing).

Independent CPU limits means each container has its **own cgroup** — its own `cpu.max` enforced independently by the kernel. Usage is accounted and limited per-container, not per-pod (though the pod-level parent cgroup aggregates usage upward automatically).

Mount namespace is also per-container by default — each has its own filesystem view. PID namespace is per-container by default unless `shareProcessNamespace: true` is set on the pod spec.

Clean summary: **pod-scoped shared** = network, IPC, UTS namespaces (why containers in a pod have the same IP and can communicate via localhost). **Per-container** = cgroup (resource accounting and limits), mount namespace (filesystem), PID namespace (process tree, by default).
