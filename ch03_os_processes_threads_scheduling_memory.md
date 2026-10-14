# Chapter 3: Operating Systems I: Processes, Threads, Scheduling, and Memory

## Where We Are

Chapter 1 drew the map: a web application is a set of **processes** on several machines, talking over networks. Chapter 2 went down to the hardware. A CPU core runs a fetch–decode–execute loop over instructions. Memory forms a hierarchy in which RAM is about a hundred times slower than L1 cache. A running program has a stack per thread and a shared heap, and a language runtime (a JIT, a garbage collector) sits between your code and the machine.

Chapter 2 left a gap on purpose. It described *one* program on *one* machine as if it owned the hardware. In reality a production server runs hundreds of processes and thousands of threads on a handful of cores, all sharing one pool of RAM, and none of them is allowed to touch the hardware directly. The piece of software that makes this possible is the **operating system**. This chapter explains how it works: how it creates and destroys processes, how it decides which thread runs next, how it gives every process its own private memory, and what happens when demand for CPU or memory exceeds supply.

These mechanisms are the source of many of the strangest backend symptoms: a service with 20% CPU but a load average of 40; a process that "vanished" with nothing in its logs; p99 latency that doubles after someone raised the thread pool size from 200 to 2,000; a database whose memory usage, summed across processes, exceeds the RAM of the machine. By the end of this chapter you can explain each of them.

**After this chapter you can:**
- explain what an operating system does, and why application code runs in a restricted mode;
- trace a system call from a line of application code into the kernel and back, and explain what it costs;
- describe a process, its Process Control Block and its lifecycle states, and draw the transitions between them;
- trace `fork`, `exec` and `wait`, explain copy-on-write, and diagnose zombie and orphan processes;
- explain signals and use them to reason about graceful versus forced shutdown;
- explain what a thread shares with its process and what it owns, and compare threading models (kernel threads, green threads, M:N, coroutines);
- compute schedules and average waiting times for FCFS, SJF, SRTF and Round Robin, and explain MLFQ and the idea behind Linux's fair scheduler;
- explain exactly what a context switch saves and restores and why too many threads hurt throughput;
- trace a virtual-to-physical address translation through a two-level page table and a TLB;
- run FIFO, LRU and Optimal page replacement by hand, demonstrate Belady's anomaly, and explain thrashing;
- explain swap, overcommit and the OOM killer, and diagnose a process that was killed for memory;
- read `top`, `vmstat`, `ps` and `/proc/<pid>` and tell high CPU, high load and high iowait apart.

**Interview questions this chapter prepares you for:**
1. What is the difference between a process and a thread?
2. What is a system call, and why is it more expensive than a function call?
3. What is the difference between user mode and kernel mode?
4. What are the states of a process, and what causes each transition?
5. How do `fork` and `exec` work? What is copy-on-write?
6. What is a zombie process? What is an orphan process? How is each cleaned up?
7. What is the difference between SIGTERM and SIGKILL? How does a server shut down gracefully?
8. What is the difference between user-level and kernel-level threads? What is M:N threading?
9. Explain FCFS, SJF, Round Robin and MLFQ. What is starvation, and how does aging fix it?
10. What happens during a context switch, and why is it expensive?
11. How does virtual memory work?
12. What is a page table? Why are page tables multi-level? What is the TLB?
13. What is a page fault? Minor versus major?
14. Compare FIFO, LRU and Optimal page replacement. What is Belady's anomaly?
15. What is thrashing, and how do you detect it?

**Scope.** Here: the CPU and memory side of the operating system, meaning privilege modes, system calls, processes, threads, scheduling and virtual memory. Files, file descriptors, the page cache, `fsync`, I/O models (`epoll`, `io_uring`), inter-process communication, containers, namespaces and cgroups are in Chapter 4. Race conditions, locks, semaphores and deadlock are in Chapter 5; this chapter mentions them only as forward references. Thread pools and execution models inside a server are in Chapter 11.

---

## 3.1 What an Operating System Is

### Definition

An **operating system (OS)** is the software that manages a computer's hardware resources (CPU time, memory, storage, network devices) and offers programs safe, simplified abstractions of them. Its privileged core, the **kernel**, is the only code allowed to touch the hardware directly. Everything else runs in **user space** and must ask the kernel for help.

### Why it exists

Imagine a machine with no operating system, running several programs at once. Each program would need its own disk driver and network driver. Any program could overwrite any other program's memory, by bug or on purpose. A program stuck in an infinite loop would keep the CPU forever, because nothing could take it away. Two programs writing to the disk at the same time would interleave their blocks and corrupt each other's files.

An operating system solves three problems at once:

| Role | Problem it solves | What the OS provides |
|---|---|---|
| **Resource manager** | Many programs compete for few cores, finite RAM, one disk, one network card | Scheduling of CPU time, allocation of memory, queuing of I/O requests, fair sharing and limits |
| **Abstraction layer** | Hardware is messy and differs between machines | Processes instead of raw CPUs, virtual memory instead of physical RAM, files instead of disk blocks, sockets instead of network-card registers |
| **Isolation and protection** | Programs are buggy or hostile | Each process gets a private address space; only the kernel can touch devices; permissions on files; the kernel can always take the CPU back |

Every abstraction in this list is a "virtualisation" of a physical thing. A **process** is a virtual computer: it believes it has a CPU and a large, private memory to itself. In reality it shares both with hundreds of other processes, and the OS multiplexes them so quickly that the illusion holds.

### Mechanism: kernel space and user space

The kernel is a large program, loaded at boot, that stays resident in memory for as long as the machine runs. It contains the scheduler (Section 3.10), the virtual memory manager (Sections 3.13–3.17), device drivers, file systems and the network stack (Chapter 4 and Chapter 6).

Memory is split into two regions with different rules:

```text
                 Virtual address space of every process (x86-64 Linux, simplified)

 high addresses  ┌──────────────────────────────────┐
                 │ KERNEL SPACE                     │  kernel code, kernel data,
                 │ (same mapping in every process,  │  page tables, per-thread
                 │  accessible only in kernel mode) │  kernel stacks, drivers
                 ├──────────────────────────────────┤
                 │        (unmapped gap)            │
                 ├──────────────────────────────────┤
                 │ USER SPACE                       │  your program: code, heap,
                 │ (different in every process)     │  stacks, libraries
 low addresses   └──────────────────────────────────┘
```

**Kernel space** is the memory and code of the kernel. **User space** is everything else: your application, its libraries, the language runtime, the database server, the shell. On Linux the kernel is mapped into the upper part of every process's address space, but the hardware refuses user-mode code that tries to read or write it. (Section 3.13 explains what an "address space" is and how the hardware enforces this; for now, take it as a fact.)

Linux is a **monolithic kernel**: the scheduler, memory manager, file systems, drivers and network stack all run together in kernel space. A **microkernel** design (for example, seL4 or QNX) keeps only a minimal core in kernel mode and runs drivers and file systems as user-space servers that communicate by messages. Monolithic kernels are faster (no message passing between subsystems); microkernels contain faults better (a crashing driver does not take down the kernel). Backends overwhelmingly run on Linux, so this book describes Linux behaviour unless it says otherwise.

### Guarantees and non-guarantees

The OS guarantees that a user-space process cannot directly access hardware or another process's memory, and that the kernel can always regain control of a CPU. It does **not** guarantee that your process gets any particular amount of CPU time or memory, that it will not be killed when the machine runs out of memory (Section 3.17), or that a kernel bug will not take down everything.

### Backend connection

Your web server, its language runtime, the database and the cache are all user-space processes. Every request they handle crosses into the kernel many times: to read bytes from a socket, to write a log line, to allocate memory, to wait for a database reply. Whenever a backend is slow and its own code looks innocent, the question "what is the kernel doing on its behalf?" is the next one to ask.

---

## 3.2 Privilege Modes, System Calls, and Interrupts

### Definitions

A **CPU privilege mode** is a hardware setting that controls which instructions the currently running code may execute and which memory it may touch. Code in **kernel mode** (also called supervisor mode or, on x86, ring 0) may do anything. Code in **user mode** (ring 3 on x86) cannot execute privileged instructions, cannot access kernel memory and cannot talk to devices.

A **system call (syscall)** is the controlled entry point through which user-mode code asks the kernel to perform a privileged operation on its behalf, such as reading from a file, sending bytes on a socket, creating a process or allocating memory. It is implemented with a special CPU instruction that switches the CPU into kernel mode and jumps to a fixed kernel entry point.

### Why they exist

If user code could run privileged instructions, isolation would be impossible: any program could disable interrupts (so the OS never regains control), rewrite the page tables (so it could read every other process's memory), or program the disk controller directly. The hardware therefore enforces the boundary. The OS still needs to *offer* services, so the hardware also provides exactly one controlled door through the wall: the system call instruction, which can only jump to an address the kernel chose at boot. User code cannot choose where in the kernel to land.

### Mechanism: a system call, step by step

Take one line of application pseudocode that writes a log line to standard output:

```pseudo
write_to_stdout("hello\n")      // 6 bytes
```

Here is what happens on x86-64 Linux:

```text
 USER MODE                                         KERNEL MODE
 ─────────                                         ───────────
 1. Application code calls the language's
    I/O routine, which eventually calls the
    C library wrapper:  write(1, buf, 6)
                │
 2. Library wrapper (e.g. glibc):
    - puts the syscall number for write (1)
      in register RAX
    - puts arguments in RDI=1 (fd), RSI=buf,
      RDX=6 (length)
    - executes the SYSCALL instruction
                │
                │  3. CPU, in hardware:
                │     - saves the user RIP (return address) in RCX
                │       and the flags in R11
                │     - switches to kernel mode
                └───────────────────────────────►  - jumps to the kernel entry address
                                                     stored in a CPU register at boot
                                                   4. Kernel entry code:
                                                     - switches to this thread's
                                                       kernel stack
                                                     - saves the remaining user
                                                       registers on it
                                                   5. Dispatch: looks up entry 1 in
                                                      the syscall table → the
                                                      kernel's write handler
                                                   6. Handler:
                                                     - checks fd 1 is open for writing
                                                     - checks buf is a valid user
                                                       address, copies 6 bytes in
                                                     - performs the write (terminal,
                                                       pipe, file, socket...)
                                                     - may BLOCK here if the target
                                                       is full (Section 3.4)
                                                   7. Puts the result (6, or a
                                                      negative error code) in RAX,
                                                      restores user registers
                ┌───────────────────────────────── 8. SYSRET: back to user mode,
                │                                     resume at the saved RIP
 9. Library wrapper:
    - if RAX is in the error range,
      stores the error code in errno
      and returns -1
    - else returns 6
                │
10. Application continues
```

You can watch the boundary from outside with `strace`, which prints every system call a process makes (shortened here):

```bash
$ strace -e trace=write ./hello
write(1, "hello\n", 6)                  = 6
+++ exited with 0 +++
```

The same mechanism serves every operation that leaves your process: `read` and `write` (files and sockets), `openat` and `close`, `mmap` and `brk` (memory), `clone`, `execve` and `wait4` (processes), `futex` (the kernel side of locks, Chapter 5), `epoll_wait` (waiting on many sockets, Chapter 4). Linux has a few hundred system calls in total.

### The cost of a system call

A function call is a jump and a few register saves: around a nanosecond. A system call is much more:

| Cost component | Why it happens |
|---|---|
| Mode switch in and out | The CPU changes privilege level, swaps stacks and saves registers |
| Security mitigations | After the 2018 Meltdown and Spectre CPU vulnerabilities, kernels added work on each entry and exit (for example, switching to a separate kernel page table on affected CPUs) |
| Argument validation and copying | The kernel cannot trust user pointers; it checks and copies data across the boundary |
| Cache and TLB pollution | Kernel code and data displace some of your program's cached lines (Chapter 2) |
| The actual work | Which may include waiting for a device |

A trivial system call (one that does almost no work) typically costs on the order of a hundred nanoseconds to a microsecond on modern hardware, depending on the CPU and which mitigations are enabled. That is cheap in absolute terms and expensive relative to a function call, so the cost matters when multiplied.

A worked example. Assume a trivial syscall costs 0.3 µs and a service writes a 64 KB response.

- Written one byte at a time: 65,536 `write` calls × 0.3 µs ≈ 20 ms of pure overhead.
- Buffered in user space and written once: 1 call, ≈ 0.3 µs of overhead plus the copy.

This is why every I/O library buffers, and why "how many syscalls per request?" is a real performance question. Linux also provides the **vDSO** (virtual dynamic shared object), a small piece of kernel-provided code mapped into every process that answers a few read-only calls, such as reading the current time, without entering the kernel at all.

> **Mode switch is not a context switch.** A system call always switches the CPU from user mode to kernel mode and back. It switches to a *different thread* only if the call blocks (for example, a `read` on a socket with no data yet) and the scheduler runs something else. Interviewers often test whether you conflate the two. Context switches are Section 3.11.

### Interrupts, traps and exceptions

The CPU leaves the normal flow of a user program in three ways. Terminology differs slightly between textbooks and CPU manuals; the distinction that matters is **who caused it** and **whether it was intended**.

| Event | Caused by | Synchronous with the running instruction? | Examples | After handling |
|---|---|---|---|---|
| **Interrupt** (hardware interrupt) | A device, external to the running program | No: can arrive between any two instructions | Network card received a packet; disk finished a read; the **timer** fired | The interrupted program resumes where it was |
| **Trap** | The program, *deliberately* | Yes | The `SYSCALL` instruction; a debugger breakpoint | Resume at the *next* instruction |
| **Exception / fault** | The program, *unintentionally* | Yes | Page fault (Section 3.15); divide by zero; invalid instruction | For a fault that the kernel can fix (a page fault), *retry the same* instruction; otherwise the kernel sends a signal (Section 3.6), usually killing the process |

All three work the same way at the hardware level: the CPU switches to kernel mode and jumps to a handler whose address the kernel registered in a table at boot (on x86, the interrupt descriptor table).

The **timer interrupt** deserves special attention. A hardware timer interrupts each core periodically or at a programmed deadline. Because the CPU *must* jump to the kernel's handler, the kernel is guaranteed to regain control even from a program stuck in `while (true) {}`. This is what makes **preemptive scheduling** (Section 3.10) possible.

### Polling versus interrupts

How does the kernel learn that a device has finished its work? Two strategies:

- **Polling:** the CPU repeatedly reads a device status register: "done yet? done yet?" Latency is minimal (you notice immediately), but the CPU does nothing else while it spins.
- **Interrupts:** the CPU does other work, and the device raises an interrupt when done. No CPU is wasted while waiting, but each interrupt has a fixed handling cost.

Neither wins everywhere. At low event rates, interrupts are better: polling would waste a core waiting for rare events. At very high rates (millions of packets per second), the per-interrupt cost dominates and the system can spend all its time in interrupt handlers, a condition called **interrupt livelock**. Linux's network stack therefore uses a hybrid called NAPI: the first packet raises an interrupt, then the kernel switches that device to polling until the queue is empty, then re-enables interrupts. Some ultra-low-latency systems dedicate whole cores to busy-polling the network card.

The same trade-off reappears at the application level. A program that loops asking "is there data on this socket yet?" wastes CPU; a program that asks the kernel to wake it when data arrives does not. Chapter 4 builds the I/O models (blocking, non-blocking, `epoll`) on exactly this idea.

### Failure and diagnosis

| Symptom | Likely cause | Evidence and tool |
|---|---|---|
| High `sy` (system CPU) percentage in `top` | Too many syscalls (tiny unbuffered writes, busy polling), or kernel work such as page faults or context switches | `strace -c -p <pid>` summarises syscall counts and time; `perf top` shows kernel functions |
| A process that is slow but shows low user CPU | Time spent waiting inside syscalls | `strace -T` shows time spent in each call |
| High `hi`/`si` (hardware/software interrupt) CPU | Very high packet rates | `/proc/interrupts`, `/proc/softirqs` |

`strace` itself slows the traced process considerably, because it stops the process on every syscall. Use it briefly and carefully on production systems.

### Backend connection

A web request typically makes dozens of system calls: reading the request from the socket, writing log lines, sending a query to the database and reading the reply, writing the response. Batching (buffered writes, pipelined database queries, larger reads) reduces this overhead. Profiles that show a large fraction of time in the kernel are often a sign of tiny I/O operations.

---

## 3.3 Programs and Processes

### Definitions

A **program** is a passive file on disk containing machine instructions and data (an executable, or a script plus the interpreter that runs it). A **process** is a running instance of a program: the program's code plus everything needed to execute it, namely a private virtual address space, one or more threads of execution, open files and sockets, and the kernel's bookkeeping about it.

One program can have many simultaneous processes. If ten users run `ls` at the same moment, there is one `ls` file and ten `ls` processes, each with its own memory and state.

### The process address space

Chapter 2 drew a program's memory as text (code), data, BSS, heap and stack. Here is the OS's fuller view of one process on 64-bit Linux:

```text
 0x7fff_ffff_ffff ┌──────────────────────────────┐  ← top of user space
                  │ stack (main thread)          │  grows downward
                  │        ↓                     │
                  │   (guard page: unmapped)     │
                  │                              │
                  │ memory-mapped region         │  shared libraries (libc...),
                  │   ↑↓                         │  other threads' stacks,
                  │                              │  large allocations, mapped
                  │                              │  files (Section 3.15)
                  │                              │
                  │        ↑                     │
                  │ heap (brk area)              │  grows upward
                  ├──────────────────────────────┤
                  │ BSS  (zeroed globals)        │
                  │ data (initialised globals)   │
                  │ text (machine code, r-x)     │
 0x0000_0040_0000 ├──────────────────────────────┤
                  │ (unmapped: catches null      │
                  │  pointer accesses)           │
 0x0              └──────────────────────────────┘
```

Three details matter:

- **Most of the space is empty.** A 64-bit process has 128 TiB of user address space (with the common 4-level page tables) and uses a tiny fraction of it. Only regions the process has set up are valid; touching anything else causes a fault and, normally, a `SIGSEGV` (Section 3.6).
- **Each region has permissions.** Code is readable and executable but not writable; the stack and heap are readable and writable but not executable. This blocks a class of attacks that inject code as data.
- **Addresses are randomised.** **ASLR** (address space layout randomisation) places the stack, heap and libraries at different addresses on each run, so an attacker cannot predict where code lives. The fixed `0x400000` start in the diagram is where traditional executables are loaded; most modern executables are built position-independent, so their code also lands at a random address, as in the `maps` output below.

You can see the real layout of any process:

```bash
$ cat /proc/4121/maps          # abbreviated
55d0c1a00000-55d0c1a2c000 r-xp ... /usr/bin/myserver     ← text
55d0c1c2c000-55d0c1c30000 rw-p ... /usr/bin/myserver     ← data
55d0c3011000-55d0c3232000 rw-p ... [heap]
7f3a2b400000-7f3a2b5c5000 r-xp ... /usr/lib/x86_64-linux-gnu/libc.so.6
7ffd9e1d2000-7ffd9e1f3000 rw-p ... [stack]
```

### The Process Control Block

The kernel keeps one record per process describing everything it needs to manage it. This is the **Process Control Block (PCB)**. In Linux, the corresponding structure is called `task_struct`, and Linux keeps one per *thread* (Section 3.7 explains why).

| PCB field group | Contents | Used for |
|---|---|---|
| Identity | PID, parent PID, process group and session IDs | Naming, signals, `wait` |
| State | Running, ready, blocked, zombie... (Section 3.4) | Scheduling |
| Saved CPU context | Program counter, stack pointer, general-purpose registers, flags, floating-point/SIMD state (saved when not running) | Resuming after a context switch (Section 3.11) |
| Scheduling information | Priority, nice value, accumulated CPU time, which CPU it last ran on, run-queue links | The scheduler (Section 3.10) |
| Memory management | Pointer to the page tables and the list of memory regions (the `maps` above) | Address translation (Section 3.14) |
| Open files | Pointer to the file descriptor table | All I/O (Chapter 4) |
| Credentials | User ID, group IDs, capabilities | Permission checks |
| Signals | Pending signals, blocked-signal mask, handler table | Signal delivery (Section 3.6) |
| Accounting and limits | CPU time used, page fault counts, context-switch counts, resource limits | `top`, `ps`, limits (Chapter 4) |
| Relationships | Parent, children, threads in the same group | Process tree, reaping |

Everything that `ps`, `top` and `/proc/<pid>/` show you is read from these structures.

### Process IDs

A **PID** (process ID) is an integer that names a process. The kernel allocates PIDs incrementally and wraps around at a configurable maximum (`/proc/sys/kernel/pid_max`), reusing IDs of processes that have fully exited. Two facts follow:

- **A PID is only meaningful while the process exists.** After a process exits and is reaped (Section 3.5), its PID can be given to a new, unrelated process. A script that stores a PID in a file and later runs `kill $(cat app.pid)` may kill the wrong process if the original died long ago. Linux offers **pidfds** (file descriptors that refer to one specific process) to avoid this race.
- **PID 1 is special.** It is the first user-space process the kernel starts at boot (`init`, usually `systemd` on modern distributions). It is the ancestor of every other process and has duties described in Section 3.5.

```bash
$ ps -eo pid,ppid,user,stat,comm --forest | head
    PID    PPID USER     STAT COMMAND
      1       0 root     Ss   systemd
    812       1 root     Ss   sshd
   3001     812 root     Ss    \_ sshd
   3010    3001 alice    Ss        \_ bash
   4121    3010 alice    Sl+           \_ myserver
```

### Guarantees and non-guarantees

A process is guaranteed a private address space that no other process can read or write directly, and its own identity and resources until it exits. It is **not** guaranteed that its PID stays unique over time, that it will get CPU when it wants it, or that its memory will stay in RAM.

### Backend connection

Isolation is why a crash in your web application cannot corrupt the database's memory on the same machine, and why the two must talk through the kernel (sockets, pipes, shared memory, covered in Chapter 4). Process boundaries are also failure boundaries (Chapter 1, Section 1.7): one process can be killed and restarted while others continue.

---

## 3.4 The Process Lifecycle and States

### Definition

A process (more precisely, each thread; see Section 3.7) is always in exactly one **scheduling state**. The classic five-state model is: **new** (being created), **ready** (able to run, waiting for a CPU), **running** (executing on a CPU), **blocked** or **waiting** (cannot run until some event happens), and **terminated** (finished, waiting to be cleaned up).

### Why it exists

The scheduler must know which processes it can usefully give a CPU to. A process waiting for a network packet would waste its time slice doing nothing, so it is taken out of consideration until the packet arrives. The states are the scheduler's filing system.

### Mechanism: the transition diagram

```text
                 admitted                     dispatched by scheduler
  ┌──────┐   (creation done)   ┌─────────┐ ─────────────────────────► ┌───────────┐
  │ NEW  │ ──────────────────► │  READY  │                            │  RUNNING  │
  └──────┘                     │(run     │ ◄───────────────────────── │ (on a CPU)│
                               │ queue)  │   preempted: time slice    └───────────┘
                               └─────────┘   expired, or a higher-       │     │
                                    ▲        priority task woke up       │     │
                                    │                                    │     │ exit() called,
                                    │ event occurred:                    │     │ or killed
                                    │ I/O completed, data arrived,       │     │ by a signal
                                    │ timer expired, lock released       │     ▼
                               ┌─────────┐                               │ ┌────────────┐
                               │ BLOCKED │ ◄─────────────────────────────┘ │ TERMINATED │
                               │(waiting)│   blocking request: read on     │ (zombie    │
                               └─────────┘   empty socket, disk read,      │ until      │
                                             sleep, wait for child, lock   │ reaped)    │
                                                                           └────────────┘
```

Read the arrows carefully, because interviewers ask about them:

- **Running → Ready** happens only through **preemption**: the kernel took the CPU away (Section 3.10). The process was willing to keep running.
- **Running → Blocked** happens because the process *asked* for something that is not available yet. It always passes through a system call (or a page fault that needs disk I/O, Section 3.15).
- **Blocked → Ready**, not Blocked → Running. When the awaited event happens (an interrupt from the network card, for instance), the process becomes *eligible* to run. It still has to wait for a CPU.
- **Ready → Blocked** is impossible. A process can only block by executing something, which requires running.

Some textbooks add **suspended ready** and **suspended blocked** states for processes whose memory has been entirely swapped out to disk. Modern general-purpose systems page individual pieces of memory rather than whole processes (Section 3.16), so these states matter mostly in exam questions.

### Concrete: the same states in Linux

`ps` and `top` show a one-letter state column:

| Letter | Linux meaning | Classic state |
|---|---|---|
| `R` | Running *or* runnable (on a run queue) | Running or Ready: Linux does not distinguish them in this column |
| `S` | Interruptible sleep: waiting for an event; a signal can wake it | Blocked |
| `D` | Uninterruptible sleep: waiting, usually for disk I/O, and signals are deferred until it wakes | Blocked |
| `T` | Stopped (by `SIGSTOP` or a debugger) | (Suspended) |
| `Z` | Zombie: exited, not yet reaped by its parent | Terminated |
| `I` | Idle kernel thread | (Kernel bookkeeping) |

A server handling one request per thread spends most of its life in `S`: its threads are blocked waiting for the next request or for the database to reply. Many threads in `D` usually mean the storage is slow. `D` matters for diagnosis because a process in `D` usually cannot be killed, even with `SIGKILL`, until the I/O completes. (Linux has a "killable" variant of this sleep that some kernel code paths use, but you cannot count on it.) (Signals are Section 3.6.)

### Guarantees and non-guarantees

The scheduler guarantees that a blocked process consumes no CPU time while it waits and becomes ready when its event arrives. It does **not** guarantee *when* a ready process actually runs; with more ready processes than cores, some wait.

### Failure and diagnosis

- **Many processes in `R` with high CPU:** CPU saturation. The run queue is long (Section 3.12).
- **Many in `D`:** storage or network-filesystem stalls. They count toward load average without using CPU (Section 3.12).
- **Many in `Z`:** a parent that does not reap its children (next section).

```bash
$ ps -eo stat= | cut -c1 | sort | uniq -c
    212 S
      3 R
     41 D          ← worth investigating
```

---

## 3.5 Process Creation: fork, exec, wait, and Exit

### Definitions

On Unix-like systems, a new process is created by **`fork`**, a system call that creates a child process as a near-identical copy of the calling (parent) process. **`exec`** (in Linux, `execve`) replaces the program running in the current process with a new one, keeping the same PID. **`wait`** (and `waitpid`) lets a parent block until a child exits and collect its **exit status**, the small integer the child reported when it finished.

### Why two calls instead of one?

Separating "make a new process" from "load a new program" looks odd at first, but it gives the parent a window *between* the two in which the child can configure itself: redirect its standard output to a file, close file descriptors, change user, set resource limits, join a new group. The shell uses this window for every redirection (`ls > out.txt`). A single "spawn" call would need a parameter for every possible tweak. (Windows took that route with `CreateProcess`; POSIX also offers `posix_spawn`, which bundles common cases into one call.)

### Mechanism: fork

`fork` returns *twice*: once in the parent and once in the child. The return value tells each which one it is.

```pseudo
pid = fork()
if pid < 0:
    error("fork failed")          // e.g. process limit reached
else if pid == 0:
    // CHILD: a copy of the parent, running the same code
    exec("/usr/bin/ls", ["ls", "-l"])
    error("exec failed")          // exec only returns on failure
else:
    // PARENT: pid is the child's PID
    status = waitpid(pid)
    print("child exited with", exit_code(status))
```

The child gets a new PID, and **logically** a complete copy of the parent's address space, open file descriptors, signal handlers, working directory and environment. It does not inherit the parent's pending signals or its resource-usage counters, and only the thread that called `fork` exists in the child. (Forking a multi-threaded program is therefore hazardous: if another thread held a lock at the moment of the fork, the lock stays held forever in the child. Chapter 5 explains locks.)

### Copy-on-write

Copying a 2 GB address space byte by byte on every `fork` would be ruinously slow, especially when the child calls `exec` microseconds later and throws the copy away. Linux therefore uses **copy-on-write (COW)**: instead of copying memory, `fork` copies only the *page tables* (the maps from virtual to physical memory, Section 3.14) so both processes point at the same physical pages, and marks every writable page read-only in both. Nothing is copied until someone writes.

```text
 (1) Right after fork(): page tables copied, physical pages shared, all read-only

     parent: page A ──┐                  ┌── child: page A
                      ├──► frame 10 ◄────┤
     parent: page B ──┼──► frame 11 ◄────┼── child: page B
                      │   (read-only     │
                      │    in both)      │

 (2) After the child writes to page B: only that one page is copied

     parent: page A ─────► frame 10 ◄───────── child: page A    (still shared, read-only)
     parent: page B ─────► frame 11            child: page B ──► frame 42 (new copy of 11)
                           (writable)                            (writable)
```

Step by step, when the child writes to page B:

1. The CPU sees that B is marked read-only and raises a page fault (Section 3.15).
2. The kernel checks its records: the page is not really read-only, it is shared copy-on-write.
3. It allocates a new physical frame (42), copies the 4 KB of frame 11 into it, points the child's entry for B at frame 42, and marks it writable. The parent's entry stays read-only for now. If the parent later writes to B, it faults too, but the kernel sees that the parent is by then the only user of frame 11 and simply makes the entry writable, without a copy.
4. It returns from the fault, and the CPU retries the write, which now succeeds.

The cost of `fork` is therefore proportional to the size of the page tables, not the memory, and each later write to a shared page costs one fault plus one page copy.

### Concrete: the fork–exec–wait timeline

What a shell does when you type `ls`:

```text
 time  shell (PID 3010)                         child
 ───── ──────────────────────────────────────── ───────────────────────────────────
  t0   reads "ls" from the terminal
  t1   fork()  ─────────────────────────────►   PID 3055 created; COW copy of bash
  t2   waitpid(3055)  → BLOCKED                 sets up redirections, if any
  t3                                            execve("/usr/bin/ls", ...)
                                                 - old memory mappings discarded
                                                 - ls code and data mapped in
                                                 - same PID 3055, same open fds
  t4                                            runs ls, prints output
  t5                                            exit(0) → becomes a ZOMBIE
                                                kernel sends SIGCHLD to parent
  t6   wakes: waitpid returns status 0  ◄─────  zombie entry freed (REAPED)
  t7   prints the next prompt
```

`strace -f` shows the same sequence (abbreviated; on Linux, `fork` is implemented with the more general `clone` system call):

```text
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, ...) = 3055
[pid  3055] execve("/usr/bin/ls", ["ls"], 0x7ffc... /* 22 vars */) = 0
[pid  3055] ... ls does its work ...
[pid  3055] exit_group(0)               = ?
[pid  3055] +++ exited with 0 +++
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WSTOPPED, NULL) = 3055
```

`exec` replaces code, data, heap and stacks, and resets signal handlers to their defaults. It keeps the PID, the parent, and open file descriptors (except those marked close-on-exec, Chapter 4).

### Exit status

A process ends by calling `exit(code)` (or returning from its main function), or by being killed by a signal. The **exit status** is an integer from 0 to 255. By convention, 0 means success and anything else means failure; the meaning of particular non-zero values is defined by each program. When a process is killed by signal *N*, shells and container runtimes conventionally report **128 + N**. So **137** means killed by signal 9 (`SIGKILL`), which is the usual signature of the out-of-memory killer (Section 3.17), and **143** means killed by signal 15 (`SIGTERM`).

### Zombie processes

When a process exits, the kernel frees almost everything it owned: memory, open files, sockets. It keeps one small thing: the PCB entry holding the PID and exit status, because the parent may still want to read them with `wait`. A process in this state, exited but not yet waited for, is a **zombie** (`Z` in `ps`, often shown as `<defunct>`). The parent collecting the status is called **reaping**.

A zombie uses no CPU and no memory beyond that entry. The problem is accumulation: each zombie holds a PID, and PIDs are finite. A parent that creates children in a loop and never calls `wait` eventually exhausts the PID space (or a per-user or per-container process limit), at which point `fork` fails for everyone in that scope.

```text
 time   parent (PID 500)                  child (PID 501)
 ────── ───────────────────────────────── ─────────────────────────────
  t0    fork() → 501
  t1    busy with other work, never       runs
        calls wait()
  t2                                      exit(0) → state Z
                                          SIGCHLD sent to 500, ignored
  t3    ...                               still Z: PID 501 still taken
  ...   (repeat 30,000 times)
  tN    fork() fails: "Resource temporarily unavailable"
```

**You cannot kill a zombie**: it is already dead. The fixes are to make the parent reap (call `wait` in a `SIGCHLD` handler or a loop), or to kill the *parent*, whose zombies are then adopted and reaped by PID 1.

### Orphan processes

An **orphan** is a process whose parent exited while the orphan was still running. The kernel immediately **re-parents** it to PID 1 (or to a designated "subreaper" process, a Linux feature that service managers use). PID 1 is expected to call `wait` for any child it adopts, so orphans do not become permanent zombies when they eventually exit. Orphans themselves are not a problem; daemons are sometimes created by deliberately orphaning a process.

| | Zombie | Orphan |
|---|---|---|
| Is it running? | No: it has exited | Yes |
| Whose fault? | The parent did not call `wait` | The parent exited first |
| Who cleans up? | The parent, or PID 1 after the parent dies | It is adopted by PID 1, which reaps it when it exits |
| Harmful? | In large numbers: exhausts PIDs | Not by itself |

### PID 1 in containers

Containers (Chapter 4) give an application its own PID numbering, in which the application itself is often PID 1. That process inherits PID 1's two duties without knowing it:

1. **Reaping adopted orphans.** If your application starts child processes (a shell script that launches helpers, for example) and they in turn spawn grandchildren that get orphaned, your application becomes their parent. If it never calls `wait`, zombies accumulate inside the container.
2. **Signals.** The kernel does not apply the *default* action of a signal to a PID 1 process that has not installed a handler for it. A `SIGTERM` sent to an application running as PID 1 with no `SIGTERM` handler is simply ignored, and the container is later killed forcefully when the platform's grace period expires.

The common remedy is a tiny init process (for example `tini`, or `docker run --init`) that runs as PID 1, forwards signals to the application and reaps zombies.

### Guarantees and non-guarantees

`fork` guarantees the child starts with a logically identical copy of the parent's memory; COW guarantees that writes by one are invisible to the other. `fork` does **not** guarantee which of parent and child runs first, nor that it will succeed (process limits, memory limits). A fork of a large process is cheap at the moment of the call but is **not** free afterwards: every page either side modifies is copied.

### Failure and diagnosis

| Symptom | Cause | Evidence |
|---|---|---|
| `fork: Resource temporarily unavailable`, or a runtime error that it cannot create a process or thread | Process/thread limit or PID exhaustion; often zombies or a thread leak | `ps -eo stat \| grep -c Z`; `ps -eLf \| wc -l`; `/proc/sys/kernel/pid_max`; per-user limit `ulimit -u` |
| Memory rises sharply while a process snapshots itself using fork | COW copies under heavy writes | RSS of parent and child during the snapshot |
| Container takes the full grace period to stop | PID 1 ignores `SIGTERM` | Exit code 137 instead of 0 or 143; add an init or a handler |

### Backend connection

Two backend systems lean directly on fork and COW. **PostgreSQL** forks a new backend process for each client connection (Section 3.9). **Redis** saves a snapshot to disk by forking: the child writes the in-memory dataset as it was at the moment of the fork, while the parent keeps serving writes. Thanks to COW, the snapshot is consistent and the fork is fast, but every page the parent modifies during the save is duplicated. Under a write-heavy load, memory use can approach double the dataset size during a snapshot, which is why Redis's documentation discusses memory overcommit settings (Section 3.17).

---

## 3.6 Signals

### Definition

A **signal** is a small, asynchronous notification sent to a process by the kernel or by another process, identified only by a number (and a name such as `SIGTERM`). When a signal arrives, the process either runs a **handler** function it registered, ignores the signal, or suffers the signal's **default action** (usually terminate, sometimes terminate and write a core dump, stop or continue).

### Why it exists

Processes need a way to be told about events that do not arrive through normal input: "please shut down", "your child exited", "you accessed invalid memory", "the user pressed Ctrl+C". Signals are the oldest, simplest such channel on Unix.

### The signals a backend engineer must know

| Signal | Number on Linux x86 | Typical sender | Default action | Can be caught or ignored? | Backend meaning |
|---|---|---|---|---|---|
| `SIGTERM` | 15 | `kill <pid>`, service managers, container platforms | Terminate | Yes | "Please shut down cleanly." The standard stop request |
| `SIGKILL` | 9 | `kill -9`, the OOM killer, platforms after a grace period | Terminate | **No** | Immediate death; no code runs |
| `SIGINT` | 2 | Ctrl+C in a terminal | Terminate | Yes | Interactive stop; servers usually treat it like `SIGTERM` |
| `SIGHUP` | 1 | Terminal closed; also sent by admins | Terminate | Yes | Historically "terminal hung up"; many daemons (e.g. Nginx) use it as "reload configuration" |
| `SIGQUIT` | 3 | Ctrl+\ | Terminate + core dump | Yes | Some runtimes catch it to print a thread dump without exiting (see the Java lens below) |
| `SIGSEGV` | 11 | The kernel, after an invalid memory access | Terminate + core dump | Yes, but rarely useful | A crash in native code |
| `SIGCHLD` | (varies by architecture) | The kernel, when a child exits | Ignore | Yes | Prompt to reap children |
| `SIGSTOP` / `SIGCONT` | (varies) | Debuggers, job control | Stop / continue | `SIGSTOP`: **No** | Pausing a process |
| `SIGPIPE` | 13 | The kernel, on writing to a closed pipe or socket | Terminate | Yes | Servers ignore it and handle the write error instead (Chapter 4) |

`SIGKILL` and `SIGSTOP` cannot be caught, blocked or ignored. This is a deliberate guarantee: an administrator must always be able to stop a misbehaving process.

### Mechanism: how a signal is delivered

1. **Generation.** A process calls `kill(pid, SIGTERM)`, or the kernel itself generates a signal (an invalid memory access, a child exiting).
2. **Pending.** The kernel sets a bit in the target's pending-signal set in its PCB. Nothing runs yet. Standard signals do not queue: if `SIGTERM` is sent twice before delivery, it is delivered once.
3. **Delivery.** The next time the target thread is about to return from kernel mode to user mode (after a syscall, an interrupt or being scheduled), the kernel checks its pending set. If a signal is pending and not blocked:
   - **Default action:** the kernel acts directly, for example terminating the process.
   - **Handler installed:** the kernel builds a special frame on the thread's user stack and arranges for the CPU to "return" into the handler function. When the handler returns, a special syscall (`sigreturn`) restores the interrupted context, and the program continues as if nothing happened.
4. **Interrupted syscalls.** If the thread was blocked in an interruptible sleep (state `S`, Section 3.4), the signal wakes it. The syscall either fails with the error `EINTR` ("interrupted system call") or is automatically restarted, depending on how the handler was installed. Threads in uninterruptible sleep (`D`) only see the signal after their I/O completes, which is why even `SIGKILL` seems to have no effect on them.

A handler runs in the middle of whatever the program was doing, possibly halfway through updating a data structure. It may therefore safely do very little: set a flag, or write a byte to a pipe that the main loop watches. The real work, such as shutting down, happens in normal code that notices the flag.

```pseudo
shutting_down = false

on_signal(SIGTERM):
    shutting_down = true          // only set a flag: nothing complicated here

main_loop:
    while not shutting_down:
        handle_next_request()
    begin_graceful_shutdown()
```

### Concrete: graceful versus forced shutdown

**Graceful shutdown** is stopping a server without failing requests it has already accepted. A typical sequence after `SIGTERM`:

```text
 t=0s    SIGTERM arrives; the handler sets a flag
 t=0s    stop accepting new connections (close the listening socket) and/or
         report "not ready" so the load balancer stops sending traffic
 t=0–20s finish in-flight requests, up to a deadline
 t=20s   close database connections, flush buffered logs and metrics
 t=20s   exit(0)
 ─────── if still running when the platform's grace period expires: SIGKILL ───────
```

Container orchestrators follow the same pattern: they send `SIGTERM`, wait for a grace period (30 seconds by default in Kubernetes at the time of writing), then send `SIGKILL`. If the application ignores `SIGTERM` (or runs as PID 1 without a handler, Section 3.5), it dies by `SIGKILL` with in-flight requests cut off. The full protocol, including how load balancers drain traffic, is in Chapter 24 (graceful shutdown) and Chapter 26 (deployments).

> **Java lens:** The JVM installs handlers for `SIGTERM`, `SIGINT` and `SIGHUP` and runs registered *shutdown hooks* when it receives them. A shutdown hook running is not the same as in-flight HTTP requests finishing. In Spring Boot, draining in-flight requests is configured separately (`server.shutdown=graceful`). The JVM also catches `SIGQUIT`: `kill -QUIT <pid>` prints the stacks of all its threads without stopping the process, a standard debugging tool.
>
> **Other runtimes:** Node.js does nothing special on `SIGTERM` unless you register `process.on('SIGTERM', ...)`; by default the process exits. Go delivers signals to a channel you subscribe to with `signal.Notify`.

### Guarantees and non-guarantees

The kernel guarantees `SIGKILL` and `SIGSTOP` cannot be blocked, and that a delivered signal's handler runs before the thread resumes user code. It does **not** guarantee that signals queue (two identical standard signals may merge into one), which thread of a multi-threaded process handles a process-directed signal, or that a process will finish anything after `SIGTERM`; that depends entirely on the application.

### Failure and diagnosis

- **Exit code 143 (128 + 15):** stopped by `SIGTERM` and the default action ran: no graceful shutdown logic.
- **Exit code 137 (128 + 9):** killed by `SIGKILL`: either the grace period expired or the OOM killer struck (Section 3.17). Check `dmesg` and the orchestrator's events to tell which.
- **`EINTR` errors in logs:** a signal interrupted a blocking call and the code did not retry it.
- **Requests failing with connection resets during every deploy:** the server exits immediately on `SIGTERM` without draining. Compare deploy timestamps with error spikes.

### Backend connection

Every deployment, autoscaling event and node drain stops processes with signals. Whether a deploy causes errors depends on how your server handles `SIGTERM`, how long it takes to drain, and how that compares with the grace period. Signals are also a debugging tool: some runtimes print the stacks of all their threads when sent `SIGQUIT`.

---

## 3.7 Threads

### Definition

A **thread** is an independent sequence of execution inside a process: its own program counter, registers and stack, running code that shares the process's address space and resources with the process's other threads. The process is the container of resources; the thread is the unit the scheduler runs on a CPU.

### Why threads exist

Suppose a server process handles one client at a time. While it waits 5 ms for a database reply, the CPU sits idle and every other client waits. You could fork a process per client, but processes are isolated: sharing an in-memory cache or a counter between them requires the explicit inter-process communication of Chapter 4, and every process carries its own page tables and memory. Threads give you several flows of execution that see the same memory directly, so one can block on the database while another runs, and both can use the same data structures.

### Mechanism: what a thread shares and what it owns

```text
┌─────────────────────────────────────────────────────────────────────┐
│ PROCESS (PID 4121)                                                  │
│                                                                     │
│  SHARED by all threads                                              │
│   - address space: code, globals, heap, memory-mapped files         │
│   - open file descriptors (files, sockets)                          │
│   - signal handlers, current directory, user ID, resource limits    │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │
│  │ THREAD 4121   │  │ THREAD 4122   │  │ THREAD 4123   │            │
│  │ (main)        │  │ (worker)      │  │ (worker)      │            │
│  │ - TID         │  │ - TID         │  │ - TID         │            │
│  │ - registers,  │  │ - registers,  │  │ - registers,  │            │
│  │   PC, SP      │  │   PC, SP      │  │   PC, SP      │            │
│  │ - stack       │  │ - stack       │  │ - stack       │            │
│  │ - TLS         │  │ - TLS         │  │ - TLS         │            │
│  │ - sched state │  │ - sched state │  │ - sched state │            │
│  │ - signal mask │  │ - signal mask │  │ - signal mask │            │
│  └───────────────┘  └───────────────┘  └───────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

| Owned by the **process** (shared by its threads) | Owned by each **thread** |
|---|---|
| Virtual address space and page tables | Thread ID (TID) |
| Code, global variables, heap | Program counter, stack pointer, other registers |
| File descriptor table | Its own stack (inside the shared address space) |
| Signal handlers | Thread-local storage |
| Credentials (user, group) | Signal mask (which signals it blocks) |
| Current directory, resource limits | Scheduling state (R, S, D...), priority, CPU time |
| PID | The `errno` error variable (thread-local in modern C libraries) |

**Thread-local storage (TLS)** is memory that each thread has its own copy of, reached through the same variable name. On x86-64 Linux the CPU's `FS` register points to the current thread's TLS block, and it is saved and restored on every context switch. Runtimes use TLS for per-thread state such as the current request's context, a per-thread allocation buffer, or the "current transaction". (TLS here is unrelated to the Transport Layer Security protocol of Chapter 8; the acronym is shared by accident.)

Note that a thread's stack is *private by convention, not by protection*. It lives in the shared address space, so a bug that passes a pointer to a stack variable to another thread lets that thread read or corrupt it.

### How Linux implements threads

Linux does not have a separate "thread" object. It has **tasks**, each with its own `task_struct` (the PCB of Section 3.3). Both `fork` and thread creation are implemented by the `clone` system call, with flags that say what the new task shares with its creator:

```text
 fork()           ≈ clone(SIGCHLD)                    share nothing; COW copy of memory
 thread creation  ≈ clone(CLONE_VM | CLONE_FILES |    share address space, fd table,
                          CLONE_SIGHAND | CLONE_THREAD  signal handlers; same thread
                          | ...)                        group (same PID)
```

A "process" in Linux is therefore a **thread group**: tasks that share an address space and a thread-group ID, which is what `getpid()` returns. Each task also has its own TID. You can see this directly:

```bash
$ ps -o pid,tid,stat,comm -L -p 4121       # -L: one line per thread
    PID     TID STAT COMMAND
   4121    4121 Sl   myserver
   4121    4122 Sl   myserver
   4121    4123 Sl   myserver
$ ls /proc/4121/task
4121  4122  4123
$ grep Threads /proc/4121/status
Threads:	3
```

Because the scheduler schedules tasks, it schedules **threads**, not processes. A process with 50 runnable threads competes for CPU as 50 entities.

### Why threads are cheaper than processes

| Operation | New process (`fork`) | New thread |
|---|---|---|
| Kernel structure | New `task_struct` | New `task_struct` |
| Page tables | Copied (proportional to mapped memory) | None: shares the existing ones |
| Later memory writes | COW faults and page copies | None: memory really is shared |
| File descriptor table | Copied | Shared |
| Context switch between two of them | Must switch address space (Section 3.11) | Same address space: cheaper |
| Communication | Pipes, sockets, explicit shared memory (Chapter 4) | Ordinary reads and writes of shared variables |

A thread still costs real resources. Each one needs a kernel stack (16 KB on x86-64 Linux at the time of writing), a `task_struct`, and a user stack. The user stack is a *reservation* of virtual address space, commonly 8 MB with the GNU C library, taken from the stack resource limit, and physical memory is committed only for the pages actually touched (Section 3.15). A thread that recurses deeply or allocates large arrays on its stack does consume that memory.

### Why threads are riskier than processes

The sharing that makes threads cheap also makes them dangerous:

1. **Data races.** Two threads that read and modify the same variable at the same time can corrupt it. Two threads each executing `count = count + 1` a million times often end with a total below two million, because each increment is a read, an add and a write, and the steps interleave. Chapter 5 explains why and how locks and atomic operations fix it.
2. **No fault isolation.** A crash in one thread (an invalid memory access in native code, for instance) delivers `SIGSEGV` to the process and kills *all* its threads, including the ones serving other users' requests.
3. **No memory isolation.** A thread that corrupts the heap corrupts it for everyone. A memory leak in one code path grows the memory of the whole process.
4. **Harder reasoning.** The program's behaviour depends on how the scheduler interleaves threads, which changes from run to run, so bugs appear and disappear.

### Guarantees and non-guarantees

Threads of one process are guaranteed to see the same memory and file descriptors. They are **not** guaranteed to see each other's writes *immediately* or *in order* without synchronisation: CPU caches and compiler optimisations can delay or reorder them (the memory model, Chapter 5). The OS guarantees nothing about which thread runs first or how their steps interleave.

### Failure and diagnosis

| Symptom | Likely cause | Evidence |
|---|---|---|
| Thread creation fails (`pthread_create` returns `EAGAIN`; runtimes surface this as their own error, often worded as being unable to create a native thread) | Per-user or per-container task limit reached, or memory for stacks exhausted | `Threads:` in `/proc/<pid>/status`; `ulimit -u`; container pid limits (Chapter 4) |
| Process memory grows with the number of threads | Stacks and per-thread runtime buffers | Count threads over time; per-thread stack usage |
| Thread count grows without bound | Thread leak: a thread created per task and never ended | Thread count trend; thread dump showing thousands of identical idle threads |

### Backend connection

The classic server model gives each in-flight request its own thread (Section 3.9 and Chapter 11). It is simple to write, because each request is straight-line code that blocks when it waits. Its costs are exactly the costs above: memory per thread, context switches between threads (Section 3.11), and shared-state bugs that Chapter 5 teaches you to prevent.

---

## 3.8 Threading Models

### Definitions

A **threading model** describes how the threads a program creates (in its language or runtime) map onto the threads the kernel schedules.

- **Kernel-level threads** are threads the kernel knows about, schedules and can run on any core. Every thread is a kernel task. This is the **1:1 model**: one program thread per kernel thread.
- **User-level threads** (historically called **green threads**) are threads implemented entirely by a library or runtime in user space. The kernel sees only one thread; the runtime switches between user-level threads itself. This is the **N:1 model**: many program threads on one kernel thread.
- The **M:N model** (hybrid) multiplexes M user-level threads onto N kernel threads, where N is usually about the number of cores.
- A **coroutine** is a function that can *suspend* itself at defined points and later *resume* where it left off, without involving the kernel. Coroutines are the building block of user-level threading and of `async`/`await` code.

### Why more than one model exists

Kernel threads are general and simple, but each costs memory and a kernel-mediated context switch. A server that wants 100,000 concurrent connections cannot comfortably afford 100,000 kernel threads with megabyte stack reservations. User-level threads are much cheaper to create and switch (a switch is a few register saves inside one process, no system call), but the kernel does not know they exist, which causes the problems below. Each model is a different answer to "who does the scheduling, and how much does a thread cost?"

### Mechanism and comparison

```text
  1:1 (kernel threads)        N:1 (green threads)          M:N (hybrid)

  program threads             program threads              program threads
   T1   T2   T3                T1   T2   T3                 T1  T2  T3  T4  T5  T6
   │    │    │                  \   │   /                    \  │   /    \  │  /
   │    │    │                   \  │  /                    runtime scheduler
   │    │    │                 runtime scheduler              /           \
   K1   K2   K3                      K1                      K1            K2
   (kernel schedules all)     (kernel sees one)            (kernel sees N)
```

| | 1:1 kernel threads | N:1 green threads | M:N |
|---|---|---|---|
| Who schedules | Kernel | Runtime, inside one kernel thread | Runtime onto kernel threads; kernel schedules those |
| Uses multiple cores | Yes | **No**: one kernel thread runs on one core at a time | Yes |
| Switch cost | Kernel context switch | Very cheap (user-space register swap) | Cheap between user threads |
| One thread makes a blocking syscall | Only that thread blocks | **The whole process blocks**: the kernel put its only thread to sleep | The runtime must detect it and move other user threads to another kernel thread |
| Memory per thread | Kernel stack + large user stack reservation | Small | Small, often growable |
| Complexity | In the kernel, already done | Moderate | High: two schedulers that must cooperate |
| Examples | Linux native threads (NPTL), JVM platform threads | Early Java "green threads" (1990s) | Go goroutines; Java virtual threads; Erlang processes |

N:1's fatal flaw is the blocking syscall: if any green thread calls `read` on a socket with no data, the kernel blocks the single kernel thread, and every other green thread freezes with it. Early user-level thread libraries worked around this by making every I/O call non-blocking internally. Once Linux's 1:1 implementation (NPTL, in the early 2000s) made kernel threads fast enough, 1:1 became the default for C and Java.

M:N has returned in modern runtimes because they control *all* I/O. In Go and in Java virtual threads, when a lightweight thread performs blocking I/O, the runtime does not let the kernel thread block. Instead it registers interest with the kernel's readiness mechanism (Chapter 4), parks the lightweight thread, and runs another one on the same kernel thread. When the data arrives, the parked thread is resumed, possibly on a different kernel thread. To the programmer, the code looks like ordinary blocking code; to the kernel, a few threads are busy.

### Coroutines

A coroutine suspends voluntarily, which makes it **cooperative**: it runs until it reaches a suspension point (an `await`, a channel operation, a blocking call the runtime intercepts) and only then can another coroutine run on that kernel thread.

```pseudo
async function handle(request):
    user  = await db.query("SELECT ...")      // suspend here; other coroutines run
    items = await http.get(inventory_url)     // suspend here too
    return render(user, items)                // runs straight through: no suspension
```

There are two families:

- **Stackful** coroutines (goroutines, Java virtual threads) have their own stack and can suspend from anywhere, even deep inside nested calls. The runtime saves and restores the stack contents or pointers.
- **Stackless** coroutines (`async`/`await` in JavaScript, Python, C#, Rust) are compiled into a state machine that remembers which `await` it stopped at. They can only suspend at an explicit `await`, which is why `async` "spreads" through a codebase: a function that awaits must itself be async.

The engine that resumes coroutines when their I/O completes is an **event loop**, recalled from Chapter 2 as Node.js's execution model; its mechanism is Chapter 5.

### Guarantees and non-guarantees

Lightweight threads and coroutines make **waiting** cheap: thousands of tasks can each wait for a database reply at the cost of a few kilobytes apiece instead of a kernel thread each. They do **not** add CPU cores, database connections or downstream capacity. If 10,000 virtual threads each need one of 20 database connections, 9,980 of them wait in line. And because they are cooperative at heart, a lightweight thread that computes for a long time without reaching a suspension point can hold its kernel thread, delaying the others scheduled on it.

### Failure and diagnosis

- **A CPU-heavy task freezes an event loop or a carrier thread.** Every other coroutine on that thread stalls. The symptom is a latency spike across unrelated requests that matches the duration of the heavy task. Profilers that show "time between event-loop iterations" (event-loop lag) confirm it.
- **A call that blocks the kernel thread** (a native library doing blocking I/O that the runtime cannot intercept) behaves like the N:1 problem. Thread dumps showing all carrier threads stuck in the same native call confirm it.

> **Java lens:** Since Java 21, Java has two kinds of threads. *Platform threads* are 1:1 wrappers around kernel threads. *Virtual threads* are M:N: the JVM mounts them on a small pool of *carrier* platform threads and unmounts them when they block on supported I/O. Chapter 11 discusses when each fits a server.
>
> **Other runtimes:** Go's goroutines start with a small stack (a few kilobytes) that the runtime grows as needed, and are scheduled M:N across a number of kernel threads controlled by `GOMAXPROCS`. Python's `asyncio` and Node.js use stackless coroutines on a single-threaded event loop.

### Backend connection

The threading model determines how a server handles waiting, which is most of what a backend does. Chapter 5 builds the concurrency primitives, and Chapter 11 compares thread-per-request, event-loop and virtual-thread servers under load.

---

## 3.9 Multi-Process versus Multi-Threaded Servers

### Definition

A **multi-process server** handles concurrent work in several processes, typically one per connection or a fixed set of worker processes. A **multi-threaded server** handles it in several threads inside one process. Many real servers combine the two: a few processes, each with many threads or an event loop.

### Why the choice matters

The choice decides the server's isolation (does one crash take everything down?), its memory cost per connection, how it shares data between connections, and how many connections it can hold. You will meet all of these models in production, often on the same machine.

### Mechanism: how well-known servers do it

```text
 PostgreSQL: process per connection
 ┌───────────────┐   accept()   fork()   ┌──────────────────┐
 │ postmaster    │ ───────────────────►  │ backend process  │ ◄── client A
 │ (listens on   │                       └──────────────────┘
 │  port 5432)   │ ───────────────────►  ┌──────────────────┐
 └───────┬───────┘                       │ backend process  │ ◄── client B
         │                               └──────────────────┘
         └── shared memory segment: shared buffer cache, lock tables,
             visible to all backends (Chapter 15)

 MySQL: thread per connection inside one server process
 ┌────────────────────────────────────────────┐
 │ mysqld process                             │
 │   thread ◄── client A   thread ◄── client B│
 │   shared: buffer pool, caches              │
 └────────────────────────────────────────────┘

 Nginx: master + a few single-threaded worker processes, each running an event loop
 ┌──────────┐     ┌──────────────────────────┐
 │ master   │ ──► │ worker 1: event loop     │ ◄── thousands of connections
 │ (config, │     ├──────────────────────────┤
 │  signals)│ ──► │ worker 2: event loop     │ ◄── thousands of connections
 └──────────┘     └──────────────────────────┘     (typically one worker per core)
```

| Server | Model | Why |
|---|---|---|
| **PostgreSQL** | One OS process per client connection, forked by the `postmaster`; shared state lives in an explicit shared-memory segment | Robustness and simplicity: a crashing backend does not corrupt the others' private memory (the postmaster then resets all backends as a precaution if shared memory might be damaged); a design that predates reliable threads on all target platforms |
| **MySQL** | One thread per connection in a single server process by default; thread-pool alternatives exist in some editions and forks | Cheaper connections; shared caches are ordinary shared memory |
| **Nginx** | One master plus several single-threaded worker processes, each multiplexing many connections with an event loop (Chapter 4) | Uses all cores without threads; isolation between workers |
| **Apache httpd** | Configurable: `prefork` (process per connection), `worker` and `event` (processes with threads) | Historical flexibility |
| **Python web servers** (e.g. Gunicorn) | A pre-forked pool of worker processes, each with one or a few threads or an event loop | The GIL (Chapter 2) prevents CPU parallelism across threads in CPython |
| **Redis** | Commands executed by one main thread; optional extra threads for network I/O in recent versions | Simplicity: no locks around data; each command is fast |
| **JVM application servers** | One process with a pool of threads (or virtual threads) | Shared heap, JIT-compiled code and caches across all requests |

**Pre-forking** is worth naming: rather than fork on every connection, the server forks a fixed set of workers at startup and hands connections to them. It keeps process isolation while avoiding a fork per request.

### Trade-offs

| | Multi-process | Multi-threaded |
|---|---|---|
| Crash isolation | Strong: one worker crash leaves others running | Weak: one crash kills all threads |
| Memory per unit | Higher: own page tables, own private caches, own copy of anything not explicitly shared | Lower: everything shared |
| Sharing data | Explicit (shared memory, pipes, sockets) | Implicit (any variable); needs synchronisation (Chapter 5) |
| Creation cost | `fork` (cheap with COW, but not free) | Thread creation (cheaper) |
| Switch cost | Address-space switch | Same address space |
| Fits | Languages with a GIL; untrusted or crash-prone code; strong isolation | Shared in-memory state; runtimes with efficient threads |

### Concrete: why PostgreSQL connections are "expensive"

Assume each PostgreSQL backend process uses about 10 MB of private memory for a typical session (its catalog caches, query memory, and so on; the real figure varies widely with workload and settings). Then:

- 100 connections ≈ 1 GB of backend memory, on top of the shared buffer cache.
- 2,000 connections ≈ 20 GB, plus 2,000 processes for the scheduler to manage, most of them idle.

This is why a PostgreSQL server rejects connections beyond its configured `max_connections` with the error `FATAL:  sorry, too many clients already`, and why applications put a **connection pool** between themselves and the database (Chapter 18) instead of opening a connection per request. MySQL's equivalent limit produces error 1040, "Too many connections". The underlying limit differs (processes versus threads), but the lesson is the same: a database connection is backed by real OS resources.

### Failure and diagnosis

- **Connection refused at a fixed connection count:** the database's connection limit. Count backends with `ps -C postgres --no-headers | wc -l`, or in SQL with `SELECT count(*) FROM pg_stat_activity;`.
- **Memory pressure grows with connection count:** per-connection memory. Measure it with PSS rather than RSS (Section 3.18), because PostgreSQL processes share their buffer cache.
- **One native crash takes down a whole multi-threaded server:** expected from the model. Process-level supervision (a restart policy) is the mitigation.

### Backend connection

When you size a system, know the model of every component: how many connections your database can accept, what each costs, and whether one bad request can crash a process serving many others.

---

## 3.10 CPU Scheduling

### Definitions

The **CPU scheduler** is the part of the kernel that decides which ready thread runs on each CPU core, and for how long. The **dispatcher** is the code that then performs the switch (Section 3.11). A **scheduling algorithm** (or policy) is the rule the scheduler uses to choose.

Scheduling is **preemptive** if the kernel can take the CPU away from a running thread (on a timer interrupt or when a more important thread wakes). It is **cooperative** (non-preemptive) if a thread keeps the CPU until it blocks, exits or voluntarily yields. Linux, Windows and macOS schedule threads preemptively. Cooperative scheduling survives *inside* runtimes, in coroutines and event loops (Section 3.8).

### Why it exists

There are almost always more ready threads than cores. Someone must choose, and the choice determines how quickly an interactive request is answered, how much total work gets done, and whether some thread waits forever.

### Goals, and why they conflict

| Metric | Definition | Who cares |
|---|---|---|
| **CPU utilisation** | Fraction of time the CPU does useful work | Cost efficiency |
| **Throughput** | Jobs completed per unit time | Batch processing |
| **Turnaround time** | Completion time − arrival time | Batch jobs |
| **Waiting time** | Total time spent in the ready queue | The usual exam metric |
| **Response time** | First time on CPU − arrival time | Interactive systems, servers |
| **Fairness** | Each thread gets its share; no one starves | Multi-tenant systems |

No algorithm optimises all of them. Running short jobs first minimises average waiting but can starve long jobs. Short time slices improve response time but waste CPU on switching. Every policy below is a different compromise.

Programs alternate between **CPU bursts** (computing) and **I/O bursts** (waiting). Schedulers care about the length of the *next CPU burst*, not the length of the whole program.

### Concrete: four algorithms on one job set

Four jobs, with arrival times and the length of their next CPU burst in milliseconds:

| Job | Arrival | Burst |
|---|---|---|
| A | 0 | 7 |
| B | 1 | 4 |
| C | 2 | 1 |
| D | 3 | 4 |

Total work is 16 ms. For each job, **waiting time = turnaround time − burst = (completion − arrival) − burst**.

**1. First-Come, First-Served (FCFS).** Run jobs in arrival order, each to completion. Non-preemptive.

```text
 | A                          | B              | C  | D              |
 0                            7               11   12               16
```

| Job | Completion | Turnaround | Waiting |
|---|---|---|---|
| A | 7 | 7 | 0 |
| B | 11 | 10 | 6 |
| C | 12 | 10 | 9 |
| D | 16 | 13 | 9 |
| **Average** | | **10.0** | **6.0** |

C needs only 1 ms but waits 9 ms behind A and B. When short jobs pile up behind a long one, the result is called the **convoy effect**. FCFS is simple and fair in a "first in line" sense, and that is all.

**2. Shortest Job First (SJF), non-preemptive.** When the CPU becomes free, run the ready job with the shortest burst. At time 0 only A has arrived, so A runs to completion. At 7, B (4), C (1) and D (4) are ready: C runs, then B and D (tied on length; ties broken by arrival).

```text
 | A                          | C  | B              | D              |
 0                            7    8               12               16
```

| Job | Completion | Turnaround | Waiting |
|---|---|---|---|
| A | 7 | 7 | 0 |
| C | 8 | 6 | 5 |
| B | 12 | 11 | 7 |
| D | 16 | 13 | 9 |
| **Average** | | **9.25** | **5.25** |

**3. Shortest Remaining Time First (SRTF)**: the preemptive version of SJF. Whenever a job arrives, if its burst is shorter than the running job's *remaining* time, it preempts.

- t=0: A starts (7 remaining).
- t=1: B arrives with 4 < A's 6 remaining → B preempts A.
- t=2: C arrives with 1 < B's 3 remaining → C preempts B; C finishes at 3.
- t=3: D arrives. Remaining: A 6, B 3, D 4 → B runs and finishes at 6.
- t=6: A 6, D 4 → D runs, finishes at 10. Then A finishes at 16.

```text
 | A  | B  | C  | B           | D              | A                      |
 0    1    2    3             6               10                       16
```

| Job | Completion | Turnaround | Waiting |
|---|---|---|---|
| A | 16 | 16 | 9 |
| B | 6 | 5 | 1 |
| C | 3 | 1 | 0 |
| D | 10 | 7 | 3 |
| **Average** | | **7.25** | **3.25** |

SJF is provably optimal for average waiting time among non-preemptive schedules when all jobs are available together, and SRTF is optimal among preemptive ones. The catch: **the scheduler does not know burst lengths in advance.** Real systems can only estimate, typically with an exponential average of past bursts, `next_estimate = α × last_burst + (1 − α) × previous_estimate`. Both algorithms can also **starve** long jobs: if short jobs keep arriving, A never runs.

**4. Round Robin (RR)** with a **time quantum** (time slice) of 2 ms. Ready jobs wait in a FIFO queue; each runs for at most one quantum, then goes to the back of the queue. (Convention used here: a job arriving at the same instant as a preemption enters the queue before the preempted job.)

```text
 time   runs    queue after this slice
 0–2    A       [B, C, A]     B arrived at 1, C at 2; A re-queued last
 2–4    B       [C, A, D, B]  D arrived at 3
 4–5    C       [A, D, B]     C finishes at 5 (needed only 1)
 5–7    A       [D, B, A]
 7–9    D       [B, A, D]
 9–11   B       [A, D]        B finishes at 11
 11–13  A       [D, A]
 13–15  D       [A]           D finishes at 15
 15–16  A       []            A finishes at 16

 | A  | B  |C| A  | D  | B  | A  | D  |A|
 0    2    4 5    7    9   11   13   15 16
```

| Job | Completion | Turnaround | Waiting | Response (first run − arrival) |
|---|---|---|---|---|
| A | 16 | 16 | 9 | 0 |
| B | 11 | 10 | 6 | 1 |
| C | 5 | 3 | 2 | 2 |
| D | 15 | 12 | 8 | 4 |
| **Average** | | **10.25** | **6.25** | **1.75** |

**Comparing them:**

| Algorithm | Avg waiting | Avg response | Preemptive | Starvation possible | Needs burst lengths |
|---|---|---|---|---|---|
| FCFS | 6.0 | 6.0 | No | No | No |
| SJF | 5.25 | 5.25 | No | Yes | Yes |
| SRTF | 3.25 | 0.75 | Yes | Yes | Yes |
| RR (q = 2) | 6.25 | 1.75 | Yes | No | No |

For a non-preemptive algorithm, response time equals waiting time, because a job's only wait is before it first runs. Under SRTF, A, B and C each run the moment they arrive, and D first runs at 6, three units after arriving: (0 + 0 + 0 + 3) / 4 = 0.75.

SRTF has the best waiting and response times here, but only because this table pretends the scheduler knows every burst length in advance. Among the algorithms that need no such knowledge, Round Robin has the worst average *waiting* time here, yet it is the basis of most interactive schedulers. Why? Because it needs no knowledge of the future, never starves anyone, and gives every job a quick first response. Users and clients notice response time, not average waiting time.

**Choosing the quantum.** If the quantum is very large, RR degenerates into FCFS. If it is very small, the CPU spends its time switching. Assume a context switch costs 5 µs:

| Quantum | Overhead per slice | Fraction of CPU lost to switching |
|---|---|---|
| 10 ms | 5 µs | ≈ 0.05% |
| 1 ms | 5 µs | ≈ 0.5% |
| 50 µs | 5 µs | ≈ 9% |

(These ignore the indirect cache effects of Section 3.11, which make small quanta worse still.)

### Priority scheduling, starvation and aging

**Priority scheduling** gives each thread a priority and always runs the highest-priority ready thread (preemptively or not). SJF is a priority scheduler whose priority is "shorter burst". The danger is **starvation**: a low-priority thread that is ready but never runs because higher-priority threads keep arriving.

The standard remedy is **aging**: gradually raise the priority of a thread the longer it waits, so that eventually it outranks the newcomers and runs.

Do not confuse starvation with deadlock. In starvation, the thread *could* run and the system makes progress, just never for that thread. In **deadlock** (Chapter 5), a set of threads each waits for another in the set, so none of them can ever proceed. A related priority problem, **priority inversion**, where a high-priority thread waits for a lock held by a low-priority one, also needs locks to explain and is in Chapter 5.

### Multilevel Feedback Queue (MLFQ)

How do you favour short, interactive jobs without knowing burst lengths? **Learn them from behaviour.** A **multilevel feedback queue** keeps several ready queues at different priorities and moves threads between them based on how they use the CPU. A common formulation of the rules:

1. If priority(X) > priority(Y), X runs.
2. Threads of equal priority share the CPU round-robin, usually with longer quanta at lower priorities.
3. A new thread starts at the highest priority.
4. If a thread uses up its whole allotment of time at a level, it moves down one level. (It behaved like a long CPU-bound job.)
5. A thread that gives up the CPU before its allotment is used (because it blocked on I/O) stays at its level. (It behaved like an interactive job.)
6. Periodically, move every thread back to the top queue (a **priority boost**). This is aging: it prevents starvation and lets a thread whose behaviour changed be re-evaluated.

```text
 Q0  (highest, quantum 8 ms)   ← new threads; interactive threads stay here
 Q1  (quantum 16 ms)           ← used a full allotment once
 Q2  (lowest, quantum 32 ms)   ← CPU-bound threads sink here; run when Q0, Q1 are empty
          ↑ periodic boost: everything back to Q0
```

Rule 4 must count *total* time used at a level, not single quanta, otherwise a thread could game the scheduler by blocking just before its quantum expires. MLFQ approximates SJF without knowing the future, which is why variants of it scheduled threads in many classic systems.

### What Linux actually does

Linux's general-purpose scheduler is built on **fairness** rather than queues of fixed priorities. The **Completely Fair Scheduler (CFS)**, used from 2007, works like this:

- Each runnable thread has a **virtual runtime** (`vruntime`): the CPU time it has received, scaled by its weight. Higher-priority threads accumulate `vruntime` more slowly.
- Runnable threads are kept in a red–black tree (a balanced search tree) ordered by `vruntime`.
- The scheduler runs the thread with the **smallest** `vruntime`, the one that has had the least fair share so far, and preempts it once another thread is sufficiently "behind".
- A thread that sleeps (blocked on I/O) does not accumulate `vruntime`, so when it wakes it is near the front: interactive and I/O-bound threads get prompt service automatically.
- Weights come from the **nice value**, from −20 (highest priority) to 19 (lowest). Each step changes a thread's weight by a factor of about 1.25, so two CPU-bound threads one nice level apart split a core roughly 55% / 45%.

Linux 6.6 replaced CFS's selection logic with an algorithm called **EEVDF** (Earliest Eligible Virtual Deadline First), which keeps the fairness idea but adds a notion of per-thread deadlines to improve latency. Separately, Linux offers **real-time policies** (`SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE`) that always beat normal threads; a runaway real-time thread can starve the whole machine, which is why they require privileges.

On a multi-core machine, each core has its own run queue. The scheduler periodically **load-balances** threads between cores, but prefers to keep a thread on the core it last ran on (**processor affinity**), because that core's caches still hold its data (Chapter 2).

### Guarantees and non-guarantees

A preemptive fair scheduler guarantees that a CPU-bound thread cannot monopolise a core forever and that every runnable normal thread eventually runs. It does **not** guarantee any specific latency: with 100 runnable threads on 4 cores, each one waits for many others. Nice values are relative weights, not reservations.

### Failure and diagnosis

- **Latency-sensitive service sharing a host with batch jobs:** CPU contention. Look for a long run queue (`r` in `vmstat`, Section 3.12) and many involuntary context switches (Section 3.11). Fix: lower the batch job's priority (`nice`), separate the workloads, or give the service its own CPU allocation (cgroups, Chapter 4).
- **A thread that never progresses:** starvation, often from a real-time or busy-spinning high-priority thread.

### Backend connection

Every queue in a backend is scheduled by some policy, and the same theory applies. A server's request queue is usually FCFS, so one slow request type can create a convoy that delays fast ones. Separating workloads into different pools (bulkheads, Chapter 24) is the backend equivalent of multilevel queues. Chapter 11 applies queueing ideas to server thread pools.

---

## 3.11 The Context Switch

### Definition

A **context switch** is the kernel's act of stopping one thread on a CPU core and resuming another. It saves the running thread's CPU state (registers, program counter, stack pointer) into kernel memory, selects the next thread, and loads that thread's saved state so it continues exactly where it left off. If the two threads belong to different processes, the kernel also switches the active address space.

### Why it exists

It turns scheduling decisions into reality. Without it, a core could only run one thread from start to finish. With it, a few cores create the illusion that thousands of threads run at once.

### Mechanism: what triggers a switch

| Kind | Trigger | Example | Counted in `/proc/<pid>/status` as |
|---|---|---|---|
| **Voluntary** | The thread blocks or yields | `read` on a socket with no data; waiting for a lock; `sleep`; a disk read | `voluntary_ctxt_switches` |
| **Involuntary** | The kernel preempts the thread | Its time slice ran out (timer interrupt); a more deserving thread woke up | `nonvoluntary_ctxt_switches` |

An I/O-bound thread switches mostly voluntarily: it gives up the CPU every time it waits. A CPU-bound thread competing with others switches mostly involuntarily. The ratio between the two is a useful diagnostic.

### Mechanism: what is saved and restored

A timer-driven switch from thread X (process P) to thread Y (process Q) on one core:

```text
 Thread X running in user mode
   │
   │ 1. Timer interrupt. CPU enters kernel mode on X's kernel stack and saves
   │    X's user instruction pointer, stack pointer and flags.
   │ 2. Entry code saves X's other general-purpose registers on that stack.
   │ 3. Timer handler charges X for the CPU time it used and decides X should
   │    be preempted: it sets a "need reschedule" flag.
   │ 4. Before returning to user mode, the kernel sees the flag and calls the
   │    scheduler, which picks Y.
   │ 5. The switch itself:
   │      - save X's kernel stack pointer and remaining registers in X's task_struct
   │      - save X's floating-point/SIMD registers if in use (these can total
   │        hundreds of bytes to a few KB)
   │      - different process? load Q's page-table base address into the CPU's
   │        page-table register (CR3 on x86); this changes which address
   │        translations are valid (Section 3.14)
   │      - load Y's kernel stack pointer, registers, and thread-local-storage
   │        pointer (the FS register)
   │ 6. Now on Y's kernel stack, where Y stopped last time. Unwind Y's path and
   │    restore Y's user registers.
   │ 7. Return to user mode at Y's saved instruction pointer.
   ▼
 Thread Y running in user mode, unaware it was ever stopped

 core 0 ───X (user)───┤ kernel: save X, pick Y, switch ├───Y (user)─────
                      ▲                                 ▲
               timer interrupt                   return to user
                      └─────────── direct cost ─────────┘
```

### The cost: direct and indirect

The **direct cost** is the work above: kernel entry and exit, saving and restoring registers, the scheduler's choice, and possibly an address-space switch. It is typically on the order of a microsecond to a few microseconds on modern hardware, depending on the CPU, kernel and security mitigations. Measure on your own systems rather than trusting a universal number.

The **indirect cost** is often larger and harder to see:

- **CPU caches.** Y's data is probably not in L1/L2, which are full of X's data (Chapter 2). Y runs slowly at first, taking cache misses until its data is reloaded. When X returns, it pays the same price.
- **TLB.** The TLB caches address translations for one address space (Section 3.14). After a switch to another process the old entries are useless. Older CPUs flushed the whole TLB on every address-space switch; modern x86 CPUs can tag entries with an address-space identifier (PCID) so they survive, but Y still starts with few useful entries.
- **Branch predictors** and other learned per-core state are similarly cold for Y.

Switching between two threads **of the same process** is cheaper: the address space stays the same, so there is no page-table switch, TLB entries stay valid, and the threads may share cached data.

### Concrete: too many threads

Assume an 8-core machine running a pool of 2,000 threads, all CPU-bound (a misconfigured image-processing service), each running about 1 ms per turn.

- At any moment 8 threads run and 1,992 wait, about 250 per core.
- A thread that has just run waits roughly 250 × 1 ms ≈ 250 ms for its next turn. A request needing 20 ms of CPU is spread over 20 turns, several seconds of wall-clock time.
- Each core switches about 1,000 times per second, and every thread resumes with a cold cache.

Reduce the pool to 8 threads and each request runs to completion with a warm cache. Waiting requests sit in the application's queue, where they consume nothing, and throughput *rises*. For CPU-bound work, more runnable threads than cores add overhead, not capacity. (I/O-bound work is different; see Section 3.12.)

### Guarantees and non-guarantees

A context switch guarantees that a thread resumes with exactly the registers and memory view it had, so preemption is invisible to the program's logic. It does **not** guarantee *when* the thread resumes or how fast it runs afterwards (cold caches). It does not make a sequence of operations atomic either: a thread can be switched out between any two instructions, which is precisely why the races of Chapter 5 exist.

### Failure and diagnosis

```bash
$ vmstat 1          # 'cs' = context switches per second, whole machine
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
37  0      0 812344  91200 6102344    0    0     0    12 9120 187402 61 34  5  0  0

$ grep ctxt /proc/4121/status
voluntary_ctxt_switches:	88231
nonvoluntary_ctxt_switches:	9120458

$ pidstat -w -p 4121 1     # per second: cswch/s (voluntary), nvcswch/s (involuntary)
$ perf stat -e context-switches,cpu-migrations -p 4121 -- sleep 10
```

This is the signature of thread oversubscription: a run queue (`r`) of 37 on an 8-core machine, about 187,000 switches per second, high system CPU (`sy` 34%), and a process whose involuntary switches dwarf its voluntary ones (its threads are being preempted, not waiting for I/O). There is no universal "too many switches" threshold; compare with the same service's healthy baseline.

### Backend connection

Context-switch cost is the main argument behind event loops (Chapters 4 and 5) and lightweight threads (Section 3.8): both handle many waiting connections without a kernel thread switch for each. Size CPU-bound pools close to the core count; size I/O-bound pools by how much time is spent waiting (next section and Chapter 11).

---

## 3.12 CPU-Bound and I/O-Bound Work, Load, and Throttling

### Definitions

A task is **CPU-bound** if the CPU limits its speed: a faster processor would finish it sooner. A task is **I/O-bound** if waiting for input/output (disk, network, a database) limits its speed: it spends most of its time blocked, and a faster CPU would barely help.

The **run queue** is the set of threads that are ready to run and waiting for a CPU. Its length is the most direct measure of CPU contention.

**Load average** on Linux is an exponentially smoothed average of the number of threads that are either runnable (state `R`: running or in the run queue) or in uninterruptible sleep (state `D`: usually waiting for disk I/O), reported over 1, 5 and 15 minutes.

### Mechanism: the shape of each kind of work

```text
 CPU-bound:  ████████████████████████████████████████   long bursts; preempted by the timer
 I/O-bound:  ██░░░░░░░░░░░░██░░░░░░░░░░░░░░██░░░░░░░░   short bursts; blocks voluntarily
             █ = on CPU     ░ = blocked waiting for I/O
```

A typical web request is I/O-bound: a millisecond or two of computation (parsing, serialising, business logic) and several milliseconds of waiting on the database and other services. That shapes thread pools. If a request spends 2 ms on CPU and 18 ms waiting, one core can keep about 10 such requests in progress. A thread-per-request server on 8 cores needs roughly 80 threads to keep the cores busy:

```text
 threads ≈ cores × (1 + wait time / compute time) = 8 × (1 + 18/2) = 80
```

This is a starting estimate, not a law. It assumes stable timings and that whatever is being waited on (the database, its connection pool) has spare capacity. Chapter 11 develops the full reasoning with Little's Law. For purely CPU-bound work the formula gives `cores × 1`, matching Section 3.11.

### Reading load average

```bash
$ uptime
 10:15:02 up 12 days,  3:04,  1 user,  load average: 6.12, 4.80, 2.41
$ cat /proc/loadavg
6.12 4.80 2.41 7/1534 88123
$ nproc
8
```

In `/proc/loadavg` the fourth field is "runnable tasks now / total tasks" and the fifth is the most recently assigned PID.

- **Compare with the number of cores.** A load of 6 on 8 cores means that, on average, about 6 threads wanted a CPU (or were stuck in `D`). A load of 6 on 2 cores means about 4 were always waiting.
- **Compare the three windows.** 6.12, 4.80, 2.41 means the load is rising; the reverse order would mean it is falling.
- **Load is not CPU utilisation.** Because Linux counts `D` threads, a machine with stalled storage (or a hung network filesystem) can show a load of 40 with an idle CPU. Many other Unix systems count only runnable threads, so the same number means different things on different systems.

| Situation | CPU utilisation | Load average | `r` (run queue) | `b` / `D` threads | Meaning |
|---|---|---|---|---|---|
| CPU saturation | ~100% | > cores | > cores | low | More computation than cores |
| Storage stall | low | high | low | high | Threads waiting for disk; check iowait and disk latency (Chapter 4) |
| Healthy and busy | 60–80% | ≤ cores | ≤ cores | low | Fine |
| Waiting on a slow database | low | **low** | low | low | Threads blocked on sockets are in `S`, which load does **not** count; use latency metrics and traces |

The last row is a trap. A service whose threads all wait on a slow database shows a *low* load average, because waiting on a socket is interruptible sleep.

### iowait

The `wa` (iowait) CPU figure means *the CPU was idle and, at that moment, some thread was waiting for disk I/O*. It is a kind of idle time, not work. High iowait says storage is a bottleneck for someone. It does not mean the CPU is busy, and it can fall simply because another process started using the idle CPU. Disk diagnostics are in Chapter 4.

### CPU throttling under quotas

In containers, CPU is usually limited by a **cgroup** (control group): a kernel feature that groups processes and enforces resource limits on the group, explained in Chapter 4. A CPU limit is enforced as a **quota per period**. For example, "200 ms of CPU time per 100 ms period" allows 2 cores' worth on average. Once the group has used its quota, all its threads are **throttled**: not scheduled until the next period begins, even if cores sit idle.

Assume a container limited to 1 CPU (100 ms per 100 ms period) whose service has 4 busy threads:

```text
           0 ms                       25 ms                                100 ms
 thread 1: ██████████████████████████░░░░░░░░░░░░ throttled ░░░░░░░░░░░░░░░░░░│
 thread 2: ██████████████████████████░░░░░░░░░░░░ throttled ░░░░░░░░░░░░░░░░░░│
 thread 3: ██████████████████████████░░░░░░░░░░░░ throttled ░░░░░░░░░░░░░░░░░░│
 thread 4: ██████████████████████████░░░░░░░░░░░░ throttled ░░░░░░░░░░░░░░░░░░│
           4 threads × 25 ms = the whole 100 ms quota, used in 25 ms of wall time
```

Average usage looks like a modest "1 core", but every request in flight at the 25 ms mark freezes for 75 ms. The symptom is p99 latency spikes while average CPU stays below the limit. The evidence is in the cgroup's statistics; on cgroup v2, the `cpu.stat` file reports `nr_periods`, `nr_throttled` and `throttled_usec`. Fixes: raise the limit, reduce how many threads burst at once, or avoid hard CPU limits for latency-sensitive services.

In virtual machines, watch **steal time** (`st` in `top` and `vmstat`): time a virtual CPU was ready to run but the hypervisor ran another guest instead. Persistent steal means you are not getting the CPU you think you have.

### Guarantees and non-guarantees

Load average promises nothing about latency; it is a smoothed count of demand. A CPU quota guarantees an upper bound on CPU *time per period*, not a smooth rate and not a minimum. CPU utilisation below 100% does not prove the CPU is not your bottleneck: throttling, run-queue bursts shorter than the sampling interval, and a single saturated thread all hide inside an average.

### Failure and diagnosis

The situation table under "Reading load average" is the diagnostic guide for this section: it separates CPU saturation, storage stalls and waiting on the network by their CPU, load, `r` and `b` signatures. The two quota-related failures have their own evidence, described above: throttling shows up as `nr_throttled` rising in `cpu.stat`, and a noisy neighbour on a VM shows up as a sustained `st`. High `sy` belongs to Sections 3.2, 3.11 and 3.15 (syscall, context-switch and page-fault storms). Section 3.19 combines all of these into one decision tree.

### Backend connection

Whether a workload is CPU-bound or I/O-bound decides how you size thread pools (Chapter 11), whether more instances help, and whether the fix is faster code or a faster dependency. Load average is a quick first look, not an answer.


---

## 3.13 Virtual Memory

### Definition

**Virtual memory** is the abstraction that gives each process its own private, contiguous-looking address space, independent of where (or whether) its data sits in physical RAM. Programs use **virtual addresses**; a hardware unit in the CPU, the **memory management unit (MMU)**, translates each one into a **physical address** (a location in RAM) on every memory access, using tables the kernel maintains for each process.

### Why it exists

Imagine programs used physical addresses directly:

- **No isolation.** Any process could read or overwrite any other's memory, or the kernel's.
- **Relocation pain.** A program compiled to use address 0x400000 could not run if another program already occupied that RAM.
- **Fragmentation.** A process asking for 1 GB would need 1 GB of *contiguous* free RAM, which a long-running machine rarely has.
- **No overcommit.** The total memory of all processes could never exceed RAM, even though most programs reserve far more than they touch.

Virtual memory fixes all four. Each process gets the same clean address layout (Section 3.3). The kernel decides, page by page, which physical RAM backs it, can leave parts unbacked until they are touched, can share one physical copy between processes (shared libraries, COW after `fork`), and can move rarely used data out of RAM entirely (Section 3.17).

### Mechanism

```text
  Process A                          Physical RAM                   Process B
  virtual addresses                  (frames)                       virtual addresses
  ┌───────────────┐                  ┌─────────┐                    ┌───────────────┐
  │ 0x400000 code │ ───────────────► │ frame 7 │ ◄───────────────── │ 0x400000 code │  same program:
  ├───────────────┤                  ├─────────┤                    ├───────────────┤  one shared copy
  │ 0x601000 data │ ──────┐          │ frame 2 │ ◄──────────┐       │ 0x601000 data │
  ├───────────────┤       └────────► │ frame 9 │            └────── ├───────────────┤
  │ heap page     │ ──────────┐      │ frame 4 │ ◄───────────────── │ heap page     │
  ├───────────────┤           └────► │ frame 1 │                    ├───────────────┤
  │ unused page   │  (no frame:      └─────────┘                    │ ...           │
  └───────────────┘   not backed)
  Same virtual address 0x601000 in A and B → different physical frames. Isolation.
```

Every load and store instruction goes through the MMU. If the translation exists and the access is permitted, the CPU proceeds. If not, the MMU raises a **page fault** (Section 3.15) and the kernel decides what to do. Because the tables are per process and only the kernel can change them, a process simply has no way to *name* another process's memory.

### Guarantees and non-guarantees

Virtual memory guarantees isolation between processes (except for memory they deliberately share) and a consistent address layout. It does **not** guarantee that a valid virtual address is backed by RAM at this moment: touching it may require the kernel to allocate a page or read one from disk, which can take microseconds or milliseconds. Successfully *reserving* memory does not guarantee that memory will be available when you touch it (overcommit, Section 3.17).

### Failure and diagnosis

Accessing an address with no valid mapping, or writing to a read-only page, makes the kernel send `SIGSEGV`; the process dies with "Segmentation fault" (exit status 139 = 128 + 11 when reported by a shell). In memory-safe languages this usually means a bug in native code (a runtime, a driver, a native library).

### Backend connection

Virtual memory explains why the memory numbers you see for a process differ so much (Section 3.18), why a large allocation "succeeds" but the process is later killed (Section 3.17), and why the first request to touch fresh memory can be slower than later ones (Section 3.15).

---

## 3.14 Paging, Page Tables, and the TLB

### Definitions

**Paging** divides virtual memory into fixed-size blocks called **pages** and physical memory into blocks of the same size called **frames**, and maps each page independently to any frame (or to none). The usual page size is 4 KB. A **page table** is the per-process data structure that records this mapping. The **translation lookaside buffer (TLB)** is a small, fast cache inside the CPU that holds recently used page-to-frame translations.

### Mechanism: splitting an address

With 4 KB (2¹² byte) pages, the low 12 bits of an address are the **offset** within the page, and the remaining high bits are the **virtual page number (VPN)**. Translation replaces the VPN with a **physical frame number (PFN)** and keeps the offset unchanged.

```text
 virtual address  = [ virtual page number | offset (12 bits) ]
                              │ page table lookup        │ unchanged
 physical address = [ physical frame no.  | offset (12 bits) ]
```

Each **page table entry (PTE)** holds the frame number plus control bits:

| Bit | Meaning | Used for |
|---|---|---|
| Present | The page is in RAM | Not present → page fault |
| Writable | Writes allowed | Read-only code; COW (Section 3.5) |
| User | Accessible from user mode | Protecting kernel memory |
| No-execute | Not executable | Stacks and heaps are not runnable code |
| Accessed | Set by hardware when the page is read or written | Page replacement (Section 3.16) |
| Dirty | Set by hardware when the page is written | Must be written back before eviction |

### Why page tables are multi-level

A single flat table needs one entry per possible page, used or not.

- **32-bit addresses, 4 KB pages:** 2²⁰ ≈ 1 million entries × 4 bytes = **4 MB per process**, even for a process using 100 KB.
- **48-bit addresses (x86-64), 4 KB pages:** 2³⁶ entries × 8 bytes = **512 GB per process**. Impossible.

Address spaces are mostly empty (Section 3.3), so the fix is a **tree**: split the VPN into pieces, use the first piece to index a small top-level table whose entries point to second-level tables, and so on. Second-level tables exist **only for regions that are actually used**.

### Concrete: translating an address through a two-level page table

Use the classic 32-bit layout: 10 bits of top-level index, 10 bits of second-level index, 12 bits of offset.

```text
 Virtual address 0x00403A7C
 binary:  0000000001 | 0000000011 | 1010 0111 1100
          dir = 1      table = 3    offset = 0xA7C
```

(Check: the VPN is 0x00403 = 1027 = 1 × 1024 + 3.)

Assume the kernel has set up:

- The CPU's page-table base register points to the process's **page directory** in frame 0x00100.
- Page directory entry 1: present, points to a **page table** in frame 0x0002F.
- Entry 3 of that page table: present, writable, user, frame **0x07B21**.

```text
             ┌─────────────────┐
 CR3 ──────► │ page directory  │ (frame 0x00100)
             │ [0] not present │
             │ [1] ────────────┼──┐
             │ ...             │  │    ┌──────────────────┐
             └─────────────────┘  └──► │ page table       │ (frame 0x0002F)
                                       │ [0] ...          │
                                       │ [3] frame 0x07B21├──┐
                                       └──────────────────┘  │
                                                             ▼
                                   physical address = 0x07B21 << 12 | 0xA7C
                                                    = 0x07B21A7C
```

**TLB miss.** The CPU does not have VPN 0x00403 cached. The hardware **page walk** reads the directory entry (memory access 1), then the page-table entry (memory access 2), checks the permission bits, stores the translation in the TLB, and finally reads the data at 0x07B21A7C (access 3). Three memory accesses to read one value.

**TLB hit.** The next access anywhere in the same page, say 0x00403B00, finds VPN 0x00403 → PFN 0x07B21 in the TLB, forms 0x07B21B00 immediately, and makes **one** memory access.

What this costs on average. Assume a memory access takes 100 ns, a TLB lookup is effectively free (it happens in parallel with other work), and 99% of accesses hit the TLB:

```text
 effective access time = 0.99 × 100 ns + 0.01 × (2 × 100 + 100) ns = 99 + 3 = 102 ns
```

With x86-64's four levels, a miss needs four table reads: 0.99 × 100 + 0.01 × 500 = 104 ns. Real CPUs do better still, because page-table entries are themselves cached in L1/L2 and in dedicated page-walk caches. The TLB is what makes paging affordable; programs with poor locality that miss it often pay noticeably more.

**x86-64 in practice** uses four levels (9 + 9 + 9 + 9 bits of index + 12 bits of offset = 48-bit addresses, 256 TiB, half of it for user space). Some recent CPUs support a fifth level for 57-bit addresses.

### The TLB and huge pages

A TLB holds tens to a few thousand entries, depending on the CPU and the level. Its **reach**, the amount of memory it can translate without a miss, is entries × page size. Assume 1,536 entries:

| Page size | TLB reach |
|---|---|
| 4 KB | 1,536 × 4 KB = 6 MB |
| 2 MB (huge page) | 1,536 × 2 MB = 3 GB |

A database with a 20 GB buffer cache, or a runtime with a 16 GB heap, touches far more than 6 MB, so with 4 KB pages it takes TLB misses constantly. **Huge pages** (2 MB or 1 GB on x86-64) map large regions with a single entry, which also skips one or two levels of the page walk. Linux offers them in two ways:

- **Explicit huge pages:** reserved by the administrator and requested by the application (PostgreSQL's `huge_pages` setting and the JVM's large-page options both use this).
- **Transparent Huge Pages (THP):** the kernel automatically backs suitable regions with huge pages and merges small pages in the background. Convenient, but the background work and the cost of assembling 2 MB of contiguous memory can cause latency stalls, and several databases (Redis among them, which warns about it at startup) recommend disabling THP or limiting it to applications that opt in.

### Segmentation: the interview contrast

**Segmentation** is an older scheme that divides memory into variable-sized **segments** (code, data, stack), each described by a base address and a length. An address is (segment, offset), translated as base + offset after checking offset < length.

| | Paging | Segmentation |
|---|---|---|
| Unit | Fixed size (pages) | Variable size (segments) |
| Allocation | Any free frame fits | Needs a contiguous free region of the right size |
| Fragmentation | Internal (unused part of the last page) | External (free memory split into unusable gaps) |
| Matches program structure | No (a page is just 4 KB) | Yes (one segment per logical region) |
| Used today | Universally | x86-64 has largely flattened segmentation; the FS and GS segment registers survive mainly to point at thread-local storage (Section 3.7) |

- **Internal fragmentation** is wasted space *inside* an allocated unit: a 5 KB need rounded up to two 4 KB pages wastes 3 KB.
- **External fragmentation** is free memory split into pieces too small to satisfy a request, even though the total free amount is enough.

Paging eliminates external fragmentation of *physical* memory at the cost of some internal fragmentation, which is why it won. (Memory allocators inside a process still face both kinds within the heap, as Chapter 2 described.)

### Guarantees and non-guarantees

Paging guarantees that each page is translated and permission-checked on every access. It does **not** guarantee that consecutive virtual pages are physically adjacent, or that a translation is cached; a TLB miss costs extra memory accesses.

### Backend connection

Large-memory services (databases, caches, big heaps) can spend measurable time on TLB misses, which is why huge pages appear in database tuning guides. When a tuning guide tells you to disable THP, it is trading TLB efficiency for predictable latency.

---

## 3.15 Page Faults, Demand Paging, and mmap

### Definitions

A **page fault** is the exception the MMU raises when a memory access cannot be completed with the current page table: the page is not present, or the access violates its permissions. The kernel handles it and either fixes the mapping and retries the instruction, or kills the process.

- A **minor (soft) fault** is resolved without disk I/O: the kernel allocates a zeroed frame, maps a page already in RAM (a shared library page another process loaded, a file page in the page cache), or performs a COW copy. Cost: roughly microseconds.
- A **major (hard) fault** requires reading the page from disk (a file, or swap). Cost: the device's latency, from around a hundred microseconds on a fast SSD to milliseconds on an HDD (Chapter 2).
- An **invalid fault** is an access to an address with no valid mapping, or a forbidden access. The kernel sends `SIGSEGV`.

**Demand paging** is the policy of not loading or allocating a page until the process first touches it.

### Mechanism: handling a fault

```text
 1. Instruction touches a virtual address; MMU finds "not present" or a permission
    violation → page fault, CPU enters the kernel with the faulting address.
 2. Kernel looks up which memory region (Section 3.3) contains the address:
      - none, or the access breaks the region's rules   → SIGSEGV
      - anonymous memory never touched before          → allocate a zeroed frame (minor)
      - write to a COW-shared page                     → copy the page (minor)
      - file page already in the page cache            → map it (minor)
      - file page not in RAM, or a page in swap        → start a disk read, BLOCK the
                                                          thread (state D), run others,
                                                          resume on completion (major)
 3. Update the page table entry (frame, permissions).
 4. Return from the fault; the CPU re-executes the same instruction, which now succeeds.
```

Demand paging is why `allocate(1 GB)` returns instantly and uses almost no RAM: the kernel records a 1 GB region and allocates frames one page at a time as the program touches them. The first write to each page costs a minor fault. A program that writes 1 GB of fresh memory takes 262,144 minor faults (1 GB / 4 KB).

**COW revisited.** After `fork`, each first write to a shared page is a minor fault that copies the page (Section 3.5). It is the same mechanism: a deliberate "not writable" mark that lets the kernel intervene at the right moment.

### mmap

**`mmap`** is the system call that creates a new region in a process's address space. It comes in two flavours:

- **Anonymous mapping:** memory not backed by any file, initially zero. Memory allocators use it for large allocations (the GNU C library's `malloc` does so above a size threshold), and runtimes use it to reserve their heaps.
- **File-backed mapping:** the region *is* a file. Reading address `base + n` reads byte `n` of the file; the first touch of each page causes a fault that brings that part of the file into memory. With a **shared** mapping, writes go back to the file; with a **private** mapping, writes are COW copies visible only to this process.

```pseudo
region = mmap(file = "data.bin", length = 1 GB, shared = true)
x = region[500_000_000]     // page fault → kernel reads that 4 KB of the file → returns the byte
region[10] = 7              // marks the page dirty; written back to the file later
```

Executables and shared libraries are loaded this way: `exec` maps the program file, and pages of code are faulted in as they are executed. File-backed mappings rely on the kernel's **page cache** (the kernel's in-memory cache of file contents; Chapter 4), so two processes mapping the same file share the same physical frames.

Some databases (LMDB, for example) read their data files through `mmap` instead of `read` calls, letting the kernel decide what stays in memory. Others manage their own buffer cache precisely so that they, not the kernel, decide what to evict and when to write; Chapter 15 discusses the trade-off.

### Concrete: counting faults

```bash
$ /usr/bin/time -v ./import-job big.csv 2>&1 | grep -i fault
        Major (requiring I/O) page faults: 1834
        Minor (reclaiming a frame) page faults: 412870
$ ps -o pid,min_flt,maj_flt,comm -p 4121
    PID  MINFLT  MAJFLT COMMAND
   4121 9341221     212 myserver
$ sar -B 1          # system-wide paging: majflt/s among other columns (sysstat package)
```

### Guarantees and non-guarantees

The kernel guarantees that a fault on a valid address is transparent: the instruction retries and succeeds, and the program cannot tell (except by timing). It does **not** guarantee how long the fault takes. Any memory access can turn into a disk read if the page was evicted, which makes major faults a source of unpredictable latency.

### Failure and diagnosis

- **Latency spikes with major faults rising:** the working set no longer fits in RAM, or a memory-mapped file is cold. Check `majflt/s` in `sar -B`, `maj_flt` per process, and swap activity (`si`/`so` in `vmstat`).
- **Slow first requests after start:** minor faults while the runtime touches its heap for the first time, combined with JIT warm-up (Chapter 2). Some runtimes can pre-touch the heap at startup to move this cost out of the request path.
- **`SIGSEGV`:** invalid access; look for the faulting address in the kernel log (`dmesg`) and a core dump.

### Backend connection

The first touch of memory is not free, and any touch of evicted memory may hit the disk. Garbage collectors make this worse, because a full collection walks the whole heap: if part of the heap has been swapped out, the collection triggers thousands of major faults and a pause that should last milliseconds lasts seconds.

---

## 3.16 Page Replacement and Thrashing

### Definitions

When a page must be brought into RAM and no frame is free, the kernel must **evict** a page from some frame. A **page replacement algorithm** chooses the victim. If the victim is **dirty** (modified since it was loaded), it must first be written to its file or to swap; a clean page can simply be dropped and re-read later.

### Why it matters

Every bad choice turns into a future major fault, costing a disk access. The goal is to evict the page that will not be needed for the longest time. Every practical algorithm is an attempt to *predict* that from the past.

### The algorithms

- **FIFO:** evict the page that has been in memory longest. Simple, but age is not use: the oldest page might be the most popular.
- **Optimal (OPT, Belady's algorithm):** evict the page whose next use is farthest in the future. Provably the fewest faults, but it needs to know the future, so it is used only as a benchmark.
- **LRU (least recently used):** evict the page unused for the longest time, betting that the recent past predicts the near future (temporal locality, Chapter 2). Exact LRU would require updating a timestamp or a list on *every* memory access, which is far too expensive in hardware or kernel.
- **Clock (second chance):** a cheap LRU approximation using the hardware's *accessed* bit (Section 3.14). Frames sit in a circle with a "hand". To find a victim, the hand inspects the frame it points to: if its accessed bit is 1, clear it to 0 (a second chance) and advance; if it is 0, evict that page. A page referenced at least once per sweep survives.

### Concrete: FIFO, LRU and OPT on one reference string

Three frames, initially empty. Reference string: **2 3 2 1 5 2 4 5 3 2 5 2**. `F` marks a fault; frames are listed in no particular order except for FIFO, where the leftmost is the oldest.

**FIFO**

| Ref | 2 | 3 | 2 | 1 | 5 | 2 | 4 | 5 | 3 | 2 | 5 | 2 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 2 | 2 3 | 2 3 | 2 3 1 | 3 1 5 | 1 5 2 | 5 2 4 | 5 2 4 | 2 4 3 | 2 4 3 | 4 3 5 | 3 5 2 |
| Fault | F | F | | F | F | F | F | | F | | F | F |

**9 faults.** At reference 5 (position 5), FIFO evicts 2, the oldest page, which is needed again immediately.

**LRU**

| Ref | 2 | 3 | 2 | 1 | 5 | 2 | 4 | 5 | 3 | 2 | 5 | 2 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 2 | 2 3 | 2 3 | 2 3 1 | 2 1 5 | 2 1 5 | 2 5 4 | 2 5 4 | 5 4 3 | 5 3 2 | 5 3 2 | 5 3 2 |
| Fault | F | F | | F | F | | F | | F | F | | |
| Evicted | | | | | 3 | | 1 | | 2 | 4 | | |

**7 faults.** At position 5, the least recently used page is 3 (last used at position 2), so the popular page 2 survives.

**OPT**

| Ref | 2 | 3 | 2 | 1 | 5 | 2 | 4 | 5 | 3 | 2 | 5 | 2 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 2 | 2 3 | 2 3 | 2 3 1 | 2 3 5 | 2 3 5 | 4 3 5 | 4 3 5 | 4 3 5 | 2 3 5 | 2 3 5 | 2 3 5 |
| Fault | F | F | | F | F | | F | | | F | | |
| Evicted | | | | | 1 | | 2 | | | 4 | | |

**6 faults.** At position 5, page 1 is never used again, so it goes. At position 7 the next uses are 5 → position 8, 3 → 9, 2 → 10, so 2 (farthest) is evicted. At position 10, neither 4 nor 3 is used again; evicting either gives the same count.

| Algorithm | Faults | Needs |
|---|---|---|
| FIFO | 9 | Arrival order only |
| LRU | 7 | Recency of every access (approximated in practice) |
| OPT | 6 | Knowledge of the future (impossible) |

### Belady's anomaly

You would expect more frames always to mean fewer (or equal) faults. For FIFO, that is false. Reference string: **1 2 3 4 1 2 5 1 2 3 4 5**.

FIFO with **3 frames** (leftmost = oldest):

| Ref | 1 | 2 | 3 | 4 | 1 | 2 | 5 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 1 | 1 2 | 1 2 3 | 2 3 4 | 3 4 1 | 4 1 2 | 1 2 5 | 1 2 5 | 1 2 5 | 2 5 3 | 5 3 4 | 5 3 4 |
| Fault | F | F | F | F | F | F | F | | | F | F | |

**9 faults.**

FIFO with **4 frames**:

| Ref | 1 | 2 | 3 | 4 | 1 | 2 | 5 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Frames | 1 | 1 2 | 1 2 3 | 1 2 3 4 | 1 2 3 4 | 1 2 3 4 | 2 3 4 5 | 3 4 5 1 | 4 5 1 2 | 5 1 2 3 | 1 2 3 4 | 2 3 4 5 |
| Fault | F | F | F | F | | | F | F | F | F | F | F |

**10 faults.** Adding a frame *increased* the faults. This is **Belady's anomaly**. It happens because FIFO's eviction order depends on arrival times, so the set of pages held with 4 frames is not a superset of the set held with 3. LRU and OPT are **stack algorithms**: the pages they keep with *n* frames are always a subset of those they keep with *n* + 1 frames, so they can never show the anomaly.

### What Linux does

Linux approximates LRU with lists of "active" and "inactive" pages, using accessed bits to promote and demote pages between them; recent kernels add a multi-generational LRU variant. It treats file-backed pages (which can be dropped and re-read) and anonymous pages (which must go to swap) differently. You do not tune these algorithms directly, but the vocabulary of FIFO, LRU, Clock and working sets carries straight into database buffer pools (Chapter 15) and application caches (Chapter 21), which face the same eviction problem.

### Working set and thrashing

The **working set** of a process is the set of pages it has referenced in a recent time window, roughly "the pages it needs right now". If every process's working set fits in RAM, faults are rare. If the sum of the working sets exceeds RAM, every process keeps evicting pages that another (or itself) needs moments later.

**Thrashing** is the state in which the system spends most of its time moving pages between RAM and disk instead of doing useful work. Throughput collapses while the disk is saturated with paging I/O.

```text
 useful
 throughput
    ▲            ┌───── peak
    │          ╱   ╲
    │        ╱       ╲
    │      ╱           ╲  thrashing: working sets exceed RAM,
    │    ╱               ╲            faults dominate
    │  ╱                   ╲____________
    └──────────────────────────────────────►  degree of multiprogramming
                                              (processes / memory demand)
```

The classic trap: the CPU looks under-used during thrashing (threads are blocked on page I/O), so a naive scheduler admits *more* work, which makes it worse. The cure is less memory demand: fewer processes, smaller heaps or caches, or more RAM. Swapping a few idle pages is normal; living in major faults is thrashing.

### Failure and diagnosis

- `vmstat`'s `si`/`so` (swap in/out per second) persistently non-zero, `b` high, `wa` high.
- Major faults per second high and rising (`sar -B`).
- **Pressure stall information:** `/proc/pressure/memory` reports the share of time tasks were stalled waiting for memory, which measures thrashing's real cost directly:

```bash
$ cat /proc/pressure/memory
some avg10=31.04 avg60=24.70 avg300=10.12 total=918273645
full avg10=12.55 avg60=9.80 avg300=3.97 total=301928475
```

`some` is the share of time at least one task was stalled on memory; `full` is the share of time all non-idle tasks were stalled at once. Sustained double-digit percentages mean memory is the bottleneck.

---

## 3.17 Swap, Overcommit, and the OOM Killer

### Definitions

**Swap** is disk space (a partition or a file) where the kernel can store anonymous pages evicted from RAM. **Memory overcommit** is the kernel's policy of allowing processes to reserve more virtual memory than RAM plus swap can actually back, betting that not all of it will be touched. The **OOM (out-of-memory) killer** is the kernel mechanism that, when memory truly runs out and nothing more can be reclaimed, chooses a process and kills it with `SIGKILL` to free memory.

### Why each exists

- **Swap** lets the kernel move genuinely idle anonymous memory (initialisation code's data, a forgotten idle process) out of RAM, freeing room for useful pages such as file cache.
- **Overcommit** exists because programs routinely reserve far more than they use: sparse data structures, thread stacks reserved at 8 MB and using 50 KB, and especially `fork`, which logically duplicates a process's entire memory even though COW means little of it will ever be copied. Without overcommit, a 10 GB process could not fork on a machine without another 10 GB free.
- **The OOM killer** exists because overcommit is a bet that can be lost. When processes actually touch more memory than exists, a page fault cannot be satisfied. The kernel cannot make a fault return "no memory" to an instruction, so it frees memory the only way it can: by killing something.

### Why backends avoid swap

For a latency-sensitive service, swap turns memory accesses into disk accesses at unpredictable moments. A request that touches a swapped-out page waits for a major fault; a garbage collector that walks a partly swapped heap faults on every swapped page. The service gets slow before anything fails, and a slow-but-alive instance is often worse than a dead one, because health checks pass and load balancers keep sending it traffic. Many production setups therefore run with little or no swap, preferring a fast, visible OOM kill and a restart. Kubernetes historically required swap to be disabled on nodes; newer versions add limited, opt-in swap support. The kernel's willingness to swap anonymous memory rather than drop file cache is tunable (`vm.swappiness`).

### Overcommit modes

Linux's policy is set by `vm.overcommit_memory`:

| Mode | Behaviour |
|---|---|
| `0` (heuristic, the default) | Refuse only allocations that are obviously impossible; otherwise allow |
| `1` (always) | Never refuse; every allocation succeeds until memory is actually touched |
| `2` (strict) | Refuse allocations once total committed memory exceeds a limit computed from swap plus a configurable percentage of RAM |

Under strict mode, an allocation fails *at the call* (the program sees an error it can handle) instead of a process being killed later. Database vendors disagree on the right choice, because they have different needs. PostgreSQL's documentation recommends strict overcommit on dedicated database servers, so that the OOM killer does not kill a backend or the postmaster. Redis recommends `vm.overcommit_memory = 1`, because its background snapshot `fork`s a large process (Section 3.5) and must not be refused.

### Mechanism: how the OOM killer chooses

When reclaim fails, the kernel computes a **badness score** for each candidate process, mainly proportional to how much memory it uses (RSS, swap and page tables), and adjusted by a per-process value, `oom_score_adj`, from −1000 (never kill) to +1000 (kill first). It kills the highest scorer with `SIGKILL`. You can read the current score of any process:

```bash
$ cat /proc/4121/oom_score
812
$ cat /proc/4121/oom_score_adj
0
```

With cgroups (Chapter 4), the same killer also runs **per container**: if a container exceeds its own memory limit, a process inside that container is killed even when the host has plenty of free memory. That is the most common OOM kill in practice.

### Concrete: what the evidence looks like

The killed process logs nothing: `SIGKILL` gives it no chance to run any code. The evidence is elsewhere:

```bash
$ dmesg -T | grep -i -E "out of memory|oom"
[Wed Oct 14 03:12:44 2026] Memory cgroup out of memory: Killed process 4121 (myserver)
    total-vm:6123444kB, anon-rss:2091232kB, file-rss:10240kB, shmem-rss:0kB, ... oom_score_adj:0
```

(Exact fields vary between kernel versions.) The kernel log is the decisive piece; the other corroborating evidence (exit status, platform events, the memory trend) is collected in the "My process vanished" checklist in Section 3.19.

### Guarantees and non-guarantees

The kernel guarantees that a process whose memory allocation succeeded has *address space*; under overcommit modes 0 and 1 it does **not** guarantee that the memory will exist when touched. The OOM killer guarantees the system keeps running; it does **not** guarantee it kills the "guilty" process, only the one with the highest score, and it gives the victim no chance to clean up.

### Backend connection

An OOM kill is invisible to the application's own logs, so it must be watched from outside. Size containers by total process memory, not by heap size (next section), protect critical processes with `oom_score_adj`, and alert on memory approaching the limit.

---

## 3.18 What the OS Measures: VSZ, RSS, and the Runtime's Heap

### Definitions

- **VSZ** (virtual memory size, `VIRT` in `top`): the total size of all virtual memory regions the process has mapped, whether or not they are backed by RAM. Includes reserved-but-untouched heaps, thread stack reservations and mapped files.
- **RSS** (resident set size, `RES` in `top`): the amount of the process's memory currently in physical RAM. Includes its private pages *and* every resident shared page it maps (libraries, shared memory, mapped files), each counted in full.
- **PSS** (proportional set size): RSS with each shared page divided by the number of processes sharing it. The PSS of all processes sums to the real total.
- **Runtime heap:** the memory region a language runtime manages for objects (Chapter 2). The OS knows nothing about it; it is just part of the process's anonymous memory.

### Why they differ

```text
 VSZ ≥ RSS, always; and RSS ≠ heap, usually

 ┌────────────────────────────── VSZ: all mapped regions ─────────────────────────────┐
 │ ┌──────────────── RSS: pages resident in RAM ──────────────┐                       │
 │ │ heap pages touched │ stacks touched │ native allocations │ reserved but never    │
 │ │ libraries & files resident │ shared memory               │ touched; swapped out  │
 │ └──────────────────────────────────────────────────────────┘                       │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

VSZ is usually meaningless for capacity planning: a runtime may reserve tens of gigabytes of address space it never touches. RSS is the number the OOM killer and container memory limits care about (cgroups also count some other memory, such as the group's file cache; Chapter 4).

RSS has a trap of its own: **shared pages are counted in full in every process that maps them.** PostgreSQL backends all map the same shared buffer cache. If 50 backends have each touched 1 GB of it, each shows roughly 1 GB more RSS, and summing RSS reports 50 GB of usage for 1 GB of physical memory. Use PSS (from `/proc/<pid>/smaps_rollup`) to add up memory across processes.

### Concrete: why a 2 GB heap uses 3 GB of RSS

A service on a JIT-compiled, garbage-collected runtime (Chapter 2), configured with a 2 GB maximum heap, shows 3 GB RSS. Where is the extra gigabyte? An illustrative breakdown (assumed figures; yours will differ):

| Component | Assumed size | What it is |
|---|---|---|
| Heap | 2,048 MB | Objects (fully resident once the heap has been used, or pre-touched) |
| Thread stacks | 150 MB | 300 threads × ~0.5 MB actually touched (reservations are larger) |
| Class metadata | 180 MB | Loaded class and type structures (kept outside the object heap by some runtimes) |
| JIT code cache | 90 MB | Compiled machine code (Chapter 2) |
| GC bookkeeping | 120 MB | Collector data structures that track the heap |
| Direct/native buffers | 256 MB | Off-heap I/O buffers used by network and driver libraries |
| Native allocator overhead | 120 MB | Fragmentation and per-thread arenas in the C library's `malloc` |
| Libraries and mapped files | 60 MB | Runtime, libc and other shared libraries, resident portions |
| **Total RSS** | **≈ 3,024 MB** | |

None of the last seven rows appears in a heap graph. Container memory limits must cover the total, which is why setting the heap equal to the container limit leads to OOM kills as soon as the heap fills up.

```bash
$ grep -E 'VmSize|VmRSS|RssAnon|RssFile|RssShmem|VmSwap|Threads' /proc/4121/status
VmSize:	 6123444 kB        ← VSZ
VmRSS:	 3096576 kB        ← RSS = anon + file + shmem
RssAnon:	 3024012 kB        ← private memory: heap, stacks, native allocations
RssFile:	   72564 kB        ← resident parts of mapped files and libraries
RssShmem:	       0 kB
VmSwap:	       0 kB
Threads:	     300
$ grep -E '^(Rss|Pss)' /proc/4121/smaps_rollup
Rss:             3096576 kB
Pss:             3071230 kB
```

> **Java lens:** HotSpot's Native Memory Tracking (start with `-XX:NativeMemoryTracking=summary`, then run `jcmd <pid> VM.native_memory summary`) breaks the JVM's own memory into these categories. It does not see memory allocated by native libraries outside the JVM. In containers, size the heap as a fraction of the limit (for example with `-XX:MaxRAMPercentage`) and verify against measured RSS.
>
> **Other runtimes:** Go reports its own view through `runtime.MemStats`; Node.js reports heap and external memory through `process.memoryUsage()`. In every runtime, the OS-level RSS is the number that gets you killed.

### Failure and diagnosis

- **RSS grows while the heap is flat:** a native leak (buffers not released, native library leaks, allocator fragmentation, thread count growth). Track RSS, heap and thread count on one graph.
- **Sum of process RSS exceeds RAM:** shared pages counted repeatedly; switch to PSS.
- **OOM-killed with a "small" heap:** non-heap memory pushed RSS over the limit.

---

## 3.19 Observing the OS: A Field Guide

This section collects the tools used throughout the chapter and shows how to read them together. The labs at the end of the chapter apply them.

### `top`, annotated

```text
top - 14:42:19 up 12 days,  7:31,  1 user,  load average: 9.85, 7.12, 4.03    ← 1/5/15 min; 8 cores
Tasks: 213 total,   9 running, 204 sleeping,   0 stopped,   0 zombie         ← zombies > 0? (3.5)
%Cpu(s): 71.2 us, 16.4 sy,  0.0 ni,  9.9 id,  1.8 wa,  0.0 hi,  0.5 si,  0.2 st
MiB Mem :  15928.4 total,    412.3 free,  11020.6 used,   4495.5 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   4102.1 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   4121 app       20   0 5980.0m   2.9g  70.9m S 612.4  18.7 812:31.02 myserver
    977 postgres  20   0 4412.1m 612.4m 590.1m S  41.0   3.8  70:12.44 postgres
```

| Field | Meaning | Section |
|---|---|---|
| `us` | Time running user-space code | 3.12 |
| `sy` | Time running kernel code on behalf of processes (syscalls, faults, switching) | 3.2, 3.11 |
| `ni` | User time of processes with a raised nice value (lower priority) | 3.10 |
| `id` | Idle | |
| `wa` | Idle while some thread waited for disk I/O | 3.12 |
| `hi` / `si` | Hardware / software interrupt handling | 3.2 |
| `st` | Steal: wanted to run, but the hypervisor ran another guest | 3.12 |
| `avail Mem` | Estimate of memory available without swapping (free + reclaimable cache) | 3.17 |
| `VIRT` / `RES` / `SHR` | VSZ / RSS / the shared part of RSS | 3.18 |
| `S` | State: R, S, D, Z, T | 3.4 |
| `%CPU` | Per-core percentages summed: 612% ≈ 6 cores busy | |

In this snapshot, load (9.85) exceeds 8 cores and is rising, idle is under 10%, and `sy` at 16% is high. The `postgres` process's `SHR` of 590 MB is almost all of its `RES`: shared buffer cache, counted again in every backend (Section 3.18). In `htop`, press `H` to toggle the display of individual threads and `F2` to add columns such as major faults.

### `vmstat`, annotated

```text
$ vmstat 1 5        # one of the later lines shown
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 9  0      0 422188  60232 4543212    0    0     4   210 8211 21934 71 16 10  2  0
```

| Column | Meaning | Healthy versus worrying |
|---|---|---|
| `r` | Runnable threads (running + waiting for CPU) | Consistently > cores: CPU saturation |
| `b` | Threads in uninterruptible sleep (`D`) | Consistently > 0: storage stalls |
| `swpd`, `free`, `buff`, `cache` | Swap used; free, buffer and cache memory | Low `free` alone is normal: the kernel uses spare RAM as cache |
| `si`, `so` | Swap in / out per second | Persistently > 0: memory pressure, possibly thrashing |
| `bi`, `bo` | Blocks read / written | Disk activity (Chapter 4) |
| `in`, `cs` | Interrupts, context switches per second | Compare with the baseline |
| `us sy id wa st` | As in `top` | |

The first line of `vmstat` output reports averages since boot; read the following lines.

### `ps` and `/proc`

```bash
$ ps -eo pid,ppid,stat,nlwp,vsz,rss,min_flt,maj_flt,etime,comm --sort=-rss | head
$ ps -eo stat,pid,ppid,comm | awk '$1 ~ /^Z/'     # zombies and their parents
$ ps -eLo pid,tid,stat,pcpu,comm -p 4121           # per-thread CPU and state
```

| `/proc/<pid>/...` | What it tells you |
|---|---|
| `status` | State, PPID, `Threads`, `VmRSS`/`RssAnon`/`RssFile`, `VmSwap`, context-switch counts |
| `stat` | Raw counters, including minor/major faults and CPU time (machine-readable) |
| `maps`, `smaps_rollup` | Memory regions; RSS and PSS totals |
| `task/` | One directory per thread |
| `oom_score`, `oom_score_adj` | OOM-killer ranking |
| `limits` | Resource limits (Chapter 4) |
| `cmdline`, `environ` | How it was started |

### High CPU, high load, or high iowait?

```text
                          Is the service slow?
                                   │
         ┌──────────────────────── top / vmstat ────────────────────────┐
         │                         │                                    │
   us + sy near 100%,       load high but CPU idle,            CPU and load low
   r > cores                b > 0, wa high, D threads          but latency high
         │                         │                                    │
   CPU saturation          storage or NFS stall              waiting on something
   - which process? top    - which threads in D? ps          the OS doesn't count:
   - sy high? → syscalls,  - disk latency (Chapter 4)        network, database,
     switches, faults                                        locks, pools
   - us high? → profile    - si/so > 0? → swapping (3.17)    → app metrics, traces
     the code                                                  (Chapters 11, 26)
   - st high? → VM steal
   - throttled? → cpu.stat
```

### "My process vanished"

1. **Exit status:** 137 means `SIGKILL` (OOM or grace-period expiry); 143 means `SIGTERM`; 139 means `SIGSEGV`; other values come from the program itself.
2. **Kernel log:** `dmesg -T | grep -i oom`, or `journalctl -k`.
3. **Platform events:** the orchestrator's record of why it stopped (for example, `OOMKilled`, or a failed health check).
4. **Memory trend before death:** RSS climbing to the limit, then a drop to zero.
5. **Core dumps** for crashes from signals such as `SIGSEGV`.


---

## Chapter Summary

The operating system turns one machine into many safe virtual machines called processes. The hardware gives it the power to do so: code runs in **user mode** or **kernel mode**, and only the kernel can touch devices, page tables or other processes' memory. User code crosses into the kernel only through a **system call** (a deliberate trap to a fixed entry point), and the kernel regains control from any program through **interrupts**, above all the timer interrupt. A syscall costs far more than a function call, which is why I/O is buffered and batched. A syscall is a **mode switch**, not necessarily a **context switch**.

A **process** is a program in execution: a private address space, open files, credentials and one or more threads, all described by a **PCB**. Processes move between **ready**, **running** and **blocked** as they are scheduled, preempted and wait for events. On Unix, `fork` creates a child as a **copy-on-write** clone, `exec` loads a new program into it, and `wait` collects its exit status. A child that has exited but not been waited for is a **zombie**; a child whose parent died is an **orphan**, adopted by PID 1. **Signals** deliver asynchronous events: `SIGTERM` asks for a graceful shutdown, `SIGKILL` (exit status 137) cannot be caught.

A **thread** is a flow of execution within a process. Threads share the address space and file descriptors and own their registers, stack and thread-local storage. That sharing makes them cheaper than processes and also riskier: races, no fault isolation. Kernel threads (1:1), green threads (N:1), M:N runtimes and coroutines are different answers to "who schedules, and how much does each thread cost?" Lightweight threads make waiting cheap; they do not add CPU or downstream capacity. Servers choose between processes (PostgreSQL, Nginx workers, Python pre-fork) and threads (MySQL, JVM servers) by weighing isolation against memory and sharing.

The **scheduler** decides which ready thread runs. FCFS suffers the convoy effect; SJF and SRTF minimise average waiting but need the future and can starve long jobs; Round Robin gives fast response at the cost of switching overhead; MLFQ learns which threads are interactive; aging prevents starvation. Linux schedules for fairness by virtual runtime. Each **context switch** saves and restores registers and possibly the address space, and its indirect cost (cold caches and TLB) is why oversubscribing CPU-bound threads lowers throughput. **Load average** counts runnable and uninterruptible threads, not CPU usage. **Throttling** under a CPU quota freezes all of a container's threads until the next period.

**Virtual memory** gives each process private addresses that the MMU translates to physical frames through **multi-level page tables**, made fast by the **TLB** and helped by **huge pages**. Pages are allocated on demand; the first touch is a **minor fault**, a page that must come from disk is a **major fault**, and COW and **mmap** are built on the same fault mechanism. When RAM is full, the kernel evicts pages using LRU approximations; FIFO can even get worse with more memory (**Belady's anomaly**). When working sets exceed RAM, the system **thrashes**. Swap makes latency unpredictable, overcommit lets processes reserve more than exists, and when the bet fails the **OOM killer** sends `SIGKILL`, leaving no application log. The OS measures **RSS**, not your heap, and RSS includes stacks, native memory, code and shared pages. Read `top`, `vmstat`, `ps` and `/proc` to tell CPU saturation, storage stalls, memory pressure and "waiting on something else" apart.

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| User/kernel mode | User code cannot touch hardware, kernel memory or other processes' memory | That the kernel is bug-free; that your process gets any amount of CPU or memory |
| System call | A controlled, validated entry into the kernel | Low cost; that it will not block |
| Process isolation | Private address space; a crash stays in the process | Unique PIDs over time; freedom from being killed |
| `fork` + COW | The child sees a logically identical, independent copy | Which runs first; that later writes are free (each copies a page) |
| `SIGTERM` | Delivered to a handler if one is installed | That the application drains in-flight work; that standard signals queue |
| `SIGKILL` | Cannot be caught or ignored | Any cleanup; immediate effect on a thread in `D` state |
| Threads | Shared memory and file descriptors | Visibility or ordering of writes without synchronisation (Chapter 5); fault isolation |
| Lightweight threads / coroutines | Cheap waiting | Extra CPU, connections or downstream capacity; fairness for CPU-heavy tasks |
| Preemptive fair scheduling | No thread monopolises a core; runnable threads eventually run | Any particular latency |
| Context switch | The thread resumes with its exact state | When it resumes; warm caches; atomicity of multi-step operations |
| CPU quota | An upper bound on CPU time per period | A smooth rate; a minimum share |
| Virtual memory | Isolation; consistent layout | That a valid address is in RAM right now |
| Demand paging / overcommit | Fast reservation | That memory exists when touched |
| Page replacement (LRU-like) | Recently used pages tend to stay | No faults; for FIFO, that more memory means fewer faults |
| OOM killer | The system survives memory exhaustion | That the "right" process dies; any chance for the victim to log |

## Interview Drill

**Q1. What is the difference between a process and a thread?**
A strong answer must mention:
- A process is a running program with its own address space, file descriptors and credentials; a thread is a flow of execution inside a process.
- Threads share the address space, heap, globals and file descriptors; each owns its registers, program counter, stack, thread-local storage and scheduling state.
- Threads are cheaper to create and switch between (no new page tables, no address-space switch).
- Threads risk races and share fate (one crash kills all); processes are isolated and communicate through the kernel.
- On Linux both are tasks created by `clone`, with different sharing flags.
*Common weak answer:* "A thread is a lightweight process." True as a slogan, but it says nothing about what is shared and what is owned.

**Q2. What is a system call, and why is it more expensive than a function call?**
A strong answer must mention:
- The controlled way user code asks the kernel for a privileged operation, entered with a special trap instruction to a fixed kernel entry point.
- Steps: library wrapper puts the syscall number and arguments in registers → trap → mode switch → kernel saves registers, validates arguments, copies data → handler runs → result returned → back to user mode.
- Costs: mode switches, register saving, argument validation and copying, security mitigations, cache/TLB pollution, and possibly blocking.
- Mitigation: buffering and batching; the vDSO avoids the trap for some read-only calls.
*Common weak answer:* "Because it causes a context switch." A syscall always switches mode but only switches threads if it blocks.

**Q3. What is the difference between user mode and kernel mode?**
A strong answer must mention:
- A hardware privilege level enforced by the CPU.
- Kernel mode can execute privileged instructions, access all memory and devices; user mode cannot.
- Transitions into kernel mode only via syscalls, interrupts and exceptions, to kernel-chosen entry points.
- This boundary is what makes isolation enforceable.
*Common weak answer:* "Kernel mode is for the admin user." Privilege modes are about the CPU, not user accounts; root's programs also run in user mode.

**Q4. What are the states of a process, and what causes each transition?**
A strong answer must mention:
- New, ready, running, blocked (waiting), terminated.
- Ready → running: dispatched; running → ready: preempted (timer or a more important thread).
- Running → blocked: a blocking request (I/O, sleep, lock); blocked → ready (not running) when the event occurs.
- Running → terminated on exit or fatal signal; the entry remains as a zombie until reaped.
- Linux letters: R, S, D, Z, T; D cannot be interrupted even by `SIGKILL`.
*Common weak answer:* "When the I/O finishes, the process starts running." It becomes ready and waits for a CPU.

**Q5. How do `fork` and `exec` work? What is copy-on-write?**
A strong answer must mention:
- `fork` creates a child that is a logical copy of the parent; it returns 0 in the child and the child's PID in the parent.
- `exec` replaces the program image in the current process, keeping the PID and open file descriptors (except close-on-exec).
- The fork–exec split allows the child to configure redirections and limits before loading the new program.
- COW: page tables are copied, pages are shared read-only, and a write triggers a fault that copies only that page.
- Consequence: fork is fast, but heavy writes afterwards duplicate memory (Redis snapshots).
*Common weak answer:* "`fork` copies all the memory of the process." Not since COW; only page tables are copied up front.

**Q6. What is a zombie process? What is an orphan? How is each cleaned up?**
A strong answer must mention:
- Zombie: has exited, but its parent has not called `wait`; only its PCB entry (PID, exit status) remains.
- Zombies use no CPU or memory but hold PIDs; many of them exhaust the PID space and make `fork` fail.
- A zombie cannot be killed; fix the parent's reaping, or kill the parent so PID 1 adopts and reaps it.
- Orphan: a running process whose parent exited; re-parented to PID 1 (or a subreaper), which reaps it later.
- In containers the application may be PID 1 and must reap and handle signals, or use a tiny init.
*Common weak answer:* "A zombie is a process stuck in an infinite loop." That is a runaway process; a zombie is already dead.

**Q7. What is the difference between SIGTERM and SIGKILL? How does a server shut down gracefully?**
A strong answer must mention:
- `SIGTERM` can be handled; `SIGKILL` cannot be caught, blocked or ignored.
- Graceful shutdown on `SIGTERM`: stop accepting new work (and signal not-ready), finish in-flight requests within a deadline, release connections, flush, exit.
- Platforms send `SIGTERM`, wait a grace period, then `SIGKILL`.
- Exit status 143 vs 137; PID 1 ignores signals it has no handler for.
*Common weak answer:* "SIGTERM is a soft kill that always lets requests finish." Only if the application implements draining.

**Q8. What is the difference between user-level and kernel-level threads? What is M:N threading?**
A strong answer must mention:
- Kernel threads are scheduled by the kernel and can run in parallel on cores; each costs a kernel structure, a stack and kernel-mediated switches.
- User-level threads are managed by a runtime; cheap to create and switch, but with N:1 a blocking syscall blocks all of them and they cannot use multiple cores.
- M:N multiplexes many user threads onto a few kernel threads; the runtime intercepts blocking I/O and parks threads (Go goroutines, Java virtual threads).
- They make waiting cheap but add no CPU or downstream capacity.
*Common weak answer:* "User threads are faster, so they're always better." They lose parallelism and block together unless the runtime handles I/O.

**Q9. Explain FCFS, SJF, Round Robin and MLFQ. What is starvation, and how does aging fix it?**
A strong answer must mention:
- FCFS: arrival order, non-preemptive, convoy effect.
- SJF / SRTF: shortest (remaining) burst first, optimal average waiting time, but needs burst predictions and can starve long jobs.
- RR: time quantum, good response, overhead if the quantum is too small, FCFS-like if too large.
- MLFQ: multiple priority queues; threads that use their full allotment move down; periodic boost.
- Starvation: a ready thread never runs; aging raises priority with waiting time. Different from deadlock (Chapter 5).
*Common weak answer:* "Round Robin is best because it is fair." It has worse average waiting than SJF; it is chosen for response time and no starvation.

**Q10. Given jobs A(0, 7), B(1, 4), C(2, 1), D(3, 4) as (arrival, burst), compute the average waiting time under FCFS and SRTF.**
A strong answer must mention:
- FCFS order A, B, C, D: completions 7, 11, 12, 16; waits 0, 6, 9, 9; average 6.0.
- SRTF: A runs 0–1, B 1–2, C 2–3, B 3–6, D 6–10, A 10–16; waits 9, 1, 0, 3; average 3.25.
- Waiting = completion − arrival − burst.
- SRTF preempts whenever a shorter job arrives.
*Common weak answer:* Computing waiting time as start time only, which is wrong for preempted jobs.

**Q11. What happens during a context switch, and why is it expensive?**
A strong answer must mention:
- Triggered voluntarily (blocking) or involuntarily (timer preemption, a more deserving thread).
- The kernel saves the old thread's registers, stack pointer, program counter and FPU/SIMD state, picks the next thread, switches page tables if the process differs, and restores the new thread's state.
- Direct cost around microseconds; indirect cost from cold caches, TLB and branch predictors.
- Same-process switches are cheaper (no address-space change).
- Too many runnable threads raise switching and lower throughput; measure with `vmstat cs`, `pidstat -w`.
*Common weak answer:* "It saves the process to disk." That is swapping, a different mechanism.

**Q12. How does virtual memory work?**
A strong answer must mention:
- Each process uses virtual addresses; the MMU translates them to physical frames through per-process page tables on every access.
- Pages (usually 4 KB) map independently to any frame, or to none.
- Provides isolation, a consistent layout, sharing (libraries, COW) and overcommit.
- Missing or forbidden mappings cause page faults, handled by the kernel.
*Common weak answer:* "Virtual memory is using the disk as extra RAM." That is swap, one optional use of virtual memory.

**Q13. What is a page table? Why are page tables multi-level? What is the TLB?**
A strong answer must mention:
- A page table maps virtual page numbers to frame numbers plus permission, present, accessed and dirty bits.
- A flat table would need an entry per possible page (4 MB per process at 32 bits; impossible at 48 bits); multi-level tables allocate lower levels only for used regions.
- The TLB caches recent translations; a hit avoids the walk, a miss costs several memory reads.
- TLB reach is limited; huge pages extend it.
- Address-space switches make TLB entries useless unless tagged (PCID).
*Common weak answer:* "The TLB caches data from memory." It caches translations, not data.

**Q14. What is a page fault? Minor versus major?**
A strong answer must mention:
- An exception raised by the MMU when a page is not present or the access is not permitted.
- Minor: resolved without disk I/O (first touch, COW copy, page already in cache).
- Major: needs a disk read (file or swap); the thread blocks.
- Invalid: `SIGSEGV`.
- Major faults are latency spikes; measure with `maj_flt`, `sar -B`.
*Common weak answer:* "A page fault is an error that crashes the program." Most are normal and invisible.

**Q15. Compare FIFO, LRU and Optimal page replacement. What is Belady's anomaly?**
A strong answer must mention:
- FIFO evicts the oldest-loaded page; LRU the least recently used; OPT the page used farthest in the future (a benchmark only).
- On 2 3 2 1 5 2 4 5 3 2 5 2 with 3 frames: FIFO 9, LRU 7, OPT 6 faults.
- Exact LRU is too costly; real systems approximate it with accessed bits (Clock).
- Belady's anomaly: with FIFO, more frames can produce more faults (1 2 3 4 1 2 5 1 2 3 4 5: 9 faults with 3 frames, 10 with 4); LRU and OPT are stack algorithms and immune.
*Common weak answer:* "LRU is optimal." OPT is optimal; LRU approximates it using the past.

**Q16. What is thrashing, and how do you detect it?**
A strong answer must mention:
- The sum of working sets exceeds RAM, so the system spends its time paging instead of working.
- Symptoms: high major faults, swap in/out, high iowait and blocked threads, collapsing throughput, CPU apparently idle.
- Evidence: `vmstat si/so`, `sar -B`, `/proc/pressure/memory`.
- Fix: reduce memory demand (fewer processes, smaller heaps or caches) or add RAM; do not add more work.
*Common weak answer:* "Thrashing is when the CPU is at 100%." The CPU is often idle, waiting for paging I/O.

**Q17. A process with a 2 GB heap uses 3 GB of RSS. Why?**
A strong answer must mention:
- The OS measures RSS: all resident pages, not just the heap.
- Additions: thread stacks, class metadata, JIT code cache, GC structures, direct/native buffers, allocator fragmentation, mapped libraries.
- Shared pages appear in every process's RSS (use PSS to sum).
- Container limits apply to the total; heap = limit leads to OOM kills.
*Common weak answer:* "It's a memory leak." It may be normal non-heap memory.

**Q18. A process disappeared with no error in its logs. What happened, and how do you prove it?**
A strong answer must mention:
- Most likely `SIGKILL` from the OOM killer (system or cgroup) or after a shutdown grace period; `SIGKILL` allows no logging.
- Exit status 137; kernel log line "Out of memory: Killed process ..." in `dmesg`; orchestrator reason `OOMKilled`.
- RSS climbing toward the limit before death.
- Overcommit means allocations succeeded earlier; the kill happens when memory is touched.
*Common weak answer:* "The application must have crashed silently." A crash usually leaves a signal-specific status (such as 139) and often a core dump.

## Scenario Exercises

1. A PostgreSQL server on a 64 GB machine reports that the sum of RSS over all `postgres` processes is 210 GB, yet the machine is not swapping. A colleague says the monitoring is broken. Is it?
2. A batch job runs a shell script that, in a loop, starts a helper process in the background for each input file. After a few days in a container, the job fails with "fork: Resource temporarily unavailable". `ps` inside the container shows thousands of entries marked `<defunct>`. Explain the mechanism and two fixes.
3. A Redis instance using 12 GB on a 16 GB host is killed every night during its background save, but never during the day. Explain why the save is the trigger and what you could change.
4. After a deploy, a service's p99 latency rises from 40 ms to 900 ms. CPU usage drops. `vmstat` shows `si` and `so` both in the thousands and `b` at 20. What happened, and what is the quickest mitigation?
5. A latency-critical service shares a host with a nightly report generator. Every night the service's p99 doubles while its own CPU usage is unchanged. What scheduling mechanisms explain this, and what are two ways to protect the service?

## Diagnosis Lab

**Lab 1: High load average, idle CPU**
- *Symptom:* a host's load average climbs to 45 on 8 cores. CPU is 85% idle. Requests that write audit files time out; read-only requests are fine.
- *Hypotheses:* (a) CPU saturation (contradicted by idle CPU); (b) threads stuck in uninterruptible sleep on storage; (c) swapping.
- *Evidence:* `vmstat 1` shows `r` around 1, `b` around 40, `wa` 60%, `si`/`so` zero (not swapping). `ps -eLo stat,pid,tid,comm | grep ^D` shows dozens of server threads in `D`. `/proc/<pid>/task/<tid>/stack` (as root) shows them waiting in the filesystem's write path. Disk latency tools (Chapter 4) show the audit volume at multi-second latency.
- *Root cause:* the network-attached volume for audit logs degraded; threads writing synchronously blocked in `D` state, counted in load average.
- *Fix:* fail over the volume; make audit writes asynchronous with a bounded buffer so request threads never block on that disk.
- *Prevention:* alert on `D`-state thread counts and disk latency, not only CPU; separate slow I/O from request threads (bulkheads, Chapter 24).

**Lab 2: The service that vanished**
- *Symptom:* a service instance restarts several times a day. Its last log lines are ordinary request logs. The orchestrator shows exit code 137.
- *Hypotheses:* (a) OOM kill; (b) `SIGKILL` after a failed graceful shutdown (for example, a liveness check failing and the platform restarting it); (c) an application crash.
- *Evidence:* `dmesg -T | grep -i oom` shows "Memory cgroup out of memory: Killed process ... (myserver)" at each restart time. The orchestrator's status says `OOMKilled`. The memory graph shows RSS climbing steadily to the 4 GB container limit, while the runtime's heap graph stays flat at about 2.5 GB. `/proc/<pid>/status` on a live instance shows `Threads` growing from 200 to 1,400 over several hours.
- *Root cause:* a thread leak: a code path created a new thread per outbound call and never terminated it. Each thread's stack and native buffers added RSS outside the heap, until the cgroup limit triggered the OOM killer.
- *Fix:* use a bounded pool for outbound calls; cap total thread count.
- *Prevention:* alert on RSS against the limit (not just heap), on thread count trends, and on exit code 137.

**Lab 3: Latency doubles after raising a thread pool**
- *Symptom:* after increasing a CPU-heavy service's worker pool from 16 to 512 on a 16-core machine, throughput drops 20% and p99 latency doubles.
- *Hypotheses:* (a) context-switch overhead and cache thrashing from oversubscription; (b) lock contention on shared state (Chapter 5); (c) memory pressure from extra stacks.
- *Evidence:* `vmstat 1`: `r` ≈ 300, `cs` rose from ~8,000 to ~150,000 per second, `sy` from 4% to 22%. `pidstat -w`: involuntary switches dominate. `si`/`so` zero, so not memory. A profiler shows no hot lock.
- *Root cause:* CPU-bound work with far more runnable threads than cores; each request's CPU time is sliced across many turns with cold caches, and switching consumes CPU.
- *Fix:* size the pool near the core count for this CPU-bound path; let excess requests wait in a queue (and reject when the queue is full; Chapter 24).
- *Prevention:* load-test pool changes; track run queue length and involuntary context switches.

**Lab 4: Periodic stalls under a CPU limit**
- *Symptom:* a containerised API with a 1-CPU limit has p99 latency spikes of about 80 ms, although its average CPU usage is 0.6 cores.
- *Hypotheses:* (a) GC pauses (Chapter 2); (b) CPU throttling; (c) a slow dependency.
- *Evidence:* the cgroup's `cpu.stat` shows `nr_throttled` rising in step with the spikes and `throttled_usec` growing; GC logs show only short pauses; traces show the time spent inside the service itself, not in calls.
- *Root cause:* bursts of parallel work (16 request threads plus GC threads) used the whole 100 ms quota in a fraction of each period; all threads were then frozen until the next period.
- *Fix:* raise the CPU limit or remove it for this service; reduce parallelism (fewer GC and worker threads matched to the limit).
- *Prevention:* graph throttling alongside latency; size runtimes and pools from the container's CPU limit, not the host's core count.

## Solution Sketches

1. Probably not broken. PostgreSQL backends share the shared buffer cache, and RSS counts shared pages in full in every process that maps them. Summing RSS multiplies the shared memory by the number of backends. Use PSS from `/proc/<pid>/smaps_rollup` (or a tool that reports it) to get a real total.
2. The script backgrounds children but never `wait`s for them; when they exit they become zombies, each holding a PID, until the container's process limit (or PID space) is exhausted. If the script is PID 1 in the container, nothing else reaps them either. Fixes: `wait` for the background jobs (reap them), or run under a minimal init that reaps zombies (for example, `tini` or the container runtime's init option); also bound the number of concurrent helpers.
3. The background save forks the 12 GB process. COW shares pages initially, but the parent keeps serving writes, so every modified page is copied; under nightly write traffic (or a batch import), the copies push total memory past what the host can provide, and the OOM killer strikes (or the fork itself fails under strict overcommit). Options: schedule saves when writes are light, reserve memory headroom for the copies, reduce the dataset or move snapshots to a replica, disable THP (a huge page makes each COW copy 2 MB instead of 4 KB), and set overcommit as Redis recommends.
4. The new version uses more memory (a bigger cache or heap, or more threads); working sets no longer fit in RAM, and the host is swapping heavily (thrashing). CPU drops because threads are blocked on paging I/O (`b` = 20). Quickest mitigation: roll back the deploy or reduce the memory footprint (smaller cache or heap, fewer instances on the host). Then size the new version properly and consider disabling swap so failure is fast and visible.
5. CPU contention: the report generator's threads compete for the same cores, so the service's threads wait longer in run queues (more involuntary switches, longer scheduling delays) even though they use no more CPU themselves; caches are also polluted by the other workload. Protections: run the report at lower priority (`nice`) or under its own CPU limit (cgroups, Chapter 4); move it to separate hosts or reserve dedicated cores for the service (CPU affinity or exclusive allocation).

---

**END OF CHAPTER 3**
