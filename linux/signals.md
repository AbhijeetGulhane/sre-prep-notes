# Signals — SIGTERM vs SIGKILL vs SIGHUP vs SIGCHLD

**Date:** Week 3, Day 1

---

## The four signals

```
SIGTERM (15): catchable + ignorable — polite request to stop
              K8s sends this FIRST on pod termination, waits
              terminationGracePeriodSeconds (default 30s), then SIGKILL.
              App should catch it, finish in-flight work, exit clean.

SIGKILL (9):  NEITHER catchable NOR ignorable — kernel-only, no cleanup
              ever runs. OOM killer uses this. Influenced (not prevented)
              via /proc/<pid>/oom_score_adj (-1000 = protect, 1000 =
              preferred target). K8s sets this per pod by QoS class:
              BestEffort=1000, Guaranteed=-997.

SIGHUP (1):   catchable + ignorable — two meanings:
              original: controlling terminal closed
              modern convention: "reload config without restart"
              (nginx -s reload, sshd, systemd daemons). Also why log
              rotation works — SIGHUP makes a daemon re-open its log
              file path after logrotate renames the old one.

SIGCHLD (17): catchable + ignorable — sent to PARENT when a child's
              state changes (exit/stop/continue).
              Normal: handler calls waitpid(-1, WNOHANG) in a loop.
              Special: explicit SIG_IGN on SIGCHLD -> children
              auto-reaped by kernel, NEVER become zombies
              (this is POSIX-defined behavior).
```

## Comparison table

| Signal | Number | Catchable? | Ignorable? | Primary trigger |
|---|---|---|---|---|
| SIGHUP | 1 | Yes | Yes | Terminal close / config reload convention |
| SIGTERM | 15 | Yes | Yes | `kill`, `systemctl stop`, K8s pod termination |
| SIGKILL | 9 | No | No | `kill -9`, OOM killer, K8s grace period expiry |
| SIGCHLD | 17 | Yes | Yes (auto-reap side effect) | Child process state change |

## The two-step production shutdown pattern

```bash
kill -TERM <pid>     # polite request
sleep 30             # grace period — matches K8s/systemd default
kill -KILL <pid>     # unconditional, only if still running
```

## Why this matters for SRE work

If an app's PID 1 inside a container is a shell script that doesn't forward signals to its children, SIGTERM never reaches the actual application. The grace period expires, SIGKILL fires, and every shutdown is dirty — partial writes, dropped connections, no cleanup. Diagnosing "why does this container never shut down gracefully" often comes down to exactly this.

## Demo — ignoring SIGTERM, proving only SIGKILL works

```bash
python3 -c "
import signal, time
signal.signal(signal.SIGTERM, signal.SIG_IGN)
print('ignoring SIGTERM — only SIGKILL will stop me')
while True: time.sleep(1)
" &
PID=$!
kill -TERM $PID   # no effect — still running
ps -p $PID
kill -KILL $PID   # this always works
```
