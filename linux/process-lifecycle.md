# Process Lifecycle — fork, exec, wait, zombie, orphan

**Date:** Week 3, Day 1
**Commands used:** `ps aux --forest`, `strace ls`, `cat /proc/$$/status`

---

## The five concepts

```
fork:    1 process -> 2 identical, child gets return value 0
         Copy-on-Write page tables under the hood — NOT a real memory
         copy. clone() is the actual syscall; fork() is a thin wrapper
         around it with a specific set of flags.

exec:    same PID, code replaced entirely (no new process created)
         Kernel loads a tiny dynamic linker (ld.so) first. THAT linker —
         not the kernel — loads libc/libselinux/etc. This is what
         strace ls shows happening after the initial execve() line.

wait:    parent collects child's exit status
         Event-driven via SIGCHLD, not polling. Kernel notifies the
         parent the instant a child's state changes (exit/stop/continue).

zombie:  child is dead but NOT yet reaped by parent -> STAT Z, <defunct>
         Only a PID + exit status remain — everything else (memory, fds)
         was already freed by the kernel. Costs a process-table slot.
         Enough zombies -> fork() fails system-wide with EAGAIN once
         pid_max / ulimit -u is hit.

orphan:  parent dies BEFORE child -> child instantly reparented to PID 1
         (or to a "subreaper" if one registered via
         prctl(PR_SET_CHILD_SUBREAPER)). Inside a container, the app is
         often PID 1 of its own namespace — if it forks subprocesses
         with no real init present, orphans can't be reaped properly.
         This is why tini / dumb-init / Docker --init exist.
```

## Diagnostic commands worth keeping

```bash
ps aux --forest                       # fork tree, visually
strace ls                             # watch execve() + library loading live
cat /proc/$$/status                   # State + PPid for current shell
cat /proc/sys/kernel/pid_max          # process table ceiling
ulimit -u                             # per-user process limit
ps -eo stat | grep -c Z               # count current zombies (real diagnostic)
```

## Why this matters for SRE work

A buggy parent process that forks constantly and never calls `wait()` will accumulate zombies indefinitely. They cost almost nothing individually, but enough of them exhausts the process table — at that point `fork()` itself starts failing for *everything* on the machine, not just the buggy service. Real incident signature: a service "can't handle load," CPU/memory look fine, and `ps -eo stat | grep -c Z` returns a number in the thousands.

## Demo scripts (proved both states live)

**Zombie:**
```python
import os, time, sys
pid = os.fork()
if pid == 0:
    sys.exit(0)
else:
    time.sleep(3)  # deliberately NOT calling wait() yet
    os.system(f"ps -o pid,ppid,stat,cmd -p {pid}")  # shows STAT Z
    os.wait()  # reaps it
```

**Orphan:**
```python
import os, time
pid = os.fork()
if pid == 0:
    time.sleep(2)
    print(f"PPid is now: {os.getppid()}")  # prints 1 — reparented
else:
    pass  # parent exits immediately, orphaning the child
```
