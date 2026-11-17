# Troubleshooting Method — One Page

Use it the same way every time, in a live-debug round or a real incident. **Say each step out loud before you do it.** A good answer is a narrowing loop; a bad one is a list of commands you'd try.

```
SCOPE → CLARIFY → HYPOTHESES (ranked) → COMMAND + EXPECTED OUTPUT → SHORT-TERM FIX → LONG-TERM FIX
                         ↑______________ disproved? next hypothesis ______________|
```

---

## 1. Scope — what is broken, for whom, since when (≤ 1 min)
State the symptom in one sentence with numbers: *"p99 latency on /search went 200ms → 2s for all users in us-east since 14:05."*
- **Blast radius:** one host / one AZ / one service / everyone?
- **Impact first:** is the user hurt *right now*? If yes → mitigation runs before root cause.
- **Signal type:** errors, latency, saturation, or wrong data?

## 2. Clarifying questions — ask before touching anything
1. **What changed?** Deploy, config, flag, cert, infra, traffic pattern (Google SRE book: ~70% of outages follow a change).
2. **When exactly** did it start, and is it constant, growing, or intermittent?
3. **Who/what is affected** vs. not? (One host fine, one bad → host-local. All bad → shared dependency.)
4. **What do the dashboards / alerts already show?** Don't re-derive what's on a graph.
5. **What access do I have**, and is there a rollback path?

## 3. Hypotheses — rank by likelihood × cheapness to test
Write 2–4, top one first, and name what would **disprove** each:
> *"Most likely: the 14:00 deploy. Disproved if the canary host on the old version is also slow."*

Walk **outside-in**: client → DNS → LB → app → dependency (DB/cache/queue) → host (CPU/mem/disk/net) → kernel. On a host, use **USE** (Utilization, Saturation, Errors) for every resource; for a service, use **RED** (Rate, Errors, Duration).

## 4. Command + expected output — predict first, then run
**Rule:** say the output you expect *before* running it. A result that matches nothing you predicted means your model is wrong — stop and rescope.

| Resource | Command | Healthy | Points to a problem |
|---|---|---|---|
| Load / overview | `uptime`; `top -o %CPU` | load avg ≤ cores | load ≫ cores; high `wa` = I/O; high `st` = noisy neighbour |
| CPU saturation | `vmstat 1 5` | `r` ≤ cores, `id` > 20 | `r` ≫ cores (run-queue backlog); `us` vs `sy` split says app vs kernel |
| Memory | `free -m`; `vmstat 1` | `available` large; `si/so` = 0 | `si/so` > 0 (swapping); `dmesg -T \| grep -i oom` shows kills |
| Disk space / inodes | `df -h`; `df -i` | < 85% | 100% → write failures; inodes 100% with free space = many small files |
| Disk I/O | `iostat -xz 1 3` | `%util` < 70, `await` low ms | `%util` ~100, `await` 10s–100s of ms → device saturated |
| Process | `ps -o pid,stat,etime,rss,cmd -p <pid>`; `ls /proc/<pid>/fd \| wc -l` | state `S`/`R`; fds ≪ `ulimit -n` | `D` = stuck in I/O; `Z` piling up; fds near limit → EMFILE |
| Syscalls (stuck proc) | `strace -fp <pid> -tt -T` | calls returning quickly | blocking on `futex` (lock), `connect`/`read` (dependency), `ENOSPC` |
| Network sockets | `ss -s`; `ss -tanp state time-wait \| wc -l` | stable counts | thousands of TIME_WAIT/CLOSE_WAIT → port exhaustion / leak (CLOSE_WAIT = app not closing) |
| Retransmits | `nstat -az \| grep -i retrans`; `netstat -s` | retrans ~flat | climbing → packet loss / congestion on the path |
| DNS / reachability | `dig +short svc.internal`; `curl -sv -o /dev/null -w '%{time_connect} %{time_total}\n' URL` | resolves; connect ms | NXDOMAIN/SERVFAIL; connect fast but total slow → server-side, not network |
| Logs / kernel | `journalctl -u <svc> --since "-30m" -p err`; `dmesg -T \| tail` | quiet | first error timestamp ≈ incident start → strong lead |
| K8s | `kubectl get pods -o wide`; `kubectl describe pod`; `kubectl top pod` | Running, 0 restarts | CrashLoopBackOff, OOMKilled (exit 137), Pending (resources/taints), throttling |

After each result say: **"That confirms / rules out H1, so next I check ___."**

## 5. Short-term fix — stop the bleeding (reversible, fast)
Pick the **smallest, most reversible** action that restores users; it does not need to be the root cause.
- Roll back the change · flip the flag off · drain/cordon the bad host or AZ · scale out · restart (only after capturing evidence: logs, `jstack`/core, `/proc` snapshot) · free disk / rotate logs · shed load / rate-limit.
- **Verify** with the same metric that defined the scope ("p99 back to 200ms"). Communicate status.

## 6. Long-term fix — make it not happen again
- **Root cause:** keep asking "why" until you reach a process or design gap, not a person.
- **Fix the class, not the instance:** capacity limits, timeouts + retries with backoff, circuit breakers, log rotation, fd limits, connection pooling.
- **Detection:** alert on the symptom (SLO burn rate) earlier than users noticed this time.
- **Prevention:** canary/progressive rollout, pre-deploy checks, runbook entry, automation of the manual fix.
- **Blameless postmortem** with owners + dates on each action item.

---

### Interview talk-track (30-second version)
*"First I'll scope impact — who, what, since when. I'll ask what changed. My top hypothesis is X because Y; I'd disprove it by running Z, where I expect to see A. [result] That rules it out, so next… Once I find it, I mitigate first with a rollback, verify on the same metric, then fix the root cause and add an alert so we catch it sooner."*

### Failure modes to avoid
- Running commands without stating what you expect → reads as guessing.
- Jumping to root cause while users are still down → mitigate first.
- Restarting before capturing evidence → the cause is destroyed.
- Naming a tool without naming the mechanism (say "run-queue saturation," not just "vmstat shows r=40").
