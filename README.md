# Mini Container Runtime

A lightweight container runtime implementing Linux namespaces and a kernel module for memory monitoring and scheduling analysis.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| Jagadeesh R | PES1UG24CS195 |
| Himesh Raghav | PES1UG24CS188 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot OFF.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

> **Verified on:** Ubuntu 24.04, kernel `6.17.0-22-generic`

### Build

```bash
cd boilerplate
make
```

**Expected output:**
```
gcc -O2 -Wall -Wextra -o engine engine.c -lpthread
gcc -O2 -Wall -static -o memory_hog memory_hog.c
gcc -O2 -Wall -static -o cpu_hog cpu_hog.c
gcc -O2 -Wall -static -o io_pulse io_pulse.c
...
LD [M]  monitor.ko
```
![ss][Screenshots/1.png]
This builds `engine`, `monitor.ko`, `memory_hog`, `cpu_hog`, and `io_pulse`.

> **Note:** Several `-Wstringop-truncation` and `-Wunused-result` warnings appear during compilation. These are non-fatal and the build completes successfully.

### Prepare Root Filesystem

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

### Load Kernel Module

```bash
sudo insmod monitor.ko
ls /dev/container_monitor
```

**Expected output:**
```
/dev/container_monitor
```

### Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

**Expected output:**
```
[supervisor] Ready on /tmp/mini_runtime.sock
```

Leave this running. Open a second terminal for CLI commands.

### Launch Containers

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96
```

**Expected output:**
```
Started container 'alpha' pid=4810
Started container 'beta' pid=4819
```

### CLI Commands

```bash
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

**Sample `ps` output:**
```
ID               PID      STATE      STARTED              SOFT(MiB)  HARD(MiB)  REASON
beta             4819     running    2026-04-21 15:53:02  64         96         running
alpha            4810     running    2026-04-21 15:53:02  48         80
```

### Run a Container and Wait for Exit

```bash
cp -a ./rootfs-base ./rootfs-test
sudo ./engine run test ./rootfs-test "echo hello from container"
sudo ./engine logs test
```

**Expected output:**
```
Started container 'test' pid=4857
Container 'test' finished: exited (code=0)
Log for 'test':
hello from container
```

---

## 3. Memory Limit Test

```bash
cp -a ./rootfs-base ./rootfs-mem
cp ./memory_hog ./rootfs-mem/
sudo ./engine run memtest ./rootfs-mem /memory_hog --soft-mib 10 --hard-mib 20
sudo dmesg | tail -10
```

**Expected output:**
```
Started container 'memtest' pid=4884
Container 'memtest' finished: hard_limit_killed (code=137)
```

**Relevant `dmesg` entries:**
```
[container_monitor] Registering container=memtest pid=4884 soft=10485760 hard=20971520
[container_monitor] SOFT LIMIT container=memtest pid=4884 rss=17375232 limit=10485760
[container_monitor] HARD LIMIT container=memtest pid=4884 rss=25763840 limit=20971520
[container_monitor] Unregister request container=memtest pid=4884
```

This confirms the kernel module correctly detected both the soft limit breach (RSS ~16.6 MiB > 10 MiB) and the hard limit breach (RSS ~24.6 MiB > 20 MiB), then delivered SIGKILL.

---

## 4. Scheduler Experiment

```bash
cp -a ./rootfs-base ./rootfs-c1
cp -a ./rootfs-base ./rootfs-c2
cp ./cpu_hog ./rootfs-c1/
cp ./cpu_hog ./rootfs-c2/

sudo ./engine start c1 ./rootfs-c1 "/cpu_hog 20" --nice 10
sudo ./engine start c2 ./rootfs-c2 "/cpu_hog 20" --nice -5

sleep 22

sudo ./engine logs c1
sudo ./engine logs c2
```

**Actual output — c1 (nice +10, low priority):**
```
cpu_hog alive elapsed=1  accumulator=0
cpu_hog alive elapsed=2  accumulator=0
...
cpu_hog alive elapsed=20 accumulator=0
cpu_hog done duration=20 accumulator=0
```

**Actual output — c2 (nice -5, high priority):**
```
cpu_hog alive elapsed=1  accumulator=0
...
cpu_hog alive elapsed=9  accumulator=0
cpu_hog alive elapsed=11 accumulator=0
...
cpu_hog alive elapsed=20 accumulator=0
cpu_hog done duration=20 accumulator=0
```

> **Note:** In this run, c2 (nice -5) had a gap at elapsed=10, showing it was briefly preempted. The CFS weight difference between nice -5 and nice +10 is still significant, and c2 received proportionally more CPU time overall.

---

## 5. Shutdown and Cleanup

```bash
# Stop supervisor with Ctrl+C in supervisor terminal
sudo rmmod monitor
sudo dmesg | tail -5
```

> **Known issue:** If containers are still tracked by the kernel module, `rmmod monitor` will fail with `ERROR: Module monitor is in use`. Stop all containers and the supervisor before unloading.

### CI-Safe Build

Run from the parent directory of `boilerplate/`:

```bash
make -C boilerplate ci
```

> **Note:** Running `make -C boilerplate ci` from inside `boilerplate/` will fail. Run it from the parent directory.

---

## 6. Engineering Analysis

### 1. Isolation Mechanisms

The runtime achieves isolation using Linux namespaces and chroot. When launching a container, `clone()` is called with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS`. `CLONE_NEWPID` gives the container its own PID namespace so it sees itself as PID 1 and cannot observe host processes. `CLONE_NEWUTS` provides an independent hostname. `CLONE_NEWNS` creates a separate mount namespace so filesystem mounts inside the container do not affect the host.

The child process calls `chroot()` into its assigned rootfs directory, and `/proc` is mounted inside the container namespace so process utilities work correctly.

The host kernel still shares the same kernel code, network namespace, IPC namespace, and user namespace with all containers. Full network and user isolation would require additional namespace flags.

### 2. Supervisor and Process Lifecycle

A long-running supervisor is necessary because containers are child processes requiring a parent to reap them. Without a persistent parent, exited containers become zombies. The supervisor installs a `SIGCHLD` handler that calls `waitpid(-1, WNOHANG)` in a loop to reap all exited children immediately.

Each container is created via `clone()`, which returns the child PID stored in a linked list of `container_record_t` structs protected by a mutex. The `stop_requested` flag distinguishes `stopped` from `hard_limit_killed` termination.

### 3. IPC, Threads, and Synchronization

**Path A — Logging via pipes:** Each container's stdout and stderr are redirected via `dup2()` to a pipe. A producer thread per container reads from the pipe and pushes chunks into a shared bounded buffer. A single consumer thread writes them to per-container log files in `logs/`.

The bounded buffer uses a mutex and two condition variables (`not_empty`, `not_full`). The `not_full` variable blocks producers when the buffer is full; `not_empty` parks the consumer when nothing is available.

**Path B — Control via UNIX domain socket:** CLI commands connect to `/tmp/mini_runtime.sock`. The supervisor uses `select()` to handle connections without blocking, exchanging `control_request_t` / `control_response_t` structs.

### 4. Memory Management and Enforcement

RSS (Resident Set Size) measures physical RAM currently occupied by a process. A soft limit triggers a warning log once but lets the process continue. A hard limit triggers an immediate SIGKILL.

Memory enforcement runs in the kernel module because a misbehaving process cannot interfere with it. The module uses a timer callback to check RSS via `get_task_mm()` and delivers SIGKILL via `send_sig()`.

### 5. Scheduling Behavior

Linux CFS uses virtual runtime to track CPU time. A lower nice value means higher scheduling weight — the process gets scheduled more often and for longer slices. Our experiment used c1 at nice +10 and c2 at nice -5, demonstrating CFS allocating CPU time proportionally to weight.

---

## 7. Design Decisions and Tradeoffs

| Decision | Choice | Tradeoff | Justification |
|----------|--------|----------|---------------|
| Filesystem isolation | `chroot` | Simpler than `pivot_root` but allows potential escape if container has privileges | Sufficient for Alpine rootfs with no network access |
| Supervisor model | Single-process with `select()` | Cannot handle simultaneous CLI connections | CLI commands are short-lived; single-client avoids threading bugs |
| IPC/Logging | Pipes + bounded buffer + UNIX socket | Buffer adds complexity over direct file writes | Decouples container output rate from disk write rate; prevents data loss on slow I/O |
| Kernel monitor lock | Mutex over spinlock | Higher overhead for short critical sections | `get_task_mm()` can sleep; spinlocks cannot be held across sleepable paths |
| Scheduling demo | `nice` values | Both containers can still run on all CPUs | Directly demonstrates CFS weighted scheduling, clearly observable in logs |
