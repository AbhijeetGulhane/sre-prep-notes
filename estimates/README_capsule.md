# capsule

A minimal container runtime built from Linux primitives — namespaces,
cgroups v2, `pivot_root`, and a PID-1 init that reaps zombies and forwards
signals. Written to understand what Docker and the kubelet do underneath,
not to replace them.

Built and verified on WSL2 Ubuntu 22.04 (kernel 6.1.x), Go 1.26, cgroup v2.

## What it does

- **Mount + UTS namespace** — own hostname; own root filesystem via
  `pivot_root` into an Alpine rootfs, old root detached cleanly.
- **PID namespace** — the workload is PID 1; a fresh `/proc` shows only
  container processes.
- **PID-1 init (`--init`)** — reaps orphaned zombies and forwards signals,
  the way tini does.
- **cgroups v2** — CPU (`--cpus`) and memory (`--memory-mb`) limits.
- **User namespace (`--rootless`)** — run without sudo; container-root maps
  to your unprivileged uid.
- **Network namespace** — isolated net stack (loopback only).

## Build

    go build -o capsule .
    gcc -static -O2 -o orphan_maker c/orphan_maker.c   # for the zombie demo
    ./scripts/fetch-rootfs.sh ./rootfs                 # Alpine minirootfs
    cp orphan_maker ./rootfs/

## Run

    sudo ./capsule run /bin/sh
    sudo ./capsule run --hostname box --cpus 0.5 --memory-mb 64 /bin/sh
    ./capsule run --rootless /bin/sh                   # no sudo

## Verify each layer

Namespace isolation:

    sudo ./capsule run --hostname box /bin/sh
    # inside:  hostname -> box;  cat /etc/os-release -> Alpine;  echo $$ -> 1
    # host stays clean:  mount | grep put_old  (empty),  hostname unchanged

PID-1 reaping (the zombie demo):

    sudo ./capsule run /orphan_maker           # workload is PID 1, no reaper
    #   -> zombies visible: 5
    sudo ./capsule run --init /orphan_maker    # capsule is PID 1, reaps them
    #   -> zombies visible: 0

CPU limit (watch `top` in another terminal — pegs near 50% of one core):

    sudo ./capsule run --cpus 0.5 /bin/sh -c 'while :; do :; done'
    cat /sys/fs/cgroup/capsule/*/cpu.max       # -> 50000 100000

Memory limit (capsule prints memory.events on exit):

    sudo ./capsule run --memory-mb 64 /bin/sh -c 'sleep 1'
    cat /sys/fs/cgroup/capsule/*/memory.max    # -> 67108864

Rootless (no sudo):

    ./capsule run --rootless /bin/sh
    # inside:  id -> uid=0(root);  cat /proc/self/uid_map -> "0  1000  1"

Network isolation:

    sudo ./capsule run /bin/sh
    # inside:  ip link -> only lo (UP,LOWER_UP);  ping -c1 127.0.0.1 works
    # host:    ip link -> your real interfaces (different netns)

Raw syscalls (C companion — watch the kernel calls fire in order):

    gcc -Wall -O2 -o mini_container c/mini_container.c
    sudo ./mini_container ./rootfs /bin/sh -c 'hostname; echo $$'
    # -> capsule-c, 1  (same isolation as the Go runtime)

    sudo strace -f -e trace=clone,pivot_root,mount,umount2,execve \
        ./mini_container ./rootfs /bin/sh -c 'echo hi'
    # clone(..., CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWPID|SIGCHLD)
    # mount(MS_REC|MS_PRIVATE) -> bind rootfs -> pivot_root -> mount proc -> execve

## WSL2 caveats (documented, not fought)

- **Memory OOM enforcement may not fire.** The `memory.max` limit is written
  and real, but WSL2's kernel often won't OOM-kill on breach. capsule prints
  `memory.events` on exit so the counters remain observable regardless.
- **cgroup limits are ignored under `--rootless`** — unprivileged users can't
  write the cgroup tree. capsule warns when the flags are combined.
- **Unprivileged user namespaces may be restricted** by kernel policy on some
  hosts. The code is correct; the policy gates it. Check with
  `sysctl kernel.apparmor_restrict_unprivileged_userns` if `--rootless` fails.
- **Controller delegation** may need a delegated scope on systemd-managed
  WSL2:  `sudo systemd-run --scope -p Delegate=yes ./capsule run --cpus 0.5 …`

## Deliberately not included

- No image format / registry / layers (no overlayfs) — bring your own rootfs.
- No working container network — loopback only; no veth pair, bridge, or NAT.
- No seccomp / AppArmor / capability dropping.
- No terminal job-control handoff (`tcsetpgrp`).
- No OCI runtime spec or CRI compatibility.

These are the honest boundaries, each a natural next phase rather than a bug.

## Reading

- LWN, "Namespaces in operation" (Kerrisk, 7 parts) — canonical. Free.
- man7: namespaces(7), pid_namespaces(7), user_namespaces(7), cgroups(7),
  clone(2), pivot_root(2). Free.
- kernel.org: Documentation/admin-guide/cgroup-v2.rst — authority for cgroups.
- Liz Rice, "Containers From Scratch" (talk + repo) — the Go re-exec pattern.
- TLPI (Kerrisk), ch. ~24–27 (fork/exec/wait) and ~20–22 (signals). Its
  namespace/cgroup material is dated; use LWN/man7 there.
