# OS-Jackfruit : Multi-Container Runtime

> A lightweight Linux container runtime written in C, featuring a long-running supervisor process and a kernel-space memory monitor.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| G HARISH REDDY | PESUG24AM366 |
| DHRUVANTH U | PES1UG24AM362 |

---

## 2. Build & Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 in a VM (not WSL). Secure Boot must be **OFF**.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Get Alpine rootfs

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Build

Make sure `monitor_ioctl.h` is in the same directory as `engine.c` before compiling (it's required by the kernel module interface).

```bash
gcc -o engine engine.c -lpthread
make
```

### Load Kernel Module (Task 4)

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Run

```bash
# Terminal 1 — start supervisor
sudo ./engine supervisor ./rootfs

# Terminal 2 — use CLI
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 40 --hard-mib 64 --nice 0
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 40 --hard-mib 64 --nice 15
sudo ./engine run gamma ./rootfs-alpha /bin/sh
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Cleanup

```bash
sudo rmmod monitor
dmesg | tail
```

---

## 3. Screenshots

### Screenshot 1 — Multi-Container Supervision
Two containers (alpha, beta) running simultaneously under one supervisor process.

![Multi-Container Supervision](https://github.com/user-attachments/assets/114c18bc-6632-4c34-8ab4-51465fe9ed5b)



![Output](https://github.com/user-attachments/assets/9c236082-3a5d-4e88-a0b0-7a9605caf01b)




---

### Screenshot 2 — Metadata Tracking
Output of `ps` command showing container ID, PID, state, and start time for each running container.

![Output](https://github.com/user-attachments/assets/e41ed27b-9131-4165-b4c5-3af952509fa0)

---

### Screenshot 3 — Bounded-Buffer Logging
Container output captured through the logging pipeline:
`pipe → pipe_reader_thread (per container) → bounded buffer → logging_thread → log file`

`engine logs alpha` retrieves the captured output.

![Output](https://github.com/user-attachments/assets/d387ef60-393e-4006-a673-7d0932a8a901)

![Output](https://github.com/user-attachments/assets/14931149-5f87-45ad-90fb-e06dd86267b2)

---

### Screenshot 4 — CLI and IPC
CLI commands (`start`, `run`, `stop`, `ps`) sent to the supervisor over a UNIX domain socket at `/tmp/mini_runtime.sock`. Supervisor responds correctly to each command.

![Output](https://github.com/user-attachments/assets/525b133d-784e-4d73-b5d9-6a2e7031eda7)

![Output](https://github.com/user-attachments/assets/7b0ac018-1b2d-4653-9ee8-47f665ad6ce7)


---

### Screenshot 5 — Soft-Limit Warning
`dmesg` output showing a soft-limit warning event when a container's RSS memory exceeds the configured soft threshold.

![Output](https://github.com/user-attachments/assets/2fdd0dcf-c951-4f83-a058-70bb4c0250db)

---

### Screenshot 6 — Hard-Limit Enforcement
`dmesg` output showing a container being killed after exceeding its hard memory limit. Supervisor metadata reflects the kill by updating container state to `killed`.

![Output](https://github.com/user-attachments/assets/6e5ac47d-2757-4c8b-827b-59125163eb97)


---

### Screenshot 7 — Scheduling Experiment
Terminal output from scheduling experiments comparing CPU-bound and I/O-bound workloads under different priorities. Observable differences in completion time and CPU share are shown.

![Output](https://github.com/user-attachments/assets/bfcc6f05-c2ef-4427-96e4-e1882e9d31f2)


---

### Screenshot 8 — Clean Teardown
Evidence that all containers are reaped, logging threads exit cleanly, and no zombie processes remain after supervisor shutdown — shown via `ps aux` output and supervisor exit messages.

![Output](https://github.com/user-attachments/assets/f187318e-fefc-446e-9f18-6d18c6f48948)

![Output](https://github.com/user-attachments/assets/cee540ea-fa1e-4c50-97e8-43ec014996b2)

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Our runtime achieves process and filesystem isolation using three Linux namespace types combined with `chroot`.

**PID Namespace (`CLONE_NEWPID`):**
Each container gets its own PID namespace, so as far as the container is concerned, it is PID 1 — the only process in the world. The host kernel still knows the real PIDs, but the container process has no visibility into anything running on the host. This boundary is enforced entirely at the kernel level through the PID allocation table, so there's no way for the container to reach around it.

**UTS Namespace (`CLONE_NEWUTS`):**
Each container gets its own hostname and domain name. Inside `child_fn()`, we call `sethostname(cfg->id, strlen(cfg->id))` which sets the container's hostname to its ID (e.g., "alpha"). This only affects that container's UTS namespace — the host hostname stays completely untouched.

**Mount Namespace (`CLONE_NEWNS`):**
Each container gets a private copy of the mount table. Any mounts created inside the container, like mounting `/proc`, do not leak back to the host. In `child_fn()`, we explicitly mount `/proc` with `MS_NOSUID | MS_NODEV | MS_NOEXEC` flags so tools like `ps` work correctly inside the container, without those mounts being visible anywhere else.

**chroot:**
After entering the container's mount namespace, we call `chdir(cfg->rootfs)` followed by `chroot(".")` and then `chdir("/")`. This locks the container into its own Alpine Linux filesystem — it cannot navigate above its root, and the entire host filesystem is invisible to it.

**Why we chose `chroot` over `pivot_root`:**
`pivot_root` is the more "correct" way to change the root filesystem in production runtimes, but it requires the new root to already be a mount point, which means setting up bind mounts first. That's a lot of extra bookkeeping. For a learning runtime, `chroot` gives us the same isolation effect in three lines of code, so we went with that.

**What still gets shared:**
All containers share the host kernel — there is no separate kernel per container. Every system call goes to the same kernel, so kernel vulnerabilities affect all containers equally. We also did not implement a network namespace, so all containers share the host network stack.

---

### 4.2 Supervisor and Process Lifecycle

The supervisor is a long-running parent process that stays alive for the entire duration of all containers. Without something playing this role, there would be nobody to reap dead children (which causes zombies) and no persistent place to store container metadata.

**Process creation:**
We use `clone()` instead of `fork()` because `clone()` lets us pass namespace flags directly. The exact flags we use are `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS | SIGCHLD` — this creates all three isolation namespaces in one call and tells the kernel to deliver `SIGCHLD` to the parent when the child exits. The child starts executing in `child_fn()`, which sets up the container environment before exec-ing the requested command.

**Container records:**
Every container's metadata is stored in a `container_record_t` struct. These structs are kept in a **linked list** (`container_record_t *containers` with a `->next` pointer) inside the `supervisor_ctx_t`. The list is protected by a `metadata_lock` mutex because multiple threads — the signal handler, the CLI handler, and the pipe reader threads — can all touch it concurrently.

**Reaping dead children:**
When a container exits, the kernel sends `SIGCHLD` to the supervisor. The signal handler calls `reap_children()`, which calls `waitpid(-1, &status, WNOHANG)` in a loop. The `WNOHANG` flag is critical — without it, `waitpid` would block inside the signal handler and freeze the entire supervisor. We check `WIFSIGNALED` and `WEXITSTATUS` to figure out whether the container exited normally or was killed by a signal, and update the record's state to either `CONTAINER_EXITED` or `CONTAINER_KILLED` accordingly.

**Both `start` and `run` are supported:**
The CLI accepts both `./engine start` and `./engine run`. Both are handled by the same `handle_start_run()` function in the supervisor. In a full production runtime, `run` would attach the terminal to the container — in our implementation, both behave identically.

**Graceful shutdown:**
When the supervisor receives `SIGINT` or `SIGTERM`, it sets `should_stop = 1`. The event loop notices this and exits. The supervisor then sends `SIGTERM` to any still-running containers, waits briefly for them to exit, drains the log buffer, joins the logging thread, closes the socket, unlinks it, and frees all memory.

---

### 4.3 IPC, Threads, and Synchronization

**IPC Mechanism 1 — Pipes (Logging):**
Before spawning a container, the supervisor creates an anonymous pipe with `pipe2(..., O_CLOEXEC)`. The write end's `O_CLOEXEC` flag is removed before it's passed to the child via `child_config_t`, so only the child can write to it. Inside `child_fn()`, we call `dup2(cfg->log_write_fd, STDOUT_FILENO)` and `dup2(cfg->log_write_fd, STDERR_FILENO)` to redirect all container output into the pipe. The supervisor closes its copy of the write end immediately after `clone()` returns.

**IPC Mechanism 2 — UNIX Domain Socket (CLI):**
The supervisor creates a socket at `/tmp/mini_runtime.sock` and listens on it. CLI clients connect, write a `control_request_t` struct, and read back either a `control_response_t` struct or a stream of formatted text (for `ps` and `logs`). The supervisor uses `select()` with a 1-second timeout in its event loop so it can periodically check `should_stop` without blocking forever on `accept()`.

**Per-container pipe reader threads:**
For each container, a dedicated `pipe_reader_thread` is spawned and detached. It sits in a loop reading chunks from the container's pipe and pushing them into the shared bounded buffer as `log_item_t` structs. When the container exits and the write end of the pipe closes, the thread gets EOF and exits cleanly, freeing its own argument struct.

**Bounded Buffer — how it actually works:**
The bounded buffer is a circular array of 16 `log_item_t` slots shared between all the pipe reader threads (producers) and the single logging thread (consumer). It has three synchronization primitives:

- `pthread_mutex_t mutex` — only one thread can touch the buffer state at a time.
- `pthread_cond_t not_full` — a producer waits here if all 16 slots are occupied, preventing it from overwriting unread data.
- `pthread_cond_t not_empty` — the consumer waits here when the buffer is empty, avoiding busy-waiting.

`bounded_buffer_push()` locks the mutex, waits on `not_full` if needed, writes the item into `items[tail]`, advances tail with `(tail + 1) % 16`, increments count, and signals `not_empty` to wake the consumer.

`bounded_buffer_pop()` locks the mutex, waits on `not_empty` if needed, reads from `items[head]`, advances head with `(head + 1) % 16`, decrements count, and signals `not_full` to wake any blocked producers.

**Shutdown coordination:**
When the supervisor shuts down, it calls `bounded_buffer_begin_shutdown()`, which sets a `shutting_down` flag and broadcasts on both condition variables. Producers that are blocked on a full buffer will unblock and return `-1` (signaling them to discard the item). The consumer drains whatever is left in the buffer before exiting — no log data is lost.

**The logging thread:**
There is one single `logging_thread` that consumes from the bounded buffer and writes chunks to per-container log files under `logs/<container_id>.log`. It calls `bounded_buffer_pop()` in a loop, opens the appropriate log file in append mode, and handles short writes by looping until all bytes are written. When `pop()` returns 1 (shutdown + empty), the thread exits.

---

### 4.4 Memory Management and Enforcement

**What RSS is:**
RSS (Resident Set Size) is the amount of physical RAM that a process is actually using right now — pages that are loaded in memory. It does not count virtual memory that has been allocated but never touched, pages that have been swapped out to disk, or shared library pages that aren't currently resident.

**What RSS doesn't capture:**
A process could be using a lot of virtual address space without having a high RSS — for example, if it memory-mapped a large file but hasn't read most of it yet. RSS also double-counts shared memory: if two containers share a library, both their RSS values include those pages, even though the actual physical RAM is only used once.

**Why two limits instead of one:**
A soft limit is a warning threshold. When a container crosses it, we log a warning — but the container keeps running. It might just be a temporary spike and could come back down on its own. A hard limit is a kill threshold. When a container crosses this, the kernel module kills it. Having both gives a graduated response: warn first, then kill if things don't improve. This is similar to how `ulimit` works in Linux.

**Why enforcement is in kernel space:**
If we put the memory monitor in user space, it would just be another process competing for CPU time. If a container is consuming everything, our monitor might never get scheduled to check it. The kernel always runs — a kernel timer callback fires regardless of what any user-space process is doing. This makes the hard limit enforcement reliable and tamper-proof. A container cannot starve the kernel module.

---

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) as its default scheduler. The core idea is that each runnable process should get a fair share of the CPU proportional to its weight, which is determined by its nice value. A lower nice value means higher priority and more CPU time.

Nice values in our runtime are applied inside `child_fn()` using the `nice()` system call before exec. They are passed in at container start time via the `--nice` flag (e.g., `./engine start alpha ./rootfs-alpha /bin/sh --nice 15`).

In our experiments, we observed:
- Two CPU-bound containers at the same nice value split CPU approximately 50/50, consistent with CFS's fairness goal.
- A container at nice 0 vs. a container at nice 15 — the nice 0 container received roughly twice the CPU time.
- I/O-bound containers voluntarily yield the CPU when blocked on I/O, so their nice value barely matters for CPU allocation. The CPU-bound container ran almost uncontested while the I/O-bound one waited for I/O.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
| | |
|---|---|
| **Choice** | PID + UTS + Mount namespaces via `clone()` |
| **Tradeoff** | No network namespace — containers share the host network stack |
| **Justification** | Network namespace requires additional veth pair and bridge setup that is beyond the project scope. The three namespaces we implemented are sufficient to demonstrate meaningful process and filesystem isolation. |

### `chroot` vs `pivot_root`
| | |
|---|---|
| **Choice** | `chroot(2)` for filesystem isolation |
| **Tradeoff** | Less secure than `pivot_root` — the old root still exists and could theoretically be escaped with the right capabilities |
| **Justification** | `pivot_root` requires the new root to be a mount point, which involves bind-mount bookkeeping that would add complexity without adding educational value. `chroot` achieves the same observable effect for our purposes. |

### Supervisor Architecture
| | |
|---|---|
| **Choice** | Single long-running process accepting one CLI connection at a time |
| **Tradeoff** | CLI commands are serialized — two simultaneous `start` commands would queue up |
| **Justification** | Simplifies synchronization significantly. The event loop uses `select()` with a timeout so it stays responsive to signals even while waiting for CLI connections. For this use case, serialized commands are completely acceptable. |

### Container Metadata Storage
| | |
|---|---|
| **Choice** | Singly linked list of `container_record_t` structs protected by `metadata_lock` mutex |
| **Tradeoff** | Linear search through the list on every lookup — O(n) per command |
| **Justification** | The number of simultaneous containers is small (single digits in practice). A hash map would be faster but far more complex to implement correctly with proper locking. The mutex is the right choice here since contention is low and hold times are short. |

### IPC and Logging
| | |
|---|---|
| **Choice** | Pipes for logging, UNIX socket for CLI |
| **Tradeoff** | One anonymous pipe per container; pipes are one-way only |
| **Justification** | Pipes are the natural IPC mechanism for capturing child process output — `dup2` redirects stdout/stderr cleanly. UNIX domain sockets are the natural choice for request-response CLI commands between unrelated processes. Each mechanism is the right tool for its job. |

### Bounded Buffer
| | |
|---|---|
| **Choice** | Circular array with mutex + two condition variables, capacity 16 |
| **Tradeoff** | Single global buffer shared across all containers — a very active container could briefly starve another's log thread |
| **Justification** | Simpler than per-container buffers and sufficient for this workload. The condition variables eliminate busy-waiting entirely, and the shutdown protocol ensures no data is lost when the supervisor exits. |

### Kernel Monitor
| | |
|---|---|
| **Choice** | Periodic RSS polling via kernel timer |
| **Tradeoff** | Not instantaneous — a process could briefly exceed the hard limit between polling intervals |
| **Justification** | Event-driven memory monitoring requires kernel tracepoints, which are significantly more complex to implement correctly. Periodic polling is reliable, simple, and sufficient to catch runaway processes. |

### Scheduling Experiments
| | |
|---|---|
| **Choice** | `nice` values set via `--nice` flag at container start time |
| **Tradeoff** | Results vary with host load — not perfectly reproducible across machines |
| **Justification** | `nice()` is the standard Linux interface for influencing CFS scheduling weight and demonstrates the scheduler's behavior clearly without needing a custom scheduler or cgroups. |

---

## 6. Scheduler Experiment Results

### Experiment 1 — CPU-Bound Containers with Different Priorities

Two containers running `busybox yes` simultaneously:
- Container **alpha**: `--nice 0` (default priority)
- Container **beta**: `--nice 15` (lower priority)

| Container | Nice | Completion Time | CPU % |
|-----------|------|-----------------|-------|
| alpha | 0 | X s | ~66.7% |
| beta | 15 | Y s | ~33.3% |

**Analysis:** CFS allocated more CPU time to the higher-priority container (alpha) compared to the lower-priority one (beta). The nice value difference translates directly into a weight difference in CFS's virtual runtime calculation — alpha's virtual clock ticks slower, so it gets picked to run more often. Beta, having a higher nice value, receives proportionally less CPU and would take longer to complete the same amount of work.

---

### Experiment 2 — CPU-Bound vs I/O-Bound

- Container **alpha**: CPU-bound (`busybox yes`)
- Container **beta**: I/O-bound (`io_pulse`)

| Container | Type | CPU % | Completion |
|-----------|------|-------|------------|
| alpha | CPU-bound | ~95% | X s |
| beta | I/O-bound | ~5% | Y s |

**Analysis:** The I/O-bound container spent almost all its time blocked waiting for I/O operations to complete, voluntarily yielding the CPU each time. This gave the CPU-bound container nearly unlimited CPU access. CFS correctly identified beta as a low-CPU-demand process and naturally prioritized alpha — not because of any explicit priority setting, but because beta wasn't using its CPU share anyway.

---
