
This project implements a lightweight container runtime similar to Docker using Linux namespaces and a kernel module for memory monitoring and scheduling analysis.

## 1. Team Information

| Name | SRN |
|------|-----|
| Jagadeesh R | PES1UG24CS195 |
| Himesh Raghav | PES1UG24CS188  |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot OFF.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate
make
```

This builds `engine`, `monitor.ko`, `memory_hog`, `cpu_hog`, and `io_pulse`.

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

### Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

Leave this running. Open a second terminal for CLI commands.

### Launch Containers

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96
```

### CLI Commands

```bash
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Run a Container and Wait for Exit

```bash
cp -a ./rootfs-base ./rootfs-test
sudo ./engine run test ./rootfs-test "echo hello from container"
sudo ./engine logs test
```

### Memory Limit Test

```bash
cp -a ./rootfs-base ./rootfs-mem
cp ./memory_hog ./rootfs-mem/
sudo ./engine run memtest ./rootfs-mem /memory_hog --soft-mib 10 --hard-mib 20
sudo dmesg | tail -10
```

### Scheduler Experiment

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

### Shutdown and Cleanup

```bash
# Stop supervisor with Ctrl+C in supervisor terminal
sudo rmmod monitor
sudo dmesg | tail -5
```

### CI-Safe Build

```bash
make -C boilerplate ci
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
<img width="1016" height="220" alt="image" src="https://github.com/user-attachments/assets/d8ab27a0-2ba0-4eea-a612-6524c8c4cb1f" />

<img width="1016" height="293" alt="image" src="https://github.com/user-attachments/assets/0c914fc8-3399-450b-9136-f338b0f5c413" />


Caption: Both containers running simultaneously under a single supervisor process.

### Screenshot 2 — Metadata tracking

<img width="1016" height="165" alt="image" src="https://github.com/user-attachments/assets/d6eea543-d4f3-4f44-abd1-89e7d72b9015" />

Caption: `ps` command showing container ID, host PID, state, start time, memory limits, and termination reason.

### Screenshot 3 — Bounded-buffer logging

<img width="1016" height="248" alt="image" src="https://github.com/user-attachments/assets/f86f8e34-ab9f-4fe9-b6eb-7222dfec6c66" />


Caption: Container stdout captured via pipe, passed through bounded buffer, and written to log file by consumer thread.

### Screenshot 4 — CLI and IPC

<img width="1016" height="143" alt="image" src="https://github.com/user-attachments/assets/193d70c9-7975-4caf-a405-036d50b2bf37" />


Caption: CLI client connects to supervisor over UNIX domain socket at `/tmp/mini_runtime.sock`, sends request, receives response.

### Screenshot 5 — Soft-limit warning

<img width="1016" height="584" alt="image" src="https://github.com/user-attachments/assets/81acbd40-cba4-4a04-a6ca-f58df7644c9d" />


Caption: Kernel module logs a warning when container RSS exceeds soft limit of 10 MiB.

### Screenshot 6 — Hard-limit enforcement

<img width="1016" height="471" alt="image" src="https://github.com/user-attachments/assets/6912a7a6-1ecd-4158-866b-9a22644c1f0b" />


Caption: Kernel module sends SIGKILL when RSS exceeds 20 MiB hard limit. Supervisor metadata shows `hard_limit_killed`.

### Screenshot 7 — Scheduling experiment

<img width="1016" height="275" alt="image" src="https://github.com/user-attachments/assets/b4435fcb-86ee-40a9-b910-7cf8dbe5e298" />
<img width="1016" height="275" alt="image" src="https://github.com/user-attachments/assets/4134da2e-101f-4f41-b2c4-06e417433b14" />
<img width="1016" height="275" alt="image" src="https://github.com/user-attachments/assets/a12b91a3-7257-429c-8861-61b27b72a0bc" />


Caption: c2 (nice -5, higher priority) received larger CPU time slices. c1 (nice 10) was preempted, visible as gaps in progress reporting.

### Screenshot 8 — Clean teardown

<img width="1016" height="272" alt="image" src="https://github.com/user-attachments/assets/2487a6d7-4083-422a-814a-c05279daeef0" />


Caption: Supervisor stops all containers, joins logging thread, and frees resources. No zombie processes remain.

---

## 4. Engineering Analysis

### 1. Isolation Mechanisms

The runtime achieves isolation using Linux namespaces and chroot. When launching a container, `clone()` is called with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS`. `CLONE_NEWPID` gives the container its own PID namespace so it sees itself as PID 1 and cannot observe host processes. `CLONE_NEWUTS` provides an independent hostname. `CLONE_NEWNS` creates a separate mount namespace so filesystem mounts inside the container do not affect the host.

The child process calls `chroot()` into its assigned rootfs directory, making that directory appear as `/`. `/proc` is mounted inside the container namespace with `mount("proc", "/proc", "proc", 0, NULL)` so process utilities work correctly inside.

The host kernel still shares the same kernel code, network namespace, IPC namespace, and user namespace with all containers. This means containers share the host IP stack and kernel resources. Full network and user isolation would require additional namespace flags.

### 2. Supervisor and Process Lifecycle

A long-running supervisor is necessary because containers are child processes requiring a parent to reap them. Without a persistent parent, exited containers become zombies — their PCB stays in the process table until someone calls `wait()`. The supervisor installs a `SIGCHLD` handler that calls `waitpid(-1, WNOHANG)` in a loop to reap all exited children immediately.

Each container is created via `clone()`, which returns the child PID to the supervisor. This PID is stored in a linked list of `container_record_t` structs protected by a mutex. When SIGCHLD fires, the handler matches the PID to a record, updates its state, and classifies the termination reason using the `stop_requested` flag and exit signal. If `stop_requested` is set, termination is classified as `stopped`. If SIGKILL arrived without `stop_requested`, it is classified as `hard_limit_killed`.

### 3. IPC, Threads, and Synchronization

The project uses two IPC mechanisms.

**Path A — Logging via pipes:** Each container's stdout and stderr are redirected via `dup2()` to a pipe write end. A producer thread per container reads from the pipe read end and pushes chunks into a shared bounded buffer. A single consumer thread pops chunks and writes them to per-container log files in the `logs/` directory.

The bounded buffer uses a mutex and two condition variables (`not_empty`, `not_full`). Without the mutex, concurrent producers could corrupt the head and tail indices. Without condition variables, threads would busy-wait. The `not_full` variable blocks producers when the buffer is full, preventing data loss. The `not_empty` variable puts the consumer to sleep when nothing is available.

**Path B — Control via UNIX domain socket:** CLI commands connect to `/tmp/mini_runtime.sock`. The supervisor uses `select()` to wait for connections without blocking. Each connection sends a `control_request_t` and receives a `control_response_t`. This is a separate mechanism from pipes.

The container metadata list is protected by `metadata_lock`. This prevents races between the SIGCHLD handler updating state and the event loop reading state for `ps` or `stop`.

### 4. Memory Management and Enforcement

RSS (Resident Set Size) measures physical RAM currently occupied by a process — pages actually loaded in memory. RSS does not measure virtual memory, memory-mapped files not yet faulted in, or shared library pages. It approximates actual physical memory usage.

Soft and hard limits serve different purposes. A soft limit is a warning threshold: when RSS crosses it, the kernel module logs a warning once but lets the process continue. A hard limit is an enforcement threshold: when RSS crosses it, the process is killed immediately with SIGKILL.

Memory enforcement belongs in kernel space because a misbehaving process cannot interfere with it. A user-space monitor could be killed or starved by the same process it monitors. The kernel module uses a timer callback to check RSS via `get_task_mm()` and delivers SIGKILL via `send_sig()` — operations that cannot be blocked by the monitored process.

### 5. Scheduling Behavior

The Linux CFS (Completely Fair Scheduler) uses virtual runtime to track CPU time consumed by each process. The `nice` value adjusts the scheduling weight. A lower nice value means higher weight — the scheduler considers that process to have consumed less virtual runtime per real second, so it gets scheduled more often and for longer slices.

In our experiment, c1 ran at nice 10 and c2 at nice -5. Both ran `cpu_hog` for 20 seconds simultaneously. c1 reported progress at elapsed 1,2,3,4,5,6,7 then jumped to 12,13 — showing it was preempted for several seconds while c2 ran. c2 reported at elapsed 1,6,7 — it received larger CPU time slices and did more computation per second.

This demonstrates CFS giving c2 proportionally more CPU time based on its higher weight, while c1 was preempted more frequently due to its lower weight.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation

**Choice:** `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` with `chroot`.

**Tradeoff:** `chroot` is simpler than `pivot_root` but allows potential escape via path traversal if the container process has privileges.

**Justification:** For this project scope, `chroot` is sufficient to demonstrate filesystem isolation. The containers run inside a minimal Alpine rootfs with no network access.

### Supervisor Architecture

**Choice:** Single-process supervisor with a `select()` event loop handling one client at a time.

**Tradeoff:** Cannot handle multiple simultaneous CLI connections. A threaded model would scale better.

**Justification:** CLI commands are short-lived and infrequent. A single-client model avoids threading bugs in the control path and is sufficient for this use case.

### IPC and Logging

**Choice:** UNIX domain socket for control, pipes for logging, bounded buffer with mutex and condition variables.

**Tradeoff:** The bounded buffer adds implementation complexity compared to writing directly to files from the container.

**Justification:** The bounded buffer decouples container output rate from disk write rate. If disk I/O is slow, producers block rather than lose data. Mutex and condition variables are the correct primitives here because threads sleep inside buffer operations.

### Kernel Monitor

**Choice:** Mutex over spinlock for the monitored list.

**Tradeoff:** Mutex has higher overhead than spinlock for short critical sections.

**Justification:** The timer callback calls `get_rss_bytes()` which calls `get_task_mm()`, a function that can sleep. Spinlocks cannot be held across sleepable code paths in the kernel, so a mutex is required.

### Scheduling Experiments

**Choice:** `nice` values to demonstrate priority differences rather than CPU affinity.

**Tradeoff:** Nice values affect scheduling weight but both containers can still run on all available CPUs. CPU affinity would force strict CPU isolation.

**Justification:** Nice values directly demonstrate CFS weighted scheduling, which is the core Linux scheduling mechanism. The effect is clearly observable in the log output.

---

## 6. Scheduler Experiment Results

### Experiment: Two CPU-bound containers with different priorities

Both containers ran `/cpu_hog 20` (20-second CPU burn) simultaneously on the same machine.

| Container | Nice Value | Progress Reported At (elapsed seconds) | Gaps |
|-----------|-----------|----------------------------------------|------|
| c1 | +10 (low priority) | 1, 2, 3, 4, 5, 6, 7, 12, 13 | 4-second gap at 8–11 |
| c2 | -5 (high priority) | 1, 6, 7 | Got larger slices, fewer prints |

### Raw Log Output

**c1 log (nice 10):**
```
cpu_hog alive elapsed=1 accumulator=4548337513673617523
cpu_hog alive elapsed=2 accumulator=3469723412426831115
cpu_hog alive elapsed=3 accumulator=13684638064056133321
cpu_hog alive elapsed=4 accumulator=9237104230200268010
cpu_hog alive elapsed=5 accumulator=11903977776708818429
cpu_hog alive elapsed=6 accumulator=14308564023570326925
cpu_hog alive elapsed=7 accumulator=2312219186486121679
cpu_hog alive elapsed=12 accumulator=13583581612097865815
cpu_hog alive elapsed=13 accumulator=11636555306494358935
cpu_hog done duration=20
```

**c2 log (nice -5):**
```
cpu_hog alive elapsed=1 accumulator=1950438522884953651
cpu_hog alive elapsed=6 accumulator=5628460969466473060
cpu_hog alive elapsed=7 accumulator=5848389523760964930
cpu_hog done duration=20
```

### Analysis

c1 was preempted between elapsed 7 and elapsed 12 — a 4-second window where c2 was consuming the CPU continuously. c2 received longer uninterrupted CPU slices, visible as larger gaps between its own progress prints.

The Linux CFS scheduler allocated CPU time proportional to process weight. A process with nice -5 has significantly higher weight than one with nice +10. This matches CFS behavior: the high-weight process receives more CPU time per scheduling period, causing the low-weight process to be preempted more often and wait longer before being rescheduled.

This project successfully demonstrates key operating system concepts including process isolation, inter-process communication, memory management, and scheduling using a practical container runtime implementation.
