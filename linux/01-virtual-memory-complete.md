# Virtual Memory — Complete Study Notes
## Kernel Page Management, Page Faults, Swap, OOM Killer

---

## 1. Why Virtual Memory Exists

Every process on Linux believes it owns the entire address space — 0 to 2^48-1 on x86-64 with 4-level paging, or 2^57-1 with 5-level paging on newer CPUs. This is a deliberate illusion maintained by the kernel for three reasons:

- **Isolation** — one process cannot read or corrupt another's memory. Each process has its own page tables mapping virtual addresses to physical frames. Even if two processes happen to use the same virtual address, they map to different physical frames.
- **Overcommit** — the system can run more total virtual memory than physical RAM exists, because most virtual memory is never actually touched. A process can `mmap()` 10GB and allocate almost no physical RAM if it only reads a small portion.
- **Simplicity** — every program links and runs as if it has a flat, private address space starting fresh. The compiler, linker, and runtime don't need to coordinate with other processes about where they put things.

### Process Address Space Layout (x86-64, low to high)

```
0x0000000000000000  ← NULL (unmapped, catches null pointer dereferences)
[text segment]       ← executable code, read-only + executable
[data segment]       ← initialized globals and statics
[BSS]                ← zero-initialized globals (size info only in binary)
[heap]               ← grows upward via brk()/mmap()
[memory-mapped]      ← shared libs, large malloc()s via mmap(), file mappings
[stack]              ← grows downward, one per thread
0x00007fffffffffff  ← top of user space (128TB on x86-64)
[kernel space]       ← upper portion, every process's page tables mapped here
                       (post-Meltdown/KPTI: mostly separated)
```

**Important:** `malloc()` for allocations above ~128KB (MMAP_THRESHOLD) uses `mmap()` for anonymous memory, not `brk()`. Only small allocations go through the heap's `brk()` interface.

**Canonical address form:** On x86-64, bits 63–48 must all match bit 47. A non-canonical address faults immediately in hardware before the page tables are even consulted.

**KPTI (Kernel Page Table Isolation):** Post-Meltdown (2018), the kernel maintains almost entirely separate page tables for user and kernel space, trading syscall/interrupt overhead for security. **PCID** (Process Context Identifiers) became critical to offset this — it tags TLB entries by address-space ID so context switches don't need full TLB flushes.

---

## 2. Address Translation — Page Tables, TLB, MMU

### The MMU

The Memory Management Unit (part of the CPU hardware) translates every virtual address into a physical address on every single memory access — not just on faults. It walks the multi-level page table on a TLB miss.

### 4-Level Page Tables (x86-64)

A 48-bit virtual address breaks into five fields:

```
Bits 47-39 (9 bits) → PML4 index   (512 entries, each 8 bytes → 4KB table)
Bits 38-30 (9 bits) → PDPT index
Bits 29-21 (9 bits) → PD index
Bits 20-12 (9 bits) → PT index
Bits 11-0  (12 bits)→ offset within the 4KB page
```

- **CR3 register** holds the physical address of the current process's PML4 table (reloaded on every context switch).
- **Why multi-level?** A flat table for 48-bit space would need 2^36 entries — impossible. Multi-level means branches for unused regions simply don't exist, saving enormous amounts of memory for sparse address spaces.
- Each table is exactly 4KB (512 × 8 bytes) — fits in one page, a useful recursive property.

### Page Table Entry (PTE) bits

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | Present (P) | Page is in physical RAM |
| 1 | Read/Write (R/W) | Page is writable |
| 2 | User/Supervisor (U/S) | Accessible from user mode |
| 5 | Accessed (A) | Hardware sets on any read/write |
| 6 | Dirty (D) | Hardware sets on any write |
| 7 | Page Size (PS) | Points to a huge page (upper-level entries) |
| 63 | NX (No-Execute) | Code execution forbidden (W^X security) |

### TLB (Translation Lookaside Buffer)

Caches recent virtual→physical translations. A TLB hit skips the 4-level table walk entirely. A miss triggers the hardware page table walker (x86 does this in hardware).

- Full TLB flush on every context switch would be expensive.
- **PCID** tags each TLB entry with an address-space ID — the CPU ignores entries tagged for a different process instead of flushing everything.
- Post-KPTI: two CR3 loads per syscall (user table ↔ kernel table), making PCID even more critical.

---

## 3. Physical Memory Management

### Memory Zones

| Zone | Range | Purpose |
|------|-------|---------|
| ZONE_DMA | < 16MB | Legacy ISA devices |
| ZONE_DMA32 | < 4GB | 32-bit DMA capable devices |
| ZONE_NORMAL | Everything else | Main pool, directly mapped |
| ZONE_HIGHMEM | (32-bit only) | Irrelevant on 64-bit |

On NUMA systems, the whole structure repeats per node (`pg_data_t`). The allocator tries to keep process memory on the same NUMA node as the CPU running it.

### Buddy Allocator

Manages physical pages in power-of-2 sized blocks called **orders** (order 0 = 4KB, order 10 = 4MB). When a block is freed, the allocator checks if its adjacent "buddy" is also free and merges them into the next order up.

- **`/proc/buddyinfo`** shows free block count per order per zone — useful when huge-page allocation fails despite `free -h` looking healthy (external fragmentation: plenty of free memory, but no contiguous 2MB run).

### Slab/SLUB Allocator

Sits on top of buddy allocator for small, frequently-allocated kernel objects (`task_struct`, `inode`, `dentry`, network socket buffers). Pre-carves buddy-allocated pages into caches of fixed-size objects.

- **`/proc/slabinfo`** and **`slabtop`** show kernel internal memory — useful when total memory used doesn't match what processes report.

### Transparent Huge Pages (THP)

Instead of 4KB pages, the kernel can use 2MB pages, drastically reducing TLB pressure. `khugepaged` runs in the background promoting regions. Controlled via `/sys/kernel/mm/transparent_hugepage/enabled`:
- `always` — aggressively promote (risky: compaction stalls)
- `madvise` — only for regions explicitly requesting it
- `never` — disabled

**The THP gotcha:** Under memory pressure, allocating a 2MB contiguous block requires **compaction** — moving pages around — which can cause multi-hundred-millisecond latency stalls. This is why Redis, MongoDB, and most databases ship guidance to set THP to `madvise` or `never`. If you see periodic ~200ms freezes with no obvious CPU/disk cause, THP compaction is a prime suspect.

---

## 4. The Address Space Model: mm_struct and VMAs

- **`mm_struct`:** per-process descriptor, hangs off `task_struct`. Holds the page table root pointer (`pgd`), aggregate RSS/VSZ counters, and the VMA collection.
- **VMA (`vm_area_struct`):** one per contiguous virtual range sharing permissions and backing. One VMA for the heap, one per shared library, one for the stack, one per `mmap()`'d file.
- Modern kernels (6.1+) store VMAs in a **maple tree** (replaced the older red-black tree) for faster range lookups and better RCU-friendly concurrent access.
- The page fault handler consults VMAs to answer "is this address supposed to be mapped, and what kind of memory is it?"

---

## 5. Page Faults

### The Mechanism

A page fault is a CPU exception (`#PF` on x86) raised when the MMU can't complete a translation. The CPU:
1. Pushes an error code on the kernel stack
2. Stores the faulting address in **CR2**
3. Traps to the kernel fault handler: `do_page_fault` → `handle_mm_fault`

The handler looks up the VMA covering the faulting address, then decides what to do.

### Three Categories

**Minor (soft) fault — no disk I/O:**
- First touch of a zero-fill anonymous page (freshly `mmap()`'d or `malloc()`'d memory — kernel allocates a zeroed physical frame and maps it)
- Write to a copy-on-write page that's already resident (kernel copies it)
- File-backed page already in the page cache from another process's earlier read (just needs a PTE wired up)

**Major (hard) fault — disk I/O required:**
- Reading a file-backed page not yet in the page cache (must read from disk)
- Reading a page back from swap (must swap-in)
- Monitored via `vmstat`'s `si`/`so` columns and `sar -B`'s `majflt/s`

**Invalid fault — SIGSEGV:**
- Address not covered by any VMA at all
- Access type violates VMA permissions (writing to read-only text segment, executing from non-executable stack)
- Results in SIGSEGV; if no handler, process dies

### Copy-on-Write (COW) — Deep Dive

At `fork()`:
1. Kernel copies the parent's page tables (not the actual pages)
2. For all private-mapping pages: marks PTEs **read-only** in both parent and child
3. Increments each physical frame's reference count

When either process writes to a shared page:
1. Hardware sees write to read-only page → raises a protection fault
2. Kernel fault handler identifies this as a COW page (refcount > 1)
3. Allocates a new physical frame via buddy allocator
4. Copies the page content into the new frame
5. Updates the writing process's PTE to point at new frame, now writable
6. Decrements the original frame's refcount
7. Faulting instruction retries and succeeds

This is a **minor** fault (no disk I/O). The key insight: `fork()` is cheap even for gigabyte processes because you never copy data that's never written, and `fork()+exec()` (the most common pattern) avoids almost all copying since `exec()` replaces the address space immediately.

### Demand Paging

`malloc()`, `mmap()`, and `brk()` don't allocate physical memory — they only create or extend a VMA. Physical allocation is deferred to the first-touch minor fault. This is why:
- `mmap()`-ing a 10GB file shows almost no RSS increase until pages are read
- A process can `malloc()` 1GB and only use a fraction before being killed — the allocator said "yes" but the pages were never faulted in

---

## 6. Swap and Page Reclaim

### The LRU Approximation

True LRU would require updating a list on every memory access — far too expensive. Linux approximates with **four inactive/active lists**:
- `active_anon`, `inactive_anon` (for anonymous/swappable memory)
- `active_file`, `inactive_file` (for file-backed/page cache memory)
- Plus `unevictable` (mlocked pages, etc.)

The hardware **Accessed bit** (set automatically on any reference, zero software cost) drives promotion/demotion:
- Page in inactive list + A bit set → promoted to active (bit cleared)
- Page in active list + A bit not set → demoted to inactive
- Reclaim always targets the **tail of the inactive** lists

View the current split in `/proc/meminfo`: `Active`, `Inactive`, `Active(anon)`, `Inactive(file)`, etc.

### kswapd and Watermarks

Three watermarks per zone: **min**, **low**, **high**.

```
Anchored by: /proc/sys/vm/min_free_kbytes
low  ≈ 1.25 × min
high ≈ 1.50 × min
(watermark_scale_factor gives finer control on newer kernels)
```

**kswapd** (per-node kernel thread) wakes when free memory drops below `low`, reclaims toward `high`:
- Walks inactive lists
- Unmaps pages from every process via **rmap** (reverse mapping — allows finding every PTE pointing at a given physical frame)
- Writes dirty pages back
- Frees clean pages

### Direct Reclaim — The Latency Spike Source

If an allocation request arrives and free memory is **already below min** before kswapd catches up:
- The **allocating process itself** is forced to do reclaim work synchronously
- It cannot get its allocation until enough memory has been freed
- This shows up as a latency spike on that request: a request that should take 2ms takes 300ms
- CPU looks idle, disk I/O looks modest — very hard to attribute without knowing this mechanism

**How to confirm:** Check `/proc/vmstat` for `allocstall_*` and `pgscan_direct` counters correlating with latency spikes.

### Swappiness

`vm.swappiness` (0–100, default 60): how aggressively to prefer swapping anonymous pages over reclaiming page cache.
- Lower = prefer page cache eviction first, swap less
- Common tuning for databases: `vm.swappiness=1` or `10`
- Not "disable swap" — just preference

### Dirty Page Writeback

A dirty page (hardware D bit set on any write) can't be dropped during reclaim — must be written back first. Tuned via:
- `vm.dirty_ratio` — max percentage of total memory that can be dirty before processes start doing synchronous writeback
- `vm.dirty_background_ratio` — percentage at which background flusher threads start writing back

`buff/cache` in `free -h` is **reclaimable** — not "used" in any concerning sense.

### Thrashing

When the working set genuinely exceeds available RAM: continuous reclaim → more time paging than working → throughput and latency collapse. The graceful degradation swap is supposed to provide only works if overcommit is reasonable.

---

## 7. The OOM Killer

### Invocation

`out_of_memory()` in `mm/oom_kill.c` is called when:
1. An allocation fails
2. Reclaim was attempted and failed
3. There is genuinely no way to get memory through eviction alone

### Global vs Cgroup-Scoped OOM — The Critical Distinction

**Global OOM:** Whole machine is out of memory. Evaluates every eligible process system-wide.

**Cgroup-scoped OOM (cgroup v2):** Fires when a cgroup exceeds its `memory.max` limit — **even if the rest of the system has plenty of free RAM.** This is the one that fires for Kubernetes containers. A pod's memory limit is `memory.max`; when the container's usage would exceed it, only processes within that cgroup are eligible victims. The node is untouched.

### Badness Scoring

Every eligible process gets a score 0–1000 computed primarily from:
- Percentage of available memory the process uses (RSS + swap + page table overhead)
- Adjustable via `/proc/[pid]/oom_score_adj` (range −1000 to +1000)
  - −1000 = "never kill this" (for critical daemons)
  - +1000 = "kill this first"

Read live score: `/proc/[pid]/oom_score`

### What Happens on Kill

- Victim receives **SIGKILL** — no graceful cleanup, no buffer flushing, no connection draining
- Kernel logs to dmesg: `"Out of memory: Killed process X (name), total-vm:..., anon-rss:..., oom_score:..."`
- Find past events: `dmesg -T | grep -i "killed process"`

### Kubernetes QoS → oom_score_adj Mapping

| QoS Class | How set | oom_score_adj | Kill priority |
|-----------|---------|----------------|---------------|
| Guaranteed | requests == limits for all resources | ~-997 | Last to die |
| Burstable | requests < limits, or partial | Scaled by usage | Middle |
| BestEffort | No requests or limits set | 1000 | First to die |

### Kubelet Node-Pressure Eviction vs Cgroup OOM Kill

**Kubelet node-pressure eviction:**
- Userspace, kubelet-driven, proactive
- Kubelet monitors signals (`memory.available`, `nodefs.available`, etc.)
- Evicts whole pods before the node actually runs out, by QoS order (BestEffort first)
- Goes through normal SIGTERM → grace period → SIGKILL flow
- Pod status: `phase: Failed`, `reason: Evicted`

**Cgroup-scoped OOM kill:**
- Kernel-driven, fires when a container's cgroup hits `memory.max`
- Immediate unconditional SIGKILL — no grace period, no preStop
- Only that container dies; pod typically stays Running, container restarts per restartPolicy
- Container status: `lastState.terminated.reason: OOMKilled`, `exitCode: 137`

**The key asymmetry (exam-critical):**
- Exceeding **memory limit** → OOMKilled (kernel, immediate, no soft mechanism)
- Exceeding **CPU limit** → throttled via CFS bandwidth control (pulled off run queue, resumes next period)
- CPU is a rate (time passes and quota refills); memory is a stock (you hold it continuously, no way to "pause" usage)

---

## 8. SRE Debugging Toolkit

```bash
# System-wide memory overview
free -h                    # buff/cache is reclaimable, not "used"
                           # trust MemAvailable over MemFree

/proc/meminfo              # MemTotal, MemFree, MemAvailable
                           # Buffers, Cached, SwapTotal/SwapFree
                           # Active/Inactive breakdown, Dirty, Writeback
                           # Active(anon), Inactive(anon), Active(file), Inactive(file)

# Per-process memory
/proc/[pid]/status         # VmRSS (resident), VmSize (virtual), VmSwap, VmHWM (peak)
/proc/[pid]/smaps          # Per-VMA breakdown with PSS (Proportional Set Size)
                           # PSS correctly accounts for shared memory
                           # (shared lib used by 10 processes → 1/10 charged to each)
smem                       # PSS-aware reporting, avoids double-counting shared memory

# Paging and swap activity
vmstat 1                   # si/so = swap in/out per second
                           # procs "b" = processes blocked in D state (I/O wait)
sar -B                     # paging activity over time (pgpgin, pgpgout, majflt/s)
sar -W                     # swap activity over time

# OOM debugging
dmesg -T | grep -i "killed process"  # past OOM events with full context
dmesg -T | grep -i oom               # broader OOM-related messages

# cgroup v2 (what governs container memory in K8s)
cat /sys/fs/cgroup/.../memory.current   # current usage
cat /sys/fs/cgroup/.../memory.max       # the limit (pod's memory limit)
cat /sys/fs/cgroup/.../memory.high      # soft throttle threshold
cat /sys/fs/cgroup/.../memory.events    # includes oom_kill counter

# Huge pages
/proc/buddyinfo             # free block count per order per zone
                            # diagnose huge-page allocation failures
/sys/kernel/mm/transparent_hugepage/enabled  # THP setting
```

**The single most important habit:** Use **MemAvailable** not **MemFree** for real memory pressure assessment. A healthy system with lots of page cache shows low MemFree and high Cached — that's the kernel doing its job. MemAvailable already accounts for how much cache could be reclaimed.

---

## 9. Retention Questions and Answers

**Q1: A container's RSS looks modest, but it just got OOMKilled — how, and what would you check first?**

The kill came from the cgroup-scoped OOM killer, not the global one. A container's memory limit is enforced as `memory.max` on its cgroup — the cgroup is killed when its usage crosses that limit, regardless of whether the node has plenty of free RAM. "Modest RSS" appears because you're looking at a snapshot *after* the kill and restart, when usage is naturally low again.

First checks:
1. `memory.events` on the container's cgroup — confirms `oom_kill` counter incremented
2. Pull a memory usage *graph* (not a snapshot) across the window before the kill — look for transient spikes
3. Remember `memory.current` counts more than plain RSS — it includes some page cache and slab memory charged to the cgroup, so RSS-only checks can understate real pressure
4. Check if the limit itself is set too tight for legitimate peak usage

**Q2: Walk through, step by step, what happens when a forked child writes to a page it shares with its parent.**

1. At `fork()`, kernel copies parent's page tables but not the physical frames
2. All private-mapping PTEs are marked **read-only** in both parent and child; frame refcounts incremented
3. Child writes to one of these pages → hardware raises a protection fault (write to read-only PTE)
4. Kernel fault handler sees this is a COW page (refcount > 1) — handles as a minor fault (no disk I/O)
5. Allocates a new physical frame via buddy allocator
6. Copies the page content into the new frame
7. Updates the child's PTE: now points at new frame, marked writable
8. Decrements the original frame's refcount
9. Faulting write instruction retries and succeeds
10. Parent's PTE is completely untouched — still points at the original frame

**Q3: Why doesn't Linux use true LRU for page reclaim, and what does it use instead?**

True LRU requires updating list position on every memory access — millions of times per second system-wide. The bookkeeping overhead would dominate. Instead, Linux uses a **two-list clock approximation** with active/inactive lists for anonymous and file-backed pages (4 lists total). The hardware Accessed bit (set automatically by the CPU on any reference, zero software cost) drives promotion/demotion. Reclaim always targets the tail of the inactive list. A page gets a "second chance" if its Accessed bit is set when scanned — promoted back to active. Genuinely cold pages have their bit clear and get reclaimed. Coarse, batch-based, efficient — captures the spirit of LRU without per-access overhead.

**Q4: A service's p99 latency spikes intermittently with no CPU or disk I/O signal. What's your hypothesis, and how do you confirm?**

**Primary hypothesis: direct reclaim stalls.** If free memory drops below the `min` watermark before kswapd catches up, the allocating process is forced to do synchronous reclaim. This shows up as a latency spike with almost no CPU% movement (it's lock/unmap work) and no classic disk I/O signal.

Competing hypotheses to rule out:
- **THP compaction stalls** (if THP is set to `always`)
- **GC pauses** (JVM/Go — look identical externally, check runtime pause logs)
- **CFS throttling** (check `container_cpu_cfs_throttled_seconds_total`)

To confirm direct reclaim: `/proc/vmstat` — look for `allocstall_*` and `pgscan_direct`/`pgsteal_direct` counters spiking with the latency. Watch `MemAvailable` approaching the min watermark. Use `perf` or eBPF to trace time in the kernel's page allocation slow path.

**Q5: Why does exceeding a CPU limit throttle a container, while exceeding a memory limit kills it?**

CPU is a **rate** — a flow of time. A container that exceeds its quota just doesn't get scheduled for the rest of its period; time passes and the quota refills. Throttling is a complete, fully reversible mechanism — no data is harmed.

Memory is a **stock** — an amount actively held at a point in time. There's no "pause usage for a while" — the process holds those bytes continuously. If a cgroup is over `memory.max`, reclaim is attempted first. But if the memory is genuinely live (actively referenced, not reclaimable cache), the only way to bring usage under the limit is to force a process to surrender its pages by killing it. No soft mechanism exists.

**Q6: What's the practical difference between kubelet node-pressure eviction and a cgroup-scoped OOM kill — how would you tell which one happened?**

**Node-pressure eviction:** kubelet-driven, userspace, proactive. Kubelet monitors node pressure signals and evicts whole pods before the node runs out, going through SIGTERM → grace period → SIGKILL. Pod status: `phase: Failed`, `reason: Evicted`.

**Cgroup-scoped OOM kill:** kernel-driven, fires when a container's own `memory.max` is exceeded. Immediate unconditional SIGKILL, no grace period. Only that container dies; pod stays `Running`, container restarts per `restartPolicy`. Container status: `lastState.terminated.reason: OOMKilled`, `exitCode: 137`.

One-line diagnostic: `Evicted` = whole pod moved/died, node-triggered. `OOMKilled` = single container restarted in place, cgroup-triggered.
