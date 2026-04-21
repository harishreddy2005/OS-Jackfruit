# Multi-Container Runtime — OS Jackfruit

**G HARISH REDDY** — PES1UG24AM366  
**DHRUVANTH U** — PES1UG24AM362

---

## Overview

A lightweight Linux container runtime implemented in C, featuring a long-running supervisor process and a kernel-space memory monitor. The runtime supports multiple isolated containers with independent namespaces, resource limits, structured logging, and scheduling experiments.

---

## Design Decisions

### Architecture

- The **supervisor** runs as a persistent daemon and manages container lifecycle via Unix domain sockets for IPC. It maintains a shared state table of all containers and handles start, stop, run, and ps operations.
- Each container is isolated using Linux **namespaces** (PID, mount, UTS) and a **chroot** into a per-container Alpine Linux rootfs.
- **cgroups** are used to enforce soft and hard memory limits per container.
- The **kernel module** (`monitor.c`) registers containers by PID and periodically samples RSS from `/proc/<pid>/status`, emitting `SOFT LIMIT` and `HARD LIMIT` events to `dmesg` when thresholds are crossed.

### IPC Design

- A Unix domain socket at a fixed path serves as the command channel between the CLI (`engine`) and the supervisor.
- Commands are serialized as simple ASCII structs; responses are line-delimited text. This kept the protocol simple and debuggable with standard tools.

### Scheduling Experiments

- Two CPU hog containers were launched with different `nice` values (0 and 10) to observe scheduler behaviour under contention.
- `pidstat` was used to measure actual CPU allocation across the two containers, demonstrating the effect of CFS priority.

---

## Implementation Tasks Completed

- [x] Multi-container runtime with namespace and chroot isolation
- [x] CLI: `start`, `stop`, `run`, `ps` subcommands
- [x] Persistent supervisor with Unix socket IPC
- [x] Per-container log files under `logs/`
- [x] Kernel module with ioctl interface for container registration and memory monitoring
- [x] Scheduling experiments with `nice`-based priority
- [x] Graceful cleanup on supervisor termination

---

## Screenshots

### 1. Supervisor startup and container start (alpha, beta)

<img width="1600" height="900" alt="1" src="https://github.com/user-attachments/assets/db97a0b6-314b-4086-994d-b3a1085d00a3" />


### 2. `engine ps` showing running containers

<!-- Paste screenshot here -->

---

### 3. `engine run`, `engine stop`, and log output

<!-- Paste screenshot here -->

---

### 4. Multi-container ps with stopped/exited states

<!-- Paste screenshot here -->

---

### 5. Kernel module — soft and hard memory limit events in dmesg

<!-- Paste screenshot here -->

---

### 6. Scheduling experiment — CPU hog containers with different nice values

<!-- Paste screenshot here -->

---

### 7. Supervisor with multiple containers; cleanup via kill

<!-- Paste screenshot here -->

---

## Engineering Analysis

### Namespace Isolation

Each container receives its own PID and mount namespace. The `chroot` into a per-container rootfs ensures filesystem isolation. UTS namespace isolation allows each container to have an independent hostname.

### Memory Enforcement

The kernel module samples each registered container's RSS every poll interval. When RSS exceeds the soft limit, a warning is logged to `dmesg`. When RSS exceeds the hard limit, the module sends `SIGKILL` to the container process and logs a hard limit event. This mirrors the behaviour of the Linux OOM killer but scoped to per-container limits.

### IPC Tradeoffs

Unix domain sockets were chosen over pipes because they support multiple concurrent clients connecting to the supervisor without requiring a separate pipe pair per client. The downside is that the socket file must be cleaned up on exit.

### Scheduling Observations

When two CPU hog workloads ran simultaneously — one at nice 0 and one at nice 10 — the CFS scheduler allocated roughly 3:1 CPU share in favour of the lower-nice process. This matches the expected behaviour from the CFS weight table (nice 0 weight = 1024, nice 10 weight = 110).

### Challenges

- **Zombie reaping**: The supervisor must `waitpid` on stopped containers to prevent zombie accumulation. We used `SIGCHLD` handling with `waitpid(-1, WNOHANG)` in a loop.
- **Concurrent access to container table**: Protected with a mutex since the supervisor handles IPC in a separate thread from the SIGCHLD handler.
- **Kernel module stability**: Accessing `/proc/<pid>/status` from kernel space required using `kernel_read` on the proc file; direct `task_struct` RSS fields were used for efficiency in the final implementation.

---

## Build Instructions

```bash
# Install dependencies
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)

# Build everything
cd boilerplate
make

# CI-safe build (no kernel module)
make ci
```

---

## Running

### Start the supervisor

```bash
sudo ./engine supervisor $(pwd)/rootfs-base 2>&1
```

### Start containers

```bash
sudo ./engine start alpha $(pwd)/rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta  $(pwd)/rootfs-beta  /bin/sh --soft-mib 64 --hard-mib 96
```

### List containers

```bash
sudo ./engine ps
```

### Run a one-shot command

```bash
sudo ./engine run alpha $(pwd)/rootfs-alpha /bin/echo "hello" --soft-mib 48 --hard-mib 80
```

### Stop a container

```bash
sudo ./engine stop alpha
```

### Load the kernel module

```bash
sudo insmod monitor.ko
```

---

## Repository Structure

```
boilerplate/
├── engine.c          # User-space runtime and supervisor
├── monitor.c         # Kernel module
├── monitor_ioctl.h   # Shared ioctl definitions
├── Makefile
├── cpu_hog.c
├── io_pulse.c
├── memory_hog.c
└── environment-check.sh
```

---

## References

- Linux man pages: `clone(2)`, `chroot(2)`, `cgroups(7)`, `namespaces(7)`
- Linux kernel documentation: `proc(5)`, `task_struct`
- Alpine Linux minirootfs: https://alpinelinux.org/downloads/
