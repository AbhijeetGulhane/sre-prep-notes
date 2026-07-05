# VFS, File Descriptors, epoll, CFS Scheduler, /proc — Complete Study Notes

---

## PART A: File Descriptors, Inodes, and the VFS Layer

## 1. The VFS Object Model

VFS (Virtual Filesystem Switch) defines a common object model that every concrete filesystem plugs into via tables of function pointers. This is what makes `open()`/`read()`/`write()`/`close()` work identically whether you're touching ext4, XFS, NFS, tmpfs, procfs, a pipe, or a socket.

Four core VFS objects:

| Object | Represents | Kernel struct |
|--------|-----------|---------------|
| Superblock | One mounted filesystem instance | `super_block` + `super_operations` |
| Inode | One unique file (metadata only) | `inode` + `inode_operations` |
| Dentry | One path component (name → inode mapping) | `dentry` |
| File | One open instance of a file | `file` + `file_operations` |

Each object has an `*_operations` table of function pointers — this is where ext4's implementation differs from NFS's or procfs's, behind the same syscall surface.

## 2. The Three-Level Indirection Chain

This is the most important structural concept — worth having completely solid.

```
Per-process fd table (files_struct)
         │
         │  [fd 3] ──────────────────────→ Open file description (struct file)
         │                                      │  offset: 1024
         │                                      │  flags: O_RDWR
         │                                      │  refcount: 2
         │                                      │  file_operations: ext4_ops
         │                                      │
         │  [fd 5] ─────────────────────────────┘  (also points here → shared offset)
         │
         │  [fd 7] ──────────────────────→ Open file description (struct file)
                                                │  offset: 0
                                                │  (independent, from separate open())
                                                │
                                                └────→ Inode (struct inode)
                                                           permissions, owner, size,
                                                           timestamps, link count,
                                                           data block pointers
```

**Level 1 — Per-process fd table (`files_struct`, hanging off `task_struct`):**
- Just an array indexed by small integers (0, 1, 2...)
- Each slot is a pointer to a `struct file`
- fd 0 = stdin, fd 1 = stdout, fd 2 = stderr (by convention)

**Level 2 — Open file description (`struct file`, system-wide):**
- Current file **offset**
- Access mode and status flags (`O_RDONLY`, `O_APPEND`, `O_NONBLOCK`, `O_CLOEXEC`)
- Reference count (how many fd table entries point here)
- Pointer to `file_operations` function table
- Pointer to the dentry/inode

**Level 3 — Inode (`struct inode`):**
- The actual file metadata: permissions, owner UID/GID, size, timestamps (atime/mtime/ctime)
- Hard link count (`nlink`)
- Pointers to data blocks (or extent trees in modern filesystems)
- One inode per unique file, regardless of how many names point to it

## 3. The Fork/Dup Gotcha — Interview Favorite

**`fork()`** duplicates the fd *table* (Level 1), creating independent integer arrays in parent and child — but both entries still point at the **same** open file description (Level 2). Same offset, same flags, same refcount (now incremented). If the child reads 100 bytes, the parent's next read continues from byte 100.

**`dup()`/`dup2()`** creates another fd table slot pointing at the *same* open file description. `dup2(oldfd, newfd)` is how shell redirection works: make newfd point at the same file description as oldfd.

**Two independent `open()` calls** on the same path: each gets its own `struct file` with its own offset — even though both ultimately point at the same inode. Independent offsets.

**Diagnostic question:** Two processes hold fd 3 open, both pointing to the same file, but reads from one don't affect the other's offset. Answer: each opened the file independently with `open()`, giving each its own `struct file`. If they had shared an offset, it would mean one was inherited via `fork()` or explicitly duplicated.

## 4. Dentries and Path Resolution

The **dentry cache (dcache)** maps path components to inodes, avoiding repeated filesystem lookups. Every `open("/usr/lib/libc.so")` resolves `usr` → dentry → inode → `lib` → dentry → inode → `libc.so` → dentry → inode.

**Hard link:** Two dentries in (possibly different) directories pointing at the same inode number. The inode's `nlink` count is incremented. To the filesystem, both names are equally "real" — neither is the "original."

**Symlink:** A distinct inode whose data content is just a path string. Followed (re-resolved) on every access.

## 5. Deletion While Open — The Classic Disk Space Gotcha

`unlink()` removes the dentry and decrements `nlink`. The inode and its data blocks are **not freed** until both the link count AND the open-reference count hit zero.

**Production scenario:** A 50GB log file is deleted with `rm` while a logging service still has it open. `df` still shows 50GB consumed. `du` shows nothing (no directory entry to find). The file physically exists and takes space until the process closes it or restarts.

**Find these:** `lsof +L1` (files with link count < 1 — deleted but still open) or `lsof | grep deleted`

**Fix:** Signal the process to reopen its log file (log rotation pattern) or use `copytruncate` in logrotate (truncates in place rather than rename-and-recreate, so the process keeps writing to the same inode which is now empty).

## 6. Fd Limits

- **Per-process soft limit:** `ulimit -n` (can be raised up to hard limit)
- **Per-process hard limit:** `ulimit -Hn` (requires root to raise)
- **System-wide ceiling:** `/proc/sys/fs/file-max`
- **Effective limits for a running process:** `/proc/[pid]/limits`
- **"Too many open files"** (EMFILE) traces directly back here — one of the most common root causes of connection pool exhaustion incidents

**Live debugging:** `ls -la /proc/[pid]/fd` shows every open fd as a symlink:
- Regular file: `/var/log/app.log`
- Socket: `socket:[12345]` (where 12345 is the socket inode)
- Pipe: `pipe:[67890]`
- epoll: `anon_inode:[eventpoll]`

---

## PART B: epoll / select / poll — I/O Multiplexing

## 7. The Problem: C10K

A single-threaded server handling many concurrent connections can't call blocking `read()` on each socket in sequence — it stalls on the first idle connection. Thread-per-connection doesn't scale past a few thousand (thread stack overhead, context-switch cost). The solution: non-blocking sockets (`O_NONBLOCK`) plus a mechanism to ask "which of these fds are actually ready right now."

**Terminology precision:** `select`/`poll`/`epoll` are **I/O multiplexing** (tell you when an fd is ready, you do the I/O yourself). True **async I/O** means the kernel performs the I/O and notifies you when done — that's POSIX AIO and the newer `io_uring`. The distinction matters in interviews.

## 8. select()

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

- You pass a bitmask (`fd_set`) of fds you care about
- Kernel **scans all N watched fds** on every call — O(n)
- **Modifies the bitmask in place** — you must rebuild it before the next call
- Hard cap at `FD_SETSIZE` (typically 1024 fds)
- Full bitmask copied between user and kernel space on every call

## 9. poll()

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- Array of `pollfd` structs instead of bitmask — removes the 1024 limit
- Still O(n) per call — kernel scans the entire array every time
- Still passes the whole fd list on every call

## 10. epoll() — The Scalable Solution

Three syscalls, persistent kernel-side state:

```c
int epoll_create1(int flags);                    // create epoll instance
int epoll_ctl(int epfd, int op, int fd,          // add/modify/remove watched fds
              struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events,  // wait for ready events
               int maxevents, int timeout);
```

**The algorithmic difference:** When a watched resource becomes ready (data arrives on a socket), the network stack's **wait-queue callback mechanism** pushes that fd directly onto epoll's internal ready list. `epoll_wait()` just returns what's already in the ready list. The kernel never scans anything.

- Cost proportional to number of **ready** fds, not **watched** fds
- Practically O(1) amortized — scales to 100K+ connections
- Persistent interest set: register once with `epoll_ctl()`, don't re-describe on every call
- No bitmask copying on every call

**Why this matters:** nginx, Redis (single-threaded), Node.js/libuv, Envoy — all built on epoll. A service built on blocking I/O with thread-per-connection hits a wall at a few thousand concurrent connections regardless of CPU. An epoll-based service scales into hundreds of thousands.

## 11. Level-Triggered vs Edge-Triggered

**Level-triggered (default):** As long as a fd remains ready (data still in buffer), `epoll_wait()` keeps reporting it on every call. Safe, easy to program — partial reads are fine.

**Edge-triggered (`EPOLLET`):** Reports **only once**, at the transition from not-ready → ready. Does not re-report just because data remains.

**The edge-triggered bug:** If you don't fully drain the fd (loop reading until `EAGAIN`) on each notification, data sitting beyond your partial read generates no new notification. The connection silently stalls. Only fails when enough data arrives in one burst to require multiple reads — intermittent, hard to reproduce. nginx uses edge-triggered and always drains in a loop.

## 12. io_uring (Know It Exists)

Newer (Linux 5.1, 2019+) true async I/O via shared submission/completion **ring buffers** between userspace and kernel:
- Zero syscalls per I/O operation (batch many ops in one syscall, or poll mode = zero syscalls)
- Works well for disk I/O (epoll historically didn't — `epoll` is for socket/pipe readiness, not file I/O)
- Direction Linux async I/O is heading
- epoll still dominates production systems today

---

## PART C: CFS Scheduler

## 13. What CFS Solves

CFS (Completely Fair Scheduler) governs `SCHED_NORMAL` (the default scheduling class). Goal: approximate an idealized processor where every runnable task gets equal, continuous CPU time — impossible with one core and N tasks, so CFS approximates it via virtual runtime.

## 14. Virtual Runtime (vruntime)

**The central concept.** Every task accumulates vruntime while running on CPU. Rate of accumulation is **inversely weighted by priority** — a higher-priority task's vruntime grows more slowly.

**The scheduler's rule:** Always run the runnable task with the **lowest vruntime.**

- A task that's been starved → low vruntime → gets picked next
- A task that's been running a lot → high vruntime → waits

## 15. Data Structure: Red-Black Tree

All runnable tasks on a CPU's run queue sit in a **red-black tree keyed by vruntime**:
- Insert/remove: O(log n)
- Pick next task: O(1) — kernel caches a pointer to the leftmost node
- Leftmost = lowest vruntime = next to run (by construction)

After a task runs and its vruntime increases, it's re-inserted further right, and a different task may now be leftmost.

## 16. Nice Values and Weights

`nice` values (−20 to +19) map to a fixed weight table. nice-0 = weight 1024. Each nice level differs by ~1.25×.

- vruntime accumulates **inversely** to weight: weight 2048 (nice -5) → vruntime grows at half the rate of weight 1024 (nice 0) → gets twice as much CPU time
- Gives proportional-share fairness, not blind round-robin
- A nice-difference of 1 always means a consistent ~10% CPU share difference regardless of how many tasks are competing

## 17. Scheduling Latency and Granularity

- **`sched_latency_ns`** (~24ms default): target period in which every runnable task gets scheduled at least once. Divided among runnable tasks to give each a timeslice.
- **`sched_min_granularity_ns`**: floor so timeslices don't get so small that context-switch overhead dominates (when many tasks compete)
- Timeslices are not fixed — they're continuously comparative. Preemption happens when a task's vruntime advantage exceeds a threshold (checked on scheduler tick or wakeup).

## 18. Load Balancing

Each CPU has its own run queue. The kernel periodically rebalances tasks across CPUs, weighing:
- Load imbalance (more tasks on one CPU than another)
- Cache affinity cost (moving a task loses its warm L1/L2 cache)
- NUMA topology (prefer memory-local CPUs)

## 19. CFS Bandwidth Control — The K8s CPU Limit Mechanism

Cgroup v2's `cpu.max` (or v1's `cpu.cfs_quota_us` / `cpu.cfs_period_us`):
- Cgroup gets `quota` μs of CPU per `period` μs
- Exceeding quota → tasks **throttled** (pulled off the run queue), not killed
- They resume at the start of the next period

**The bursty workload gotcha:**
- A request handler uses 80ms of CPU in the first 85ms of a 100ms period
- Hits quota → throttled for the remaining 15ms of the period
- Average CPU usage = 80% (under the limit)
- But that specific request stalled for 15ms unexpectedly
- P99 latency shows spikes; average CPU% looks fine

**Metric to watch:** `container_cpu_cfs_throttled_seconds_total` or `container_cpu_cfs_throttled_periods_total` from cAdvisor/Prometheus. Also readable directly: `/sys/fs/cgroup/.../cpu.stat` (`nr_throttled`, `throttled_usec`).

## 20. Scheduling Classes (Hierarchy)

Higher classes always preempt lower ones:

| Class | Policy | Description |
|-------|--------|-------------|
| `SCHED_DEADLINE` | Deadline | Newest; explicit deadline/period/runtime parameters |
| `SCHED_FIFO` | Real-time | Run until blocked or preempted by higher RT |
| `SCHED_RR` | Real-time | Round-robin within same priority |
| `SCHED_NORMAL` | CFS | Default for all ordinary processes |
| `SCHED_IDLE` | Idle | Only runs when nothing else wants CPU |

A `SCHED_FIFO` task will starve every CFS task if it never blocks.

## 21. EEVDF (Know It Exists)

Linux 6.6+ replaced CFS with **EEVDF** (Earliest Eligible Virtual Deadline First) as the default `SCHED_NORMAL` scheduler. Same underlying fairness concepts but adds explicit eligibility and deadline notions to reduce latency for interactive tasks (CFS could let latency-sensitive tasks wait too long behind CPU-bound ones). Most production kernels today (RHEL 8/9, Ubuntu LTS) still run CFS.

## 22. CFS Debugging Toolkit

```bash
chrt -p <pid>              # view scheduling policy and priority
nice / renice              # adjust nice value
/proc/[pid]/sched          # live vruntime, sum_exec_runtime, nr_switches
/proc/[pid]/status         # voluntary_ctxt_switches vs nonvoluntary_ctxt_switches
                           # rising nonvoluntary = CPU contention signal
taskset -p <pid>           # view/set CPU affinity
perf sched record          # record scheduling events
perf sched latency         # show per-task scheduling latency
```

---

## PART D: /proc Filesystem

## 23. What /proc Is

A **pseudo-filesystem** — no disk backing at all. Every read triggers a kernel function that formats live internal state as text on the fly. Another instance of "everything is a file." `/proc` is a filesystem-costumed view into running kernel data structures.

## 24. Per-Process: /proc/[pid]/

```
status       Human-readable summary: State, VmRSS, VmSize, Threads, UIDs
stat/statm   Machine-parseable (what ps/top read internally)
maps         VMA layout: address ranges, permissions, offsets, device/inode, path
smaps        Per-VMA breakdown with PSS, Referenced, Swap — more detailed than maps
fd/          Symlinks for every open fd:
               fd/3 -> /var/log/app.log
               fd/7 -> socket:[12345]
               fd/9 -> anon_inode:[eventpoll]
cwd          Symlink to current working directory
exe          Symlink to the executable binary
root         Symlink to the chroot root
environ      Environment variables, null-separated
cmdline      Exact argv, null-separated (args to process)
limits       Effective soft/hard rlimits (ulimit -n shows just one)
cgroup       Cgroup membership (v1: multiple lines; v2: single hierarchy path)
sched        Live scheduler state: vruntime, sum_exec_runtime, nr_voluntary_switches
task/[tid]/  Per-thread breakdown — a "process" is a thread group (tgid), each
             thread has its own tid subdirectory
oom_score    Current OOM badness score (read-only)
oom_score_adj  OOM adjustment (read-write if privileged)
```

## 25. Process State Codes — Memorize These

| Code | State | Notes |
|------|-------|-------|
| R | Running or runnable | On the run queue |
| S | Interruptible sleep | Waiting for an event; can be signal-interrupted |
| D | Uninterruptible sleep | Blocked in kernel I/O; cannot be interrupted by signals |
| Z | Zombie | Exited but not yet reaped by parent |
| T | Stopped | By SIGSTOP or job control |
| I | Idle kernel thread | (newer kernels) |

**D state is critical:** A process in D state cannot be killed by SIGKILL. It's blocked inside a kernel operation (usually I/O) that must complete or time out on its own. Classic example: NFS mount becomes unavailable → all processes touching it go to D state → load average climbs → `kill -9` does nothing → wait for NFS to recover or timeout.

## 26. System-Wide: /proc/

```
meminfo       Complete memory accounting (MemTotal, MemFree, MemAvailable,
              Buffers, Cached, SwapTotal/SwapFree, Active/Inactive, Dirty, etc.)
cpuinfo       Per-CPU model, flags, cache sizes, stepping
loadavg       "1.23 4.56 7.89 2/450 12345"
              1/5/15-min averages, runnable/total tasks, last PID scheduled
uptime        Seconds since boot, seconds idle
mounts        Currently mounted filesystems (same as /etc/mtab)
interrupts    Per-CPU IRQ counts — diagnose interrupt imbalance
softirqs      Per-CPU software interrupt counts
net/dev       Per-interface TX/RX packets/bytes/errors
net/tcp       TCP socket table
net/sockstat  Socket usage summary
sys/          The sysctl tunable tree (see below)
buddyinfo     Free memory blocks per order per zone (buddy allocator state)
slabinfo      Kernel slab allocator cache statistics
```

## 27. Load Average — The Most Misunderstood Metric

Load average is NOT CPU utilization. It is the running exponential average of:
- Tasks in **R** state (runnable, waiting for CPU) **PLUS**
- Tasks in **D** state (uninterruptible sleep, usually waiting for I/O)

**The critical implication:** High load average + low CPU% = tasks stuck in D state waiting on I/O, not CPU-bound work.

**Production pattern:** "Load average is 25 on an 8-core box but CPU shows 4%"
- Hypothesis: processes blocked on disk or NFS, not CPU starvation
- Confirm: `vmstat 1` — watch `b` column (number of processes in D state)
- Then: `ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /D/'` — see which processes and what kernel wait channel
- Also: `iostat -x 1` — check `await` time on storage devices
- Also: `dmesg | grep "server not responding"` — NFS timeout messages

## 28. /proc/sys — Identical to sysctl

`sysctl -w vm.swappiness=10` and `echo 10 > /proc/sys/vm/swappiness` are **identical operations** — both write to the same kernel variable. The dotted sysctl name maps directly to a slash-separated procfs path (dots become slashes).

`/etc/sysctl.conf` and `/etc/sysctl.d/*.conf` are just lists of key=value pairs replayed as these writes at boot (via `sysctl --system`).

**Operational implications:**
1. Change takes effect **immediately** — no daemon restart
2. A live change via `echo` or `sysctl -w` reverts on next reboot unless also written to a config file
3. Works inside containers without the `sysctl` binary (just `echo` to the path)
4. Some sysctls (`net.*`) are **network-namespace scoped** — the value inside a container's network namespace can differ from the host's value for the same-named parameter

---

## Retention Questions and Answers

**Q1: Two processes both hold fd 3 open on the same file, but writes from one aren't visible via the other's offset. What does that tell you about how each fd was opened?**

Each process called `open()` on the path independently. This gave each process its own `struct file` (open file description) with its own offset. Shared offset only happens when the same `struct file` is shared — which only occurs via `fork()` (child inherits parent's fd table, entries point at same `struct file`) or `dup()`/`dup2()`. The inode being the same is irrelevant to offset behavior — offset sharing is purely a function of whether the `struct file` itself is shared.

**Q2: Why can `rm`-ing a huge log file fail to free any disk space, and what's the one command that finds the culprit?**

`unlink()` removes the dentry and decrements the inode's link count, but the inode and data blocks aren't freed until both the link count AND the open-reference count hit zero. If a logging service still has the file open, the inode persists with its full disk allocation — invisible to `ls` (no directory entry) but `df` still shows it consumed. The command: **`lsof +L1`** — lists open files with link count below 1 (deleted but still open). Fix: signal the process to reopen its log file (releasing the old fd) or restart it.

**Q3: Explain precisely why epoll scales to 100K connections where select tops out — what's algorithmically different, not just "it's newer"?**

select/poll: kernel scans every watched fd on every call — O(n) in the number of watched fds, paid regardless of how many are actually active. Plus the entire interest set is copied user→kernel on every call.

epoll: interest set registered once and persists. When a watched resource becomes ready, a **callback on that resource's wait queue** pushes it directly onto epoll's ready list. `epoll_wait()` returns what's already in that list — never scans anything. Cost proportional to **ready** fds, not **watched** fds. With 100K open connections and 10 active at any instant: select pays O(100,000) per call; epoll pays O(10). That's the real difference: persistent kernel-side state + event-driven push notification vs. stateless per-call description + kernel-side poll.

**Q4: What's the practical difference between level-triggered and edge-triggered epoll, and what bug does getting this wrong cause?**

Level-triggered (default): keeps reporting a fd ready on every `epoll_wait()` call as long as the condition remains true (data still in buffer). Safe — partial reads work fine, fd stays reported.

Edge-triggered (`EPOLLET`): reports only once, at the not-ready → ready transition. Doesn't re-report just because data remains in the buffer.

The bug: in edge-triggered mode, if you don't fully drain the fd on each notification (loop reading until `EAGAIN`), data beyond your partial read sits undelivered with no new notification coming (no new edge transition). The connection silently stalls — only fails when data arrives faster than a single read drains it, making it intermittent and hard to reproduce. The mandatory pattern in edge-triggered code: always read/write in a loop until `EAGAIN`.

**Q5: A Kubernetes pod's CPU usage graph looks well under its limit on average, but requests are intermittently slow. What CFS mechanism explains this, and what metric would confirm it?**

CFS bandwidth throttling. The pod's CPU limit is enforced as `cpu.max` — a quota of CPU-microseconds per period (e.g. 100ms period). It's a bucketed budget over a rolling window. A bursty request handler can consume the entire quota in the first few milliseconds of a period, getting throttled for the rest, even though average usage looks fine. Requests that land during a throttled window stall.

Confirming metric: **`container_cpu_cfs_throttled_seconds_total`** or **`container_cpu_cfs_throttled_periods_total`** from cAdvisor/Prometheus. Also readable directly from `cpu.stat` in the cgroup directory. If throttled periods are climbing while average CPU% looks low, the throttle is the cause.

**Q6: A box shows load average of 25 on 8 cores but top shows 4% CPU usage. First hypothesis and how to confirm?**

Load average counts R (runnable) AND D (uninterruptible sleep) tasks. High load + low CPU = D-state processes stuck on I/O — not CPU starvation.

First hypothesis: processes blocked on disk I/O or network I/O (NFS, SAN, etc.)

Confirmation:
1. `vmstat 1` — watch the `b` column (number of D-state processes). If it's tracking the load average, confirmed.
2. `ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /D/'` — find which processes are in D state and what kernel function they're waiting in (`wchan` column)
3. `iostat -x 1` — check `await` (average wait time) on storage devices for high values
4. `dmesg | grep "server not responding"` — NFS-related messages
5. `netstat -s` or `ss` — check for network-level issues if NFS is involved

**Q7: What does it mean that /proc/sys and sysctl are "the same interface," and why does that matter operationally?**

There's no separate sysctl subsystem with its own storage. `sysctl -w vm.swappiness=10` and `echo 10 > /proc/sys/vm/swappiness` perform the literal identical write to the same kernel variable. The dotted naming maps directly to a slash-separated path (dots → slashes). `/etc/sysctl.conf` just replays these writes at boot.

Operationally this matters because:
1. Change is **immediate** — no service restart, no commit step. The write-and-done model is important during incident response.
2. A live change reverts on reboot unless also written to a persistent config file — these are separate acts. A common incident postmortem action item is "ensure sysctl change made during incident is persisted."
3. Works inside minimal containers without the `sysctl` binary — just `echo` to the path.
4. Some `net.*` sysctls are network-namespace scoped — the same parameter name can have different values inside a container vs. the host. A container-level override doesn't affect the host, which matters when debugging "why doesn't this tuning take effect on the node."
