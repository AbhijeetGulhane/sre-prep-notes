# capsule — Complete Interview Reference

**For behavioral and technical interviews. Every detail, every decision, every number.**

---

## 1. The One-Paragraph Answer

"I built a minimal container runtime from scratch in Go — named capsule — that
recreates what Docker and the kubelet do at the Linux kernel level. It uses
namespaces for isolation: UTS for hostname, mount for the filesystem via
pivot_root, PID for process isolation with a fresh /proc, user namespace for
rootless operation, and network namespace for an isolated network stack. It
wires up cgroups v2 to enforce CPU and memory limits. It has a PID-1 init mode
that reaps orphaned zombie processes and forwards signals — exactly what tini
does in Docker. There's also a raw C companion that does the same isolation
with clone(2) directly, so you can strace it and see every kernel call. The
whole thing is documented with per-phase design notes and WSL2 caveats. The
goal wasn't to build another Docker — it was to be able to answer 'what is a
container, actually?' by having written one."

---

## 2. Why This Project Exists

**The gap it closes:** 11 years of Linux infrastructure ops, but the explanation
of container internals was always "Docker uses namespaces and cgroups" without
having directly called those APIs. capsule closes that gap by going from
conceptual knowledge to written, running code.

**The interview payoff:**
- Answers "explain what a container actually is" with code you wrote
- Provides Go evidence (search-api was Python/K8s)
- Ties directly back to every K8s concept — kubelet uses exactly these primitives
- Generates multiple concrete STAR stories

**Why Go, not Python/Bash:**
Go's `syscall` package exposes Linux primitives directly — `clone`, `pivot_root`,
`sethostname`, `mount`, `unshare`. It's statically compiled (single binary,
no runtime deps). It's also what runc (the OCI runtime under Docker and containerd)
is written in. The C companion shows the same thing at an even lower level.

---

## 3. Project Structure

```
capsule/
├── go.mod                    # module github.com/AbhijeetGulhane/capsule
├── main.go                   # arg dispatch: run (parent) vs child (re-exec)
├── run.go                    # parent: namespace flags, cgroup setup, re-exec
├── child.go                  # child: hostname, pivot_root, /proc, exec workload
├── reaper.go                 # PID 1 init: zombie reaping + signal forwarding
├── cgroup.go                 # cgroup v2: cpu.max + memory.max
├── util.go                   # must(), envOr(), boolEnv()
├── c/
│   ├── mini_container.c      # same isolation via raw clone(2) — strace target
│   └── orphan_maker.c        # spawns orphaned grandchildren for zombie demo
├── scripts/
│   ├── fetch-rootfs.sh       # pulls Alpine minirootfs
│   └── zombie-demo.py        # Python version of the zombie demo
├── Makefile                  # build, c, orphan, rootfs, clean targets
├── docs/
│   ├── DESIGN.md             # per-phase technical notes + WSL2 caveats
│   └── INTERVIEW.md          # interview revision sheet
└── README.md                 # build/run/verify + deliberately-not-included
```

---

## 4. The Phases — What Was Built and Why

### Phase 0 — Recon (before writing any code)

Before writing a line, verified what the host kernel exposes:

```bash
lsns                          # list all namespaces on the host
ls -la /proc/self/ns/         # show current process's namespace symlinks
cat /proc/self/cgroup         # confirm cgroup v2 (single line: 0::/...)
cat /sys/fs/cgroup/cgroup.controllers  # available controllers
```

This is why the code works: confirmed the host had all 7 namespace types
and cgroup v2 unified hierarchy before trying to use them.

### Phase 1 — UTS + Mount Namespace (Filesystem Isolation)

**UTS namespace** (`CLONE_NEWUTS`): isolated hostname. `sethostname()` inside
the container doesn't affect the host. Simplest namespace to verify:
```
inside:  hostname → capsule (or --hostname value)
outside: hostname → JeetPC
```

**Mount namespace** (`CLONE_NEWNS`): isolated mount table.

**pivot_root** — the correct way to change a container's root filesystem:
```
1. Mount / as MS_PRIVATE|MS_REC     → stop mount propagation to host
2. Bind-mount rootfs to itself       → make it a mount point (pivot_root requirement)
3. pivot_root(".", ".put_old")       → swap current root with rootfs
4. mount("proc", "/proc", "proc")    → fresh procfs (PID namespace view)
5. umount2("/.put_old", MNT_DETACH)  → detach old root
6. rmdir("/.put_old")               → clean up
```

**Why pivot_root, not chroot?**
`chroot` only changes the path resolution root — it doesn't change the actual
root mount. It's escapable: a root-privileged process can `chroot(".")` back up
the tree. `pivot_root` changes the actual mount table entry for `/` — the
container genuinely has a different root filesystem, not just a different name
resolution starting point. runc uses `pivot_root`. Docker uses `pivot_root`.

**Why bind-mount rootfs to itself first?**
`pivot_root` requires that `new_root` is a mount point (has an entry in the
mount table). A plain directory is not a mount point. Bind-mounting a directory
to itself (`mount(rootfs, rootfs, MS_BIND|MS_REC)`) creates that entry. This
is a well-known kernel requirement for `pivot_root`.

**Why MS_PRIVATE|MS_REC on / first?**
Without this, mounts inside the container propagate to the host via shared
mount propagation (Linux's default). Running `mount /proc` inside would add
a `/proc` entry to the host's mount table too. `MS_PRIVATE` breaks that
propagation relationship.

**The rootfs:** Alpine Linux minirootfs (~2.7MB tarball). Pulled by
`scripts/fetch-rootfs.sh` from the official Alpine CDN. Contains: busybox
symlinks for sh/ls/ps/mount/id, basic /etc files, no kernel, no packages
beyond minimal. The workload (`/bin/sh`, `/bin/ps`, etc.) lives here.

### Phase 2 — PID Namespace

`CLONE_NEWPID`: the child process gets its own PID number space. The workload
becomes PID 1 in its namespace, but has a different PID from the host's view.

**Why mount a fresh /proc?**
The `/proc` filesystem is namespace-aware — it reflects the PID namespace of
the process that mounted it. If you don't remount `/proc` in a new PID namespace,
you get the *host's* `/proc` view showing all host processes. After `pivot_root`,
we explicitly `mount("proc", "/proc", "proc", 0, 0)` to get the container's view.

```
inside (after phase 2): ps aux → only processes inside the container
outside:                 ps aux → all host processes, including the container
```

**Verify:** Inside the container, `echo $$` shows PID 1. The same process on
the host is visible with a much higher PID (it exists in both namespaces
simultaneously — two different numbers for the same process).

### Phase 3 — PID-1 Init: Zombie Reaping + Signal Forwarding

**The PID 1 problem — the most important interview story in this project:**

When a process exits, it becomes a zombie (`STAT: Z`, `<defunct>`) until its
parent calls `wait()` to read its exit status. If the parent exits before
doing `wait()`, the orphaned child gets reparented to PID 1. PID 1 is
responsible for reaping all orphaned processes by calling `wait()`.

If the workload (e.g., `/bin/sh`) is PID 1 and it exits, the kernel reaps it
normally. But if the workload spawns child processes and exits without reaping
them, those children become orphans reparented to PID 1. If PID 1 is the shell
and it just exited, there's no one calling `wait()` → zombies accumulate
indefinitely → PID table leaks.

**The demo — `c/orphan_maker.c`:**
```c
for i in 0..5:
    fork() → middle child
        fork() → grandchild (sleeps 1s, then exits)
        middle child exits immediately     ← grandchild is now orphaned
```
Result: 5 grandchildren reparented to PID 1. If PID 1 never `wait()`s, all 5
show as `Z` in `ps`.

**Without --init:**
```bash
sudo ./capsule run /orphan_maker
# another shell: ps -eo pid,ppid,stat,comm
# → 5 processes with STAT=Z (defunct)
```

**With --init:**
```bash
sudo ./capsule run --init /orphan_maker
# another shell: ps -eo pid,ppid,stat,comm
# → 0 processes with STAT=Z — all reaped
```

The `5 → 0` flip across the two runs is the demo. Clean, measurable, concrete.

**How reaper.go works:**
```go
func runInit(cmd []string) int {
    proc = exec.Command(cmd...)           // start workload as child
    signal.Notify(sigs)                   // capture all signals

    for s := range sigs {
        if s == syscall.SIGCHLD {
            for {                          // drain reaping queue
                pid, _ = syscall.Wait4(-1, WNOHANG)
                if pid == childPid → return exit code
                if pid <= 0 → break
                // else: reaped an orphan, loop again
            }
        } else {
            syscall.Kill(-childPid, s)     // forward to workload's process group
        }
    }
}
```

**Bug caught and fixed during development:**
Initial version used `proc.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}`.
This moved the workload into its own process group, but interactive shells
(`/bin/sh`) use `tcsetpgrp()` to take terminal control. With `Setpgid: true`
and the workload in a different process group, the shell received `SIGTTIN`
(terminal input from background process) and stopped immediately. Fix: the
`Setpgid: true` was needed for signal forwarding (so `kill(-pgid, sig)` reaches
the right processes) but the initial implementation broke the terminal. Corrected
by properly handling the terminal ownership.

**Why this matters for K8s:**
Every pod without a SIGTERM handler in its PID 1 process waits the full
`terminationGracePeriodSeconds` (default 30s) because the default PID 1
action is to ignore signals — the kernel doesn't apply signal defaults to PID 1.
tini (Docker's `--init`) and dumb-init solve exactly this by being the PID 1
that handles SIGTERM and forwards it to the workload.

### Phase 4 — cgroups v2: CPU and Memory Limits

**cgroup v2 (unified hierarchy):** Single tree at `/sys/fs/cgroup/`. One cgroup
can either hold processes OR distribute controllers to children — not both.
The root cgroup is exempt from this rule (has processes and distributes controllers).

**How capsule uses it:**
```
/sys/fs/cgroup/                     ← root (exempt from no-internal-processes rule)
  cgroup.subtree_control: +cpu +memory   ← delegate to children

/sys/fs/cgroup/capsule/             ← parent created by capsule
  cgroup.subtree_control: +cpu +memory   ← delegate to children

/sys/fs/cgroup/capsule/capsule-<pid>/   ← leaf: container lives here
  cpu.max: "50000 100000"              ← 0.5 CPU (50ms per 100ms period)
  memory.max: "67108864"               ← 64MB in bytes (no "M" suffix at kernel level)
  cgroup.procs: <child_pid>            ← move the process in
```

**CPU limit mechanism:**
`cpu.max = "<quota_us> <period_us>"`. `--cpus 0.5` writes `50000 100000`
meaning "50ms of CPU per 100ms period." This is CFS bandwidth control — the
exact same mechanism K8s uses for CPU limits. When the container exceeds 50ms
in a 100ms window, the kernel throttles it (pulls tasks off the run queue until
the next period). Not killed — throttled.

**Memory limit mechanism:**
`memory.max` is the hard limit in bytes. Set with `--memory-mb 64` →
`memory.max = 67108864` (64 × 1024 × 1024). Exceeding this triggers the
cgroup-scoped OOM killer — SIGKILL, exit code 137.

**WSL2 caveat documented:**
On WSL2, the memory OOM kill often doesn't fire. This is a Microsoft kernel
modification. However, `memory.events` still shows the limit engaging — the
`max` and `oom` counters increment even when no kill fires. capsule prints
`memory.events` after every `--memory-mb` run so this is visible:

```
[capsule] memory.events:
anon 12288
file 45056
...
max 3       ← limit was hit 3 times
oom 0       ← OOM kill didn't fire (WSL2 limitation)
```

**CPU controller addition:**
On some WSL2 configurations, the `cpu` controller is available but not yet
in `subtree_control`. capsule writes `+cpu +memory` to `cgroup.subtree_control`
at the root level before creating its cgroup tree. This was a debug step during
development — `memory` was already delegated, `cpu` needed to be added explicitly.

**cgroup lifecycle:**
Parent creates the cgroup, starts the child, adds child PID to `cgroup.procs`,
waits for child to exit, then `rmdir()` the leaf cgroup. The rmdir only succeeds
when the cgroup is empty (no processes). This is cgroup cleanup — important to
not leak cgroup directories.

### Phase 5 — User Namespace (Rootless) + Network Namespace

**User namespace (`CLONE_NEWUSER`, `--rootless` flag):**
Maps a container user ID to a host user ID. capsule maps container UID/GID 0
(root) to the caller's unprivileged UID/GID:

```go
child.SysProcAttr.UidMappings = []syscall.SysProcIDMap{
    {ContainerID: 0, HostID: os.Getuid(), Size: 1},
}
child.SysProcAttr.GidMappings = []syscall.SysProcIDMap{
    {ContainerID: 0, HostID: os.Getgid(), Size: 1},
}
child.SysProcAttr.GidMappingsEnableSetgroups = false  // required for unprivileged mapping
```

After this: inside the container, `id` shows `uid=0(root) gid=0(root)`.
On the host, the same process runs as the unprivileged user (UID 1000).
No `sudo` required for `./capsule run --rootless /bin/sh`.

**Written in `/proc/<pid>/uid_map`:** The mapping file format is
`container_id host_id size`. `0 1000 1` means: container UID 0 maps to host
UID 1000, mapping covers 1 UID. Anything the container does as "root" is
actually the unprivileged user to the host kernel.

**`setgroups` must be denied:** Writing "deny" to `/proc/<pid>/setgroups`
before writing the GID map is required for unprivileged GID mappings. Without
this, the kernel rejects the GID map write. This is a security requirement —
prevents privilege escalation via group removal.

**User namespace + cgroups:** Under `--rootless`, cgroup limits are typically
ignored because unprivileged users can't write to the system cgroup tree.
capsule warns when both `--rootless` and `--cpus`/`--memory-mb` are used.

**Network namespace (`CLONE_NEWNET`):**
The container gets an isolated network stack. On creation, only the loopback
interface (`lo`) exists — and it's DOWN.

```bash
# inside the container after Phase 5:
ip link
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN

ip link set lo up
ping 127.0.0.1   # works after bringing lo up
```

The kernel requires an explicit `ioctl(SIOCSIFFLAGS, IFF_UP)` to bring `lo`
up in a fresh network namespace — it's not automatic. This is exactly what
Kubernetes CNI plugins do as the first step when setting up a pod's network.

**WSL2 caveat:** `sudo unshare --net` for backgrounded processes in WSL2 doesn't
fully isolate. The `CLONE_NEWNET` in capsule's `Cloneflags` creates a proper
netns for the child, but loopback-only is the extent of what's demonstrated.
A full working container network would require a veth pair, host bridge, and
NAT — deliberately out of scope.

### Phase 6 — C Companion (`c/mini_container.c`)

The same isolation in raw C with `clone(2)` directly. No Go runtime, no
`SysProcAttr` abstraction — just the system call.

```c
int flags = CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWIPC | SIGCHLD;
pid_t pid = clone(child_fn, child_stack + STACK_SIZE, flags, &ca);
```

`clone(2)` creates a new process with a manual stack buffer. The child function
pointer runs in the new namespaces. This is exactly what runc does internally
— Go's runtime wraps `clone` with `Cloneflags`, C calls it directly.

**Why it exists:**
You can strace the C binary and see the exact kernel call sequence:
```bash
sudo strace -f -e trace=clone,pivot_root,mount,execve ./mini_container ./rootfs /bin/sh
```
Output shows every syscall in order — `clone(... CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWPID ...)`,
then `pivot_root(".", ".put_old")`, then `mount("proc", "/proc", "proc", ...)`,
then `execve("/bin/sh", ...)`. This is the raw kernel view with no abstraction.

---

## 5. The Re-exec Pattern — Critical to Understand

**Q: Why does capsule re-exec itself instead of using unshare()?**

Go's runtime is multi-threaded from the moment `main()` starts — goroutines,
the garbage collector, and the netpoller all run on multiple OS threads.
`unshare(CLONE_NEWNS)` affects only the calling thread. In a multi-threaded
process, other threads still have the old namespaces — you end up with a
split-namespace process where different threads are in different mount namespaces.
This is undefined behavior and causes crashes or unexpected behavior.

**The solution:** Use `exec.Command("/proc/self/exe", "child", ...)` with
`SysProcAttr.Cloneflags`. This calls `clone(2)` (not `unshare`) with the
namespace flags, creating a NEW process in the new namespaces from the start.
The new process has only one thread when it starts, before Go's runtime
initializes. By the time `childRun()` runs, the process is cleanly inside
the namespaces.

This is not a workaround — this is the standard pattern used by:
- runc (the OCI runtime under Docker and containerd)
- Liz Rice's "Containers From Scratch" (the canonical reference implementation)
- Any Go code that needs namespace isolation

The "sentinel arg" pattern: `main.go` checks `os.Args[1]` — if it's `"run"`,
it's the parent path. If it's `"child"`, it's the re-exec'd path. The config
the child needs (hostname, rootfs path, init flag) travels as environment
variables set by the parent.

---

## 6. The Code — Function by Function

### `main.go`
Pure dispatch — 20 lines. No logic. Reads `os.Args[1]` and routes to either
`parentRun()` or `childRun()`. Anything else prints usage and exits.

### `run.go` — The Parent Path
1. Parse flags: `--hostname`, `--rootfs`, `--cpus`, `--memory-mb`, `--init`, `--rootless`
2. Resolve rootfs to absolute path
3. Build `exec.Command("/proc/self/exe", "child", <workload_cmd>...)`
4. Set `Stdin/Stdout/Stderr` to current process's (terminal passthrough)
5. Set env vars: `CAPSULE_HOSTNAME`, `CAPSULE_ROOTFS`, `CAPSULE_INIT`
6. Set `SysProcAttr.Cloneflags`: always `CLONE_NEWUTS | CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET`, plus `CLONE_NEWUSER` if `--rootless`
7. If `--rootless`: set `UidMappings`, `GidMappings`, `GidMappingsEnableSetgroups: false`
8. If limits: call `newCgroup()` to create the cgroup leaf and set limits
9. `child.Start()` — kernel forks with the namespace flags
10. If cgroup: `cg.addPid(child.Process.Pid)` — move child into the cgroup
11. `child.Wait()` — block until workload exits
12. If cgroup: print `memory.events`, call `cg.remove()`
13. Propagate exit code

### `child.go` — The Child Path (runs inside new namespaces)
1. Read env vars: hostname, rootfs, init flag
2. `sethostname(hostname)` — UTS namespace
3. `mount("", "/", "", MS_REC|MS_PRIVATE, "")` — stop propagation
4. `mount(rootfs, rootfs, "", MS_BIND|MS_REC, "")` — make it a mount point
5. `chdir(rootfs)` — go there
6. `mkdir(".put_old")` — landing pad for old root
7. `pivot_root(".", ".put_old")` — swap root filesystem
8. `chdir("/")` — re-anchor to new root
9. `mount("proc", "/proc", "proc", 0, "")` — fresh /proc for PID namespace
10. `umount2("/.put_old", MNT_DETACH)` — detach old root lazily
11. `os.Remove("/.put_old")` — clean up
12. If `--init`: hand off to `runInit(cmd)`
13. Else: `syscall.Exec(path, cmd, env)` — replace self with workload (PID 1)

### `reaper.go` — PID 1 Init
1. Start workload as a child process (not as PID 1 — capsule is PID 1, workload is PID 2)
2. `signal.Notify(sigs)` — capture all signals
3. Loop:
   - On `SIGCHLD`: drain `Wait4(-1, WNOHANG)` — reap everything ready
     - If reaped PID is the workload: return its exit code
     - If reaped PID is anything else: an orphan — reaped and discarded (or logged)
   - On any other signal: `syscall.Kill(-childPid, s)` — forward to workload's process group

### `cgroup.go` — cgroup v2
1. `newCgroup(name, cpus, memMB)`:
   - Check `cgroup.controllers` at root for requested controllers
   - Write `+cpu +memory` to root `cgroup.subtree_control`
   - `mkdir /sys/fs/cgroup/capsule/` (parent dir for all capsule cgroups)
   - Write `+cpu +memory` to parent's `cgroup.subtree_control`
   - `mkdir /sys/fs/cgroup/capsule/<name>/` (leaf for this run)
   - If `--cpus`: write `"<quota> 100000"` to `cpu.max`
   - If `--memory-mb`: write bytes string to `memory.max`
2. `addPid(pid)`: write PID to `cgroup.procs`
3. `events()`: read `memory.events`
4. `remove()`: `rmdir` the leaf (only works when empty)

---

## 7. What It Demonstrates — Mapping to K8s

**Every K8s concept has a capsule primitive underneath:**

| K8s Concept | capsule Primitive | How It Works |
|-------------|------------------|--------------|
| Container isolation | UTS + Mount + PID namespace | CLONE_NEWUTS, CLONE_NEWNS, CLONE_NEWPID |
| Container root filesystem | pivot_root into Alpine rootfs | Swap mount table root entry |
| Pod network namespace | CLONE_NEWNET | Isolated network stack, lo only |
| CPU limit (`resources.limits.cpu`) | `cpu.max` in cgroup v2 | CFS bandwidth throttling |
| Memory limit (`resources.limits.memory`) | `memory.max` in cgroup v2 | cgroup-scoped OOM killer |
| OOMKilled (exit 137) | `memory.max` exceeded | Kernel sends SIGKILL to processes in cgroup |
| CPU throttling (not OOM) | `cpu.max` quota exhausted | CFS pulls process off run queue |
| tini / dumb-init | `reaper.go` + `--init` | wait4(-1, WNOHANG) on SIGCHLD loop |
| Pod having PID 1 signal issue | No `--init` + SIGTERM | PID 1 ignores signals by default |
| Rootless containers | CLONE_NEWUSER + UID mapping | Container root → host unprivileged UID |
| CNI plugin first step | `ip link set lo up` in netns | SIOCSIFFLAGS ioctl on fresh netns |
| containerd/runc exec | `childRun` → `syscall.Exec` | Replace process with workload |

---

## 8. Numbers and Facts to Know Cold

| Fact | Value |
|------|-------|
| Alpine minirootfs size | ~2.7MB |
| Number of namespace types used | 6 (UTS, Mount, PID, IPC, User, Net) |
| Number of Go source files | 6 (main, run, child, reaper, cgroup, util) |
| C files | 2 (mini_container.c, orphan_maker.c) |
| cgroup CPU limit format | `"<quota_us> <period_us>"` — e.g. "50000 100000" for 0.5 CPU |
| cgroup memory limit | Raw bytes — no "M" suffix — e.g. "67108864" for 64MB |
| Zombie demo: orphans spawned | 5 |
| Zombie demo result (no init) | 5 Z-stat processes in ps |
| Zombie demo result (with init) | 0 Z-stat processes — all reaped |
| Commit count | 5 (per-phase commits: aa5cb2e, a892af5, cc8cb99, bce55bb + Phase 6) |
| WSL2 kernel version | 6.1.83-microsoft-standard-WSL2 |

---

## 9. WSL2 Caveats — Know These Cold

These came up during development and are documented in DESIGN.md and README.md.
Being able to name them shows operational awareness, not ignorance.

**1. cgroup memory OOM enforcement doesn't fire:**
WSL2's kernel patches disable cgroup memory OOM enforcement. `memory.max` is
set correctly, `memory.events` shows the limit engaging (`max` counter increments),
but no SIGKILL fires. Fixed by printing `memory.events` output so the limit
engagement is still verifiable. On a real Linux host or in a production K8s node,
this fires correctly.

**2. `unshare --net` doesn't fully isolate backgrounded processes:**
WSL2's network namespace support for background processes launched with `&` is
incomplete. The network namespace in capsule works correctly for the child process
started with `clone()` flags, but `sudo unshare --net` on a backgrounded process
in a shell doesn't fully isolate. The capsule implementation is correct for the
production pattern.

**3. CPU controller not pre-delegated:**
On some WSL2 configurations, the `memory` controller is available in
`cgroup.subtree_control` but `cpu` is not. capsule explicitly writes `+cpu +memory`
to `cgroup.subtree_control` at both the root and parent level before creating the
leaf cgroup. This was discovered and fixed during development.

**4. User namespace may be restricted by kernel policy:**
Unprivileged user namespace creation (`CLONE_NEWUSER` without root) can be
restricted by `/proc/sys/kernel/unprivileged_userns_clone`. On the development
host this was enabled and `--rootless` worked. Some hardened systems set this
to 0 to prevent unprivileged namespace abuse.

---

## 10. STAR Stories — Ready to Tell

### Story 1 — The PID 1 Zombie Problem

**Situation:** Building a container runtime from scratch and needed to implement
PID-1 init behavior — the mechanism Docker uses `--init`/tini for.

**Task:** Demonstrate the zombie process problem concretely, then fix it, in
a way that's both educational and shows the mechanism clearly.

**Action:** Wrote `orphan_maker.c` in C — spawns 5 orphaned grandchildren via
a double-fork pattern. Middle children exit immediately, orphaning the
grandchildren to PID 1. Ran capsule without `--init` so the workload was PID 1
and never called `wait()`. Observed 5 processes with `STAT=Z` in `ps`. Then
implemented `reaper.go` — a `Wait4(-1, WNOHANG)` loop triggered by `SIGCHLD`,
draining all reparented orphans. Ran capsule with `--init`. Zero zombie processes.

Caught a bug during implementation: `Setpgid: true` moved the workload into
its own process group for signal forwarding, but interactive shells use
`tcsetpgrp()` to take terminal control — the shell received `SIGTTIN` and
stopped immediately. Fixed by correctly handling terminal process group ownership.

**Result:** Working PID 1 init that correctly reaps all orphaned processes.
The before/after `ps` output shows the `5 → 0` flip cleanly. This is exactly
the problem tini solves in Docker and why every K8s pod without a SIGTERM
handler burns the full 30-second grace period.

---

### Story 2 — Understanding the Re-exec Pattern

**Situation:** Started building the container runtime in Go and hit a fundamental
problem: Go's runtime is multi-threaded, and `unshare()` is per-thread. Calling
`unshare(CLONE_NEWNS)` from a Go goroutine would only affect that OS thread —
other threads would still be in the old namespace.

**Task:** Find the correct pattern for namespace isolation in a multi-threaded
Go runtime that's used in production container runtimes.

**Action:** Researched how runc handles this — it uses the re-exec pattern.
Instead of calling `unshare()`, the parent passes `SysProcAttr.Cloneflags` to
`exec.Command()`. This triggers `clone(2)` at the kernel level when starting
the child process, creating a new single-threaded process in the new namespaces
before Go's runtime starts. The child re-execs `/proc/self/exe` with a sentinel
argument (`"child"`). The parent/child distinction happens at `os.Args[1]`.
Config passes via environment variables.

**Result:** Clean namespace isolation without any multi-threading issues.
The C companion (`mini_container.c`) demonstrates the same pattern with raw
`clone(2)` and a manual stack buffer, showing exactly what the Go layer abstracts.
This pattern is used by runc, Kata Containers, and every production Go container
runtime.

---

### Story 3 — Debugging cgroup Controller Delegation

**Situation:** Implementing cgroup v2 CPU limits. Had the code written, ran
`--cpus 0.5`, got an error writing to `cpu.max`.

**Task:** Debug why the cpu controller wasn't accessible in the leaf cgroup.

**Action:** Checked `/sys/fs/cgroup/cgroup.controllers` — showed `cpuset cpu io memory hugetlb pids rdma misc`. The cpu controller was available. Checked
`/sys/fs/cgroup/cgroup.subtree_control` — showed only `memory pids`. The cpu
controller existed but wasn't delegated down the hierarchy. In cgroup v2, a
controller must be explicitly added to `subtree_control` at every level of the
path before it can be used in leaf cgroups. Added `+cpu` to `cgroup.subtree_control`
at both the root level and the `capsule/` parent directory. Also added to the
explicit `subtree_control` write in `cgroup.go` so it's handled automatically.

**Result:** CPU limits working. Learned the cgroup v2 no-internal-processes rule
and the requirement to explicitly delegate controllers at every hierarchy level —
the same mechanism Kubernetes uses when the kubelet sets up pod cgroups.

---

## 11. Hard Questions and Answers

**"What is a container, actually?"**
A container is a process (or group of processes) isolated from the rest of the
system using Linux kernel primitives — specifically namespaces for isolation and
cgroups for resource limits. There is no "container" struct in the kernel. When
you run `docker run ubuntu`, Docker calls `clone(2)` with namespace flags (the
same way capsule does), sets up a root filesystem via pivot_root, writes cgroup
limits, and executes the workload. The process runs in the Linux kernel like
any other process — it just has a restricted view of the system.

**"What's the difference between capsule and Docker?"**
Docker adds: image format (OCI layers, overlayfs for copy-on-write filesystems),
image registry pull (authentication, layer deduplication), networking (veth pairs,
bridge network, NAT via iptables), security (seccomp profiles, AppArmor/SELinux,
capability dropping), a daemon (dockerd manages container lifecycle), CRI compatibility
(can be plugged into K8s via containerd), and a full CLI. capsule has: namespace
isolation, cgroup limits, pivot_root, PID-1 init, and user namespace. The kernel
mechanisms are identical — Docker has more of them and more production-hardening
around each.

**"Why pivot_root and not chroot?"**
`chroot` changes the path resolution root but doesn't change the actual root mount.
A root-privileged process inside a chroot can escape by calling `chroot(".")` on
a directory outside the chroot. `pivot_root` swaps the actual root mount entry in
the VFS mount table — the old root is genuinely detached (we umount it with
`MNT_DETACH`). There's no path back to the original filesystem from inside the
container. This is why runc uses `pivot_root`.

**"Why does Go need the re-exec pattern for namespaces?"**
Go starts multiple OS threads before `main()` runs — the garbage collector,
goroutine scheduler, and netpoller all use OS threads. `unshare(2)` is per-thread.
Calling `unshare(CLONE_NEWNS)` would put only the calling OS thread in a new
mount namespace, while other threads remain in the old one — a split-namespace
process that causes undefined behavior. The re-exec pattern avoids this by using
`clone(2)` via `SysProcAttr.Cloneflags`, which creates a new single-threaded
process in the new namespaces before Go's runtime initializes.

**"What would it take to give containers real networking?"**
Currently capsule creates a `CLONE_NEWNET` namespace with only a (down) loopback.
To give containers real network connectivity you need: (1) create a veth pair on
the host (`ip link add veth0 type veth peer name eth0`), (2) move one end into
the container's netns (`ip link set eth0 netns <pid>`), (3) assign IPs to both
ends, (4) add a route, (5) configure NAT on the host side (iptables MASQUERADE
rule). This is exactly what Flannel and Calico do — they're CNI plugins that
automate exactly these steps when Kubernetes schedules a pod.

**"Why did you use Alpine minirootfs?"**
Minimal — the tarball is ~2.7MB, total rootfs is ~5-6MB. It has busybox (which
provides sh, ps, ls, mount, id, ping), basic /etc files, and nothing else. No
init system, no service manager, no unnecessary files. A larger base image would
work too, but Alpine is the standard for minimal container images — same reason
most Docker base images use Alpine.

**"What happens to cgroup limits under --rootless?"**
They're typically ignored. Writing to `/sys/fs/cgroup/capsule/<name>/cpu.max`
requires write permission on the cgroup filesystem, which is owned by root.
An unprivileged user can only write to cgroups in a subtree explicitly delegated
to them — which requires system configuration (e.g., systemd `Delegate=yes`).
In the standard WSL2 setup, `--rootless --cpus 0.5` runs but the CPU limit
doesn't enforce. capsule warns when both flags are combined. This is the same
limitation Podman's rootless mode has without `newuidmap`/`newgidmap` setup.

**"The cgroup memory OOM didn't fire on WSL2. Is your code wrong?"**
No. `memory.max` is set correctly — confirmed by reading back the file. The
`memory.events` output shows `max` counter incrementing when the limit is hit.
The kernel is correctly tracking that the cgroup hit its memory ceiling. The OOM
kill not firing is a deliberate Microsoft decision in the WSL2 kernel — they
patched out the enforcement to avoid interfering with Windows memory management.
On a real Linux host, bare metal or a production VM, the same code triggers the
cgroup OOM killer and the process exits with code 137. Documented in README as
a known caveat, not a bug.

**"How does this connect to what you do at Barclays?"**
The 60,000-node Linux estate at Barclays runs on physical servers and VMs managed
by shell scripts, Ansible, and monitoring tooling. Most services run as bare
processes or in Docker containers. Understanding what Docker actually does at the
kernel level — the namespaces, the cgroup limits, the pivot_root — means that
when a container has an OOM kill or an unexpected process termination, I can
reason about it from first principles rather than just escalating to "the
container is broken." capsule is the hands-on version of that understanding.

---

## 12. Key Technical Vocabulary

Use these terms precisely — interviewers notice when candidates confuse them:

**namespace vs cgroup:**
- Namespace = isolation (what you can SEE). Namespaces control visibility.
- Cgroup = resource accounting and limits (what you can USE). Cgroups control consumption.
- A container uses both — namespaces for isolation, cgroups for limits.

**pivot_root vs chroot:**
- chroot = changes path resolution root only, escapable, no mount table change
- pivot_root = changes actual root mount in VFS, old root detached, not escapable

**clone vs fork vs unshare:**
- `fork(2)` = creates child process, inherits all namespaces
- `clone(2)` = like fork but with flags to create new namespaces simultaneously
- `unshare(2)` = calling process leaves shared namespace, per-thread

**zombie vs orphan:**
- Zombie = exited process whose exit status hasn't been read by its parent (STAT: Z)
- Orphan = process whose parent has exited, reparented to PID 1
- Orphans become zombies if PID 1 doesn't reap them

**cgroup v1 vs cgroup v2:**
- v1 = per-controller hierarchies, multiple trees under /sys/fs/cgroup/
- v2 = unified single hierarchy, all controllers under one tree
- capsule uses v2 only. K8s supports both; v2 required for memory QoS

**PID 1 signal behavior:**
- Regular process: unhandled signal → default action (terminate, core dump, etc.)
- PID 1: unhandled signal → silently ignored by kernel (protection against accidental termination)
- Why it matters: container workload that ignores SIGTERM as PID 1 never terminates gracefully
