# Chapter 5: Concurrency Fundamentals

## Where We Are

Chapter 1 described a web application as cooperating processes on several machines. Chapter 2 showed what one machine is made of: CPU cores that run instructions, private caches per core kept consistent by a coherence protocol (MESI, Section 2.3), and main memory that is far slower than the caches. Chapter 3 introduced the operating system's view of execution: a **process** owns an address space, a **thread** is a schedulable stream of instructions inside a process, all threads of one process share its memory (Section 3.7), and the scheduler can stop any thread at any instruction and run another one (Section 3.10). Chapter 3 also listed the threading models (kernel threads, green threads, M:N, coroutines; Section 3.8). Chapter 4 showed how a thread waits for I/O, and how one thread can watch thousands of sockets with `epoll` (Section 4.5).

Those chapters left a dangerous gap. They told you that threads share memory and can be interrupted at any point, but not what happens when two of them touch the same data. This chapter fills that gap. It explains why `count = count + 1` can lose updates, why a thread can loop forever on a flag that another thread set long ago, how the hardware and the compiler reorder your instructions, and what contract (a **memory model**) lets you write correct code anyway. It then builds the toolbox: atomic instructions, locks, semaphores, condition variables, and the classic problems they solve. It covers the ways concurrent programs stop making progress (deadlock, livelock, starvation, priority inversion) and how to prevent, avoid and detect them. Finally it covers the higher-level tools a backend engineer uses every day (thread pools, futures, async/await, event loops, virtual threads, goroutines) and the Global Interpreter Lock in Python.

Every backend server is a concurrent program. Two requests for the last item in stock, two writes to an in-memory cache, a shutdown flag set by a signal handler, a pool of workers taking jobs from a queue: each is a concurrency problem, and each can fail in ways that never show up in a test on a laptop and appear only under production load.

**After this chapter you can:**
- distinguish concurrency from parallelism and explain why backend software is concurrent even on one core;
- identify read-modify-write and check-then-act race conditions and draw the interleaving that breaks them;
- state the three requirements of a correct critical-section solution;
- explain atomicity, visibility and ordering in terms of store buffers, caches and compiler optimisations, and use happens-before to reason about correctness;
- explain how compare-and-swap works, build a counter and a spinlock from it, and describe the ABA problem;
- choose between a mutex, a spinlock, a read–write lock, a semaphore and a condition variable, and explain the misuse pattern of each;
- solve producer–consumer, readers–writers and dining philosophers in pseudocode;
- state the four Coffman conditions and apply prevention, avoidance (Banker's algorithm), detection (wait-for graph) and recovery;
- explain why lock-free does not automatically mean faster;
- size a thread pool for CPU-bound and I/O-bound work and choose a rejection policy;
- explain futures, continuations and async/await, and compare thread-per-task, event-loop, M:N and actor execution models;
- explain what the GIL is and what it does and does not protect;
- diagnose deadlocks, contention and races with thread dumps, profilers, race detectors and stress tests.

**Interview questions this chapter prepares you for:**
1. What is the difference between concurrency and parallelism?
2. What is a race condition? How do you prevent one?
3. What is a critical section, and what must a solution to the critical-section problem guarantee?
4. What is a memory model? What is a visibility problem? What does `volatile` do in Java?
5. What is happens-before?
6. What is compare-and-swap? What is the ABA problem?
7. What is the difference between a mutex and a semaphore? Between a mutex and a spinlock?
8. What is a condition variable, and why must you wait in a `while` loop?
9. How do you solve the producer–consumer problem?
10. What are the four conditions for deadlock? How do you prevent, avoid and detect deadlock?
11. What is the difference between deadlock, livelock and starvation? What is priority inversion?
12. What does lock-free mean? Is lock-free always faster?
13. How do you size a thread pool?
14. What is the difference between async/await and threads? How does an event loop work?
15. What is the GIL, and what are its consequences?

**Scope.** Here: concurrency inside one process (and, briefly, between processes on one machine): races, the memory model, atomics, locks, semaphores, condition variables, classic problems, liveness failures, lock-free structures, thread-safety strategies, thread pools, futures, execution models, and testing and diagnosis. Processes, threads, threading models and the scheduler were taught in Chapter 3 and are only recalled; `epoll` and the I/O models are in Chapter 4. How a *server* uses these tools (choosing an execution model, sizing pools with Little's Law, request-scoped state, overload and backpressure, coordination across instances) is Chapter 11. Concurrency control *inside a database* (transactions, isolation levels, row locks, MVCC, database deadlocks) is Chapter 17. Locks that span machines (distributed locks and fencing tokens) are in Chapter 23.

---

## 5.1 Concurrency and Parallelism

### Definition

**Concurrency** is the property of a program that has several tasks in progress during the same period of time, whose steps may be interleaved in any order. **Parallelism** is the simultaneous execution of several computations at the same instant, on different hardware (cores, processors or machines). Concurrency is about the *structure* of a program (dealing with many things at once); parallelism is about its *execution* (doing many things at once).

### Why the distinction exists

The two words are often used interchangeably, and the confusion leads to wrong expectations. A program with 200 threads on a single-core machine is highly concurrent and not parallel at all: the scheduler switches between threads (Chapter 3, Section 3.11), so at any instant exactly one runs. A program that splits a matrix multiplication across 8 cores is parallel; if the 8 pieces never interact, it barely needs concurrency control. Most backend servers are both: hundreds of concurrent requests, executed in parallel on a few cores.

The distinction matters because **correctness problems come from concurrency, not from parallelism**. A race condition (Section 5.2) can occur on a single core, because the scheduler can preempt a thread between any two instructions. Adding cores makes races *more frequent* and adds new problems (visibility, Section 5.3), but it does not create the basic problem.

### Mechanism: two timelines

```text
Concurrency without parallelism (1 core, 3 tasks):

  core 0 | A A | B B B | A | C C | B | A A | C |
         +----------------------------------------> time
  At every instant one task runs. All three are "in progress" across the window.

Parallelism (3 cores, 3 tasks):

  core 0 | A A A A A A A A |
  core 1 | B B B B B B     |
  core 2 | C C C C C C C   |
         +-----------------> time
  Tasks run at the same instant.

Both (2 cores, 4 tasks):

  core 0 | A A | C C C | A A | D |
  core 1 | B B B | D D | B | C C |
```

### Why software is concurrent

There are three distinct reasons to structure a program as several concurrent tasks, and they lead to different designs.

1. **Overlapping I/O with computation (latency hiding).** Chapter 2 gave the numbers: a main-memory access takes on the order of 100 ns, a round trip within a data centre on the order of 0.5 ms, an SSD read on the order of 100 µs. A request handler that queries a database spends most of its wall-clock time *waiting*. If the server could only handle one request at a time, the CPU would sit idle during every wait. With concurrency, while request 1 waits for the database, request 2 can run.

   *Worked example.* Assume a handler uses 2 ms of CPU and then waits 48 ms for a database. Serving one request at a time, the server finishes one request per 50 ms (20 per second) and the CPU is busy 2/50 = 4% of the time. With 25 requests in progress at once on one core, the core can in principle be kept fully busy: 1 core / 2 ms of CPU each = 500 requests per second, a 25× increase, *with no additional hardware*. That gain comes entirely from concurrency, and none from parallelism.

2. **Using multiple cores (throughput for CPU-bound work).** Since the mid-2000s, single-core speed has improved slowly while core counts have grown (Chapter 2, Section 2.1). A CPU-bound computation can run faster only if it is split into parts that run in parallel. Concurrency is the structure; parallelism is the payoff.

3. **Responsiveness and isolation.** A program that does one thing at a time cannot respond while it works. A user interface must keep responding while a file downloads; a server must accept new connections while old ones are being processed; a health-check endpoint must answer while a slow report is being generated. Separate tasks let short work avoid waiting behind long work. (This is the same "convoy effect" that FCFS scheduling suffers from, Chapter 3, Section 3.10.)

### Guarantees and non-guarantees

Concurrency guarantees nothing about the *order* in which the steps of different tasks happen. Unless you add synchronisation (the rest of this chapter), any interleaving that respects each task's own program order is possible, and the one you see in testing is not necessarily the one you get in production. Parallelism does not guarantee a speed-up: Amdahl's law says that if a fraction *s* of the work is serial, the speed-up on *N* cores is at most 1 / (s + (1 − s)/N). With 10% serial work (*s* = 0.1), 8 cores give at most 1 / (0.1 + 0.9/8) ≈ 4.7×, and infinitely many cores give at most 10×. Locks (Section 5.5) are a common source of that serial fraction.

### Failure and diagnosis

The typical failure of the concurrency-for-I/O argument is an unbounded number of concurrent tasks. Each task costs memory (a thread stack, buffers), and each one usually holds or waits for a downstream resource (a database connection). Past some point, more concurrency increases latency without increasing throughput. The symptom is rising latency and memory with flat throughput; the evidence is the number of in-flight tasks (threads in a **thread dump**, which is a snapshot of every thread's state and current stack, Section 5.14; or the number of pending tasks) against throughput. The capacity model for this, Little's Law, is in Chapter 11.

The typical failure of the parallelism argument is a program that uses many threads but is limited by one lock or one shared resource, so that CPU usage never exceeds about one core regardless of the core count. The evidence is per-thread CPU (`top -H`, Chapter 3, Section 3.19) showing threads mostly blocked, and a lock-contention profile (Section 5.14).

### Backend connection

A web server is concurrent for reason 1 (requests spend most of their time waiting on databases and other services), parallel for reason 2 (several cores), and separates tasks for reason 3 (health checks and admin endpoints must not queue behind slow requests). Every execution model in Section 5.13 is a different answer to the question "how do we keep many requests in progress cheaply?"

---

## 5.2 Shared State and the Race Condition

### Definition

**Shared mutable state** is data that more than one concurrent task can access, where at least one of them writes it. A **race condition** is a bug in which the correctness of a program depends on the relative timing or interleaving of concurrent operations on shared state, so that some interleavings produce a wrong result. A **critical section** is a section of code that accesses shared state and must not be executed by more than one task at a time if the program is to be correct.

A related but distinct term: a **data race** is two accesses to the same memory location from different threads, at least one of them a write, that are not ordered by synchronisation. Section 5.3 makes "ordered by synchronisation" precise. A data race is almost always a bug (in C and C++ it is undefined behaviour; Section 5.3); not every race condition is a data race (a check-then-act race between two properly locked operations is a race condition with no data race).

### Why it exists

Threads of one process share the heap and global variables (Chapter 3, Section 3.7). That is the point of threads: they can cooperate on shared data without copying it. The price is that the scheduler can stop a thread between *any* two machine instructions, and on a multicore machine two threads can execute instructions at literally the same moment. A single line of source code is usually several instructions. If two threads execute those instructions on the same data, the instructions of one can fall between the instructions of the other.

### Mechanism 1: the read-modify-write race (`count++`)

The statement `count = count + 1` looks indivisible. To a CPU it is three steps:

```pseudo
r = load(count)     // read the current value from memory into a register
r = r + 1           // modify the private register
store(count, r)     // write the register back to memory
```

The register is private to each thread (each thread has its own saved register set, Chapter 3, Section 3.11). If thread T1 is preempted after the load, or if T2 runs the same three steps on another core at the same moment, both threads can read the same old value.

**Interleaving table: a lost update.** `count` starts at 0. Each thread increments it once. The correct result is 2.

| Step | Thread T1 | Thread T2 | T1's register | T2's register | `count` in memory |
|---|---|---|---|---|---|
| 1 | `r = load(count)` | | 0 | – | 0 |
| 2 | | `r = load(count)` | 0 | 0 | 0 |
| 3 | `r = r + 1` | | 1 | 0 | 0 |
| 4 | | `r = r + 1` | 1 | 1 | 0 |
| 5 | `store(count, r)` | | 1 | 1 | **1** |
| 6 | | `store(count, r)` | 1 | 1 | **1** |

Final value: 1. T2's store overwrote T1's, and one increment was lost. This is the **lost update** anomaly. (Databases have the same anomaly between transactions, Chapter 17.)

**The interleaving that happens to work:**

| Step | Thread T1 | Thread T2 | `count` |
|---|---|---|---|
| 1 | `r = load(count)` → 0 | | 0 |
| 2 | `r = r + 1` → 1 | | 0 |
| 3 | `store(count, r)` | | 1 |
| 4 | | `r = load(count)` → 1 | 1 |
| 5 | | `r = r + 1` → 2 | 1 |
| 6 | | `store(count, r)` | 2 |

The bug is not that the code always fails; it is that it fails *for some interleavings*. If each thread increments a million times, the window between load and store is open a million times per thread, and lost updates become very likely. A typical demonstration (two threads, a million increments each, no synchronisation) prints a different final value on every run, usually well below two million. On a single core, the loss occurs only when a preemption happens to fall inside the three-instruction window, so it is rarer; it is still not zero.

### Mechanism 2: the check-then-act race

The second pattern is a decision based on a value that may change before the action is taken:

```pseudo
// Sell one unit if any are in stock
if stock > 0:            // check
    stock = stock - 1    // act
    confirm_order()
```

**Interleaving table: overselling the last unit.** `stock` starts at 1.

| Step | Request A (thread T1) | Request B (thread T2) | `stock` |
|---|---|---|---|
| 1 | `stock > 0`? 1 > 0, yes | | 1 |
| 2 | | `stock > 0`? 1 > 0, yes | 1 |
| 3 | `stock = stock - 1` | | 0 |
| 4 | `confirm_order()` | | 0 |
| 5 | | `stock = stock - 1` | **−1** |
| 6 | | `confirm_order()` | −1 |

Two customers were told they bought the last item. The check was true when it was made, and false by the time B acted on it. This shape appears everywhere:

- **Lazy initialisation:** `if instance == null: instance = new Thing()`. Two threads both see `null` and create two instances.
- **Put-if-absent on a map:** `if not map.contains(k): map.put(k, v)`. Even if the map itself is thread-safe, the *pair* of calls is not atomic.
- **File creation:** `if not exists(path): create(path)`. Another process creates the file between the two calls. (This is called a **TOCTOU** bug: time-of-check to time-of-use. With files it can be a security hole. The fix is an atomic operation such as `open` with `O_CREAT | O_EXCL`, which fails if the file exists.)

The general rule: **an operation that must be atomic must be performed as one indivisible step, or protected so that no other task can observe or modify the state between its parts.**

### The critical-section problem

The textbook framing (due to Dijkstra) is: several threads repeatedly execute code of the form

```pseudo
loop:
    entry_section()       // ask permission
    critical_section()    // touch shared state
    exit_section()        // announce we are done
    remainder_section()   // everything else
```

Design `entry_section` and `exit_section` so that three requirements hold:

1. **Mutual exclusion.** At most one thread is in its critical section at any time.
2. **Progress.** If no thread is in its critical section and some threads want to enter, the choice of which enters next is made only among those wanting to enter, and cannot be postponed forever. (A thread in its remainder section cannot block others from entering. A rule like "strictly alternate T1, T2, T1, T2" violates progress: if T2 never wants to enter again, T1 waits forever.)
3. **Bounded waiting.** After a thread requests entry, there is a limit on how many times other threads may enter their critical sections before it is admitted. (This rules out starvation of one thread, Section 5.8.)

A solution must also make no assumption about the relative speed of threads or the number of CPUs.

**A software-only solution: Peterson's algorithm (two threads).**

```pseudo
shared flag[2] = {false, false}   // flag[i]: thread i wants to enter
shared turn                       // whose turn it is to wait

enter(i):
    j = 1 - i
    flag[i] = true                // I want in
    turn = j                      // but you go first if you also want in
    while flag[j] and turn == j:  // wait while the other wants in and it is its turn
        spin

exit(i):
    flag[i] = false
```

Why it works: if both threads want to enter, both write `turn`; whichever wrote it *last* waits, because the other thread's condition becomes false. Mutual exclusion holds; progress holds (a thread not wanting to enter has `flag = false` and cannot block); bounded waiting holds (after one entry by the other thread, the other thread's next `turn = j` lets the waiter in).

Peterson's algorithm is correct under the assumption that every thread sees every memory write in the order it was executed. **On modern hardware it is broken as written.** Section 5.3 shows that x86 processors can let thread *i* read `flag[j]` *before* its own write `flag[i] = true` is visible to the other core, so both threads can see the other's flag as false and both enter. That failure is the motivation for memory models and for hardware atomic instructions (Section 5.4), which are what real locks are built from.

### Guarantees and non-guarantees

Making individual memory accesses safe does not make a *sequence* of them safe. A thread-safe map guarantees that each single call (`get`, `put`) behaves atomically; it does not guarantee that `get` followed by `put` is atomic (check-then-act). A lock (Section 5.5) guarantees mutual exclusion only among code that *uses the same lock*: one path that forgets to take the lock reintroduces the race. Nothing in this section protects state shared across processes or machines; an in-process lock protects nothing when two instances of a service handle requests for the same item.

### Failure and diagnosis

Race conditions have recognisable symptoms:

- **Counters that drift low** (metrics, rate limiters, in-memory statistics): lost updates.
- **Invariants that are violated** "impossibly": negative stock, a balance that does not match the sum of transactions, a list whose size field disagrees with its contents.
- **Corrupted data structures:** a hash map whose internal arrays were resized by two threads at once can lose entries, contain cycles (in some older map implementations, an unsynchronised concurrent resize could create a cycle in a bucket's linked list, so that a later `get` looped forever at 100% CPU), or throw unexpected exceptions such as index-out-of-bounds.
- **Non-reproducibility:** the bug appears under load, disappears when you add logging (logging often adds synchronisation and changes timing), and never appears in a single-threaded unit test. This kind of bug is nicknamed a **Heisenbug**.

Evidence: a stress test that runs the operation from many threads and checks the invariant afterwards; a race detector (Section 5.14); or a code review that asks of every piece of shared mutable state, "what protects this, and does every access path use it?"

### Backend connection

Inside one server process, request handlers run concurrently and share everything that is not local to the request: singletons, caches, connection pools, counters, configuration objects. Any of these that is mutable needs a protection strategy (Section 5.10). Two points are worth fixing in your mind now:

- The stock example is usually *not* solved by an in-process lock in a real backend, because several server instances run behind a load balancer (Chapter 1, Section 1.5), each with its own memory. The fix belongs in the shared system of record: an atomic `UPDATE items SET stock = stock - 1 WHERE id = ? AND stock > 0` and a check of the affected row count, or a transaction with appropriate isolation (Chapter 17). The *reasoning* (find the check, find the act, make them one atomic step) is the same.
- Request-scoped state (data that belongs to one request) is safe precisely because it is not shared. How servers keep it that way is in Chapter 11.

---

## 5.3 Atomicity, Visibility, and Ordering: The Memory Model

Section 5.2 assumed that memory behaves like one big array: each write instantly visible to everyone, in the order the program wrote it. Real hardware and real compilers do not behave that way, and the difference produces a second family of bugs that look impossible from the source code.

### Definition

Correct sharing between threads needs three properties:

- **Atomicity:** an operation happens entirely or not at all, with no other thread able to observe an intermediate state. (`count++` is not atomic, Section 5.2.)
- **Visibility:** a write made by one thread eventually becomes observable to reads in other threads, and you know *when* it is guaranteed to be.
- **Ordering:** other threads observe a thread's memory operations in an order consistent with what the program requires.

A **memory model** is the contract, defined by a hardware architecture or a programming language, that specifies which values a read of shared memory is allowed to return in a multithreaded program. It tells the programmer what they may rely on, and tells compiler writers and hardware designers which optimisations they may perform.

### Why it exists: the hardware and the compiler both reorder

Two kinds of optimisation, each of which makes single-threaded programs much faster, break the "one big array" picture.

**1. Hardware: store buffers and the cache hierarchy.** Recall from Chapter 2 (Section 2.3) that each core has private caches, and that a coherence protocol (MESI) ensures that all cores agree on the value of each individual cache line. Coherence alone would make visibility simple, but before a write even reaches the cache, it goes somewhere else.

A write to a cache line that this core does not currently own requires a message to the other cores ("invalidate your copy") and waiting for their replies, which can take tens to hundreds of cycles. Rather than stall, each core puts the write into a small private queue called the **store buffer** and continues executing. The write drains from the store buffer into the cache (and becomes visible to other cores) later. Meanwhile, the same core's *later loads* can complete from the cache. Its own loads check its own store buffer first, so a thread always sees its own writes; other cores cannot see into it.

```text
        Core 0                                 Core 1
  +--------------------+                 +--------------------+
  |  executing thread  |                 |  executing thread  |
  |   T0: x = 1        |                 |   T1: y = 1        |
  |   T0: r1 = y       |                 |   T1: r2 = x       |
  +---------+----------+                 +---------+----------+
            | stores                               | stores
  +---------v----------+                 +---------v----------+
  |  store buffer      |                 |  store buffer      |
  |  [x = 1] (pending) |                 |  [y = 1] (pending) |
  +---------+----------+                 +---------+----------+
            | drains later                         | drains later
  +---------v----------+   coherence     +---------v----------+
  |  L1 cache          |<--------------->|  L1 cache          |
  |  y = 0             |   (MESI)        |  x = 0             |
  +--------------------+                 +--------------------+
                    \                       /
                     +---- shared L3 / RAM -+   x = 0, y = 0
```

In the picture, each core's write is still in its store buffer when its next instruction (a load of the *other* variable) reads from the cache. Both loads return 0. From the outside, it looks as if each core executed its load *before* its store: a **store→load reordering**.

Some architectures go further. On ARM and POWER processors, stores can drain from the store buffer out of order, loads can be satisfied out of order, and invalidation messages can be queued (in an **invalidate queue**) so that a core briefly reads a stale cached value. The result is that almost any pair of independent memory operations may appear reordered to another core.

**2. The compiler (and the JIT).** Compilers transform code as long as the result is the same *for a single thread* (the **as-if-serial** rule). Typical transformations that change what other threads see:

- **Keeping a value in a register.** A variable read in a loop is loaded once into a register and the register is reused, instead of reloading from memory on every iteration.
- **Hoisting a load out of a loop.** `while (!stop) { work(); }` where `work()` does not visibly change `stop` may become `if (!stop) { while (true) { work(); } }`.
- **Reordering independent stores or loads.** `a = 1; b = 2;` may be emitted as `b = 2; a = 1;`.
- **Eliminating or merging stores.** Two writes to the same variable in a row may become one.

Each of these is invisible to the thread performing them. Each can be catastrophic for another thread that relies on the order or timing of those writes.

Without a memory model, programmers could not write correct concurrent code without knowing exactly which compiler, which optimisation level and which CPU would run their program. With one, they write against a contract and the toolchain is responsible for honouring it.

### Mechanism: two litmus tests

Memory-model behaviour is usually described with tiny programs called **litmus tests**. Two matter most.

**Store buffering (SB).** Initially `x = 0, y = 0`.

```pseudo
Thread 1:          Thread 2:
x = 1              y = 1
r1 = y             r2 = x
```

Can the program end with `r1 == 0 and r2 == 0`? Under the "one big array" model, no: whichever thread's store comes first, the other thread's load must see it. On real x86 and ARM hardware, **yes**, via the store-buffer mechanism above. This is exactly why Peterson's algorithm (Section 5.2) fails: `flag[i] = true` is the store, and the read of `flag[j]` is the load.

**Message passing (MP).** Initially `data = 0, ready = false`.

```pseudo
Thread 1 (producer):     Thread 2 (consumer):
data = 42                if ready:
ready = true                 print(data)
```

Can Thread 2 see `ready == true` and print 0? On x86, the hardware does not reorder store→store or load→load, so the hardware will not cause it; but the compiler may reorder the two stores or the two loads, so the answer for a program in a high-level language is still **yes, unless you synchronise**. On ARM and POWER, the hardware itself can produce it.

| Reordering (as seen by another core) | x86 / x86-64 (TSO) | ARMv8 / POWER |
|---|---|---|
| store → later load (SB) | allowed | allowed |
| store → later store (MP, producer side) | not allowed | allowed |
| load → later load (MP, consumer side) | not allowed | allowed |
| load → later store | not allowed | allowed |

x86's model is called **total store order (TSO)**: there is one global order of stores that all cores agree on, and the only visible relaxation is the store buffer. ARM and POWER have **weak** (relaxed) memory models. A program that "works" on an x86 laptop can fail on an ARM server; with ARM-based cloud instances now common, this is a practical concern, not a theoretical one. And regardless of hardware, the compiler can reorder anything the language model allows.

### Sequential consistency and the programmer's contract

The intuitive model has a name. A system is **sequentially consistent (SC)** (Lamport, 1979) if the result of any execution is the same as if all threads' operations were executed in some single global order, and each thread's operations appear in that order in program order. SC is what you imagine when you draw an interleaving table.

Enforcing SC for every memory access would forbid most hardware and compiler optimisations and make all code slower. Modern languages (Java since 2004, C and C++ since 2011, Go explicitly since its 2022 memory-model revision) make a different deal, known as **DRF-SC** (data-race-free implies sequential consistency):

> If your program has no data races, it will behave as if it were sequentially consistent.

So the programmer's job is not to understand store buffers and invalidate queues. It is to make sure that every pair of conflicting accesses to shared memory (same location, different threads, at least one write) is **ordered by synchronisation**. The tool for stating "ordered by synchronisation" precisely is happens-before.

### Happens-before

**Happens-before** is a partial order on the operations of a program, defined by the memory model, such that if operation A happens-before operation B, then the effects of A (and of everything that happened-before A) are guaranteed to be visible to B, and A appears to occur before B. If two conflicting accesses are not ordered by happens-before in either direction, they form a **data race**.

Happens-before is built from a small number of rules. The exact list is language-specific, but the core is the same everywhere:

1. **Program order.** Within one thread, each operation happens-before every later operation of that thread.
2. **Synchronises-with edges** between threads, created by synchronisation operations:
   - **Lock:** an unlock of a mutex happens-before every later lock of *the same* mutex.
   - **Atomic/volatile variable:** a write to an atomic variable (with release semantics) happens-before a read of that variable (with acquire semantics) that sees the written value.
   - **Thread start:** the action that starts a thread happens-before everything that thread does.
   - **Thread join:** everything a thread does happens-before another thread's successful return from waiting for it to finish (`join`).
   - **Channel/queue hand-off:** in languages and libraries that provide them, putting a message into a channel or concurrent queue happens-before the receipt of that message.
3. **Transitivity.** If A happens-before B and B happens-before C, then A happens-before C.

Transitivity is what makes happens-before useful: one synchronising edge carries *everything before it* across to the other thread.

```text
Thread 1                                   Thread 2
--------                                   --------
data = 42          (a)
config = loaded    (b)
lock(m)
  ready = true     (c)
unlock(m) ----------------- synchronises-with ---------> lock(m)
                                                           if ready:  (d)
                                                             use(data, config)  (e)
                                                         unlock(m)

(a) -> (b) -> (c) -> unlock     program order in Thread 1
unlock -> lock                  lock rule
lock -> (d) -> (e)              program order in Thread 2
therefore (a) and (b) happen-before (e): Thread 2 sees data = 42 and the loaded config
```

Notice that `data` and `config` were not written inside the lock, yet they are safely visible, because they were written before the unlock in program order. That is called **safe publication**: making an object visible to other threads only through a synchronising action, so that its contents are visible too.

### Memory barriers

How does the toolchain honour happens-before? With **memory barriers** (also called **fences**): instructions that restrict how memory operations may be reordered around them, together with compiler rules that stop the compiler moving loads and stores across them.

Barriers are commonly classified by which pair of operations they keep in order: **LoadLoad**, **LoadStore**, **StoreStore** and **StoreLoad**. A StoreLoad barrier is the expensive one because it must wait for the store buffer to drain (x86 needs it explicitly, with `MFENCE` or with any `LOCK`-prefixed instruction; the other three are free on x86 because TSO already provides them).

Two combined forms are what programmers use, usually implicitly:

- An **acquire** operation (a lock, an atomic read with acquire semantics) prevents later loads and stores in that thread from being moved *before* it. Nothing after the acquire can "leak out" above it.
- A **release** operation (an unlock, an atomic write with release semantics) prevents earlier loads and stores from being moved *after* it. Nothing before the release can "leak out" below it.

```text
     ... earlier operations ...          may not move down past the release
  ---------- release (unlock / atomic store) ----------
  ---------- acquire (lock / atomic load) -------------
     ... later operations ...            may not move up past the acquire
```

A release-write paired with an acquire-read of the same variable creates exactly the synchronises-with edge from the happens-before rules. A **full fence** (sequentially consistent) also orders the release's stores before the acquire's loads (StoreLoad), which is what Peterson's algorithm needs.

Application programmers almost never write fence instructions directly. They use locks, atomic variables and concurrent library classes, which contain the right fences. The point of learning about barriers is to understand *why* those tools are needed and what they cost.

### Concrete example 1: the stop flag that is never seen

```pseudo
shared stop = false            // plain variable: no synchronisation

worker thread:
    while not stop:
        process_next_item()    // assume the compiler can see this never writes 'stop'
    print("worker exiting")

main thread:
    start(worker)
    sleep(1 second)
    stop = true                // ask the worker to stop
    join(worker)               // wait for it: this hangs forever
```

What goes wrong, layer by layer:

1. **The compiler or JIT** sees that nothing inside the loop writes `stop`. Under the as-if-serial rule it may load `stop` into a register once and never reload it, transforming the loop into `if not stop: loop forever`. This is the usual cause of a *permanent* hang. In JIT-compiled languages it often appears only after the loop has run long enough to be optimised (Chapter 2, Section 2.5), so the program works in a short test and hangs in production.
2. **The store buffer** may delay the visibility of `stop = true` to the worker's core. On its own this is only a short delay (store buffers drain in nanoseconds to microseconds, and the cache coherence protocol then propagates the new value), so hardware alone does not usually cause a permanent hang; it causes the "how long until I see it?" uncertainty that the memory model removes.
3. **The memory model's verdict:** the write `stop = true` and the read `while not stop` are conflicting accesses not ordered by happens-before. This is a data race, and the language gives you no guarantee that the worker ever sees the write.

**The fix** is to make `stop` a synchronisation variable: an atomic (or, in Java, `volatile`) variable.

```pseudo
shared atomic stop = false

worker thread:
    while not atomic_load_acquire(stop):    // must actually re-read memory each time
        process_next_item()

main thread:
    atomic_store_release(stop, true)
```

Now the compiler must reload `stop` on every iteration (it is an atomic load, which it may not cache in a register), the hardware ordering is enforced by the acquire/release semantics, and the write happens-before any read that observes it. If `main` also wrote, say, `reason = "shutdown"` before setting `stop`, the worker is guaranteed to see that too.

Note what the fix did *not* do: it did not add a lock, and it did not make anything "atomic" in the read-modify-write sense. Visibility and atomicity are separate problems.

### Concrete example 2: publishing an object

```pseudo
shared config = null      // plain reference

loader thread:
    c = new Config()
    c.timeout_ms = 500
    c.max_retries = 3
    config = c            // publish

request thread:
    if config != null:
        use(config.timeout_ms)   // may see 0!
```

This is the message-passing litmus test with objects. The writes of the fields and the write of the reference are independent stores; the compiler or a weakly ordered CPU may make the reference visible before the fields. The request thread then reads a non-null reference to an object whose fields still hold their default values. The fix is the same: publish through a synchronisation action (an atomic/volatile reference, a lock, a concurrent map, or a channel), or make the object immutable with the language's special guarantees for immutable fields (see the Java lens).

### Java lens

> **Java lens:** The Java Memory Model (JLS chapter 17) defines happens-before for Java.
> - A `volatile` field: every read sees the most recent write in the happens-before order; a write to a volatile field happens-before every subsequent read of that field. The JIT may not cache it in a register, and it emits the needed barriers. `volatile` gives **visibility and ordering, not atomicity of compound actions**: `volatileCount++` is still a lost-update race. Use `AtomicInteger.incrementAndGet()` (Section 5.4) for that.
> - `synchronized`: entering a `synchronized` block or method locks the object's built-in **monitor**; leaving it unlocks it. An unlock happens-before every later lock of the same monitor, so `synchronized` provides atomicity (mutual exclusion), visibility and ordering together, for code that uses the same object.
> - `Thread.start()` happens-before the started thread's actions; a thread's actions happen-before another thread's return from `join()` on it.
> - `final` fields: if a constructor does not let `this` escape, other threads that obtain a reference to the object are guaranteed to see the correctly constructed values of its `final` fields, even without synchronisation. This is why immutable objects with `final` fields are safe to share.
> - Non-volatile `long` and `double` writes may be split into two 32-bit writes (JLS 17.7), so a reader could see half of one value and half of another (a **torn read**). Volatile `long`/`double` are always atomic.
> - **Double-checked locking** (`if (instance == null) { synchronized(lock) { if (instance == null) instance = new Thing(); } }`) is broken unless `instance` is `volatile`: the first, unsynchronised check can see a non-null reference to a not-yet-initialised object (concrete example 2). With `volatile` it is correct. A simpler correct alternative is the holder-class idiom, which relies on the JVM's thread-safe class initialisation.

> **Other runtimes:**
> - **C and C++:** since C11/C++11, `std::atomic<T>` (C++) and `_Atomic` (C) provide atomic variables, with operations taking an explicit **memory order**: `memory_order_seq_cst` (the default, sequentially consistent), `acquire`, `release`, `acq_rel` and `relaxed` (atomicity only, no ordering). A data race on a non-atomic variable is **undefined behaviour**: the compiler may assume it never happens, and anything can result. The C/C++ keyword `volatile` is *not* a threading tool; it is for memory-mapped hardware registers and gives no inter-thread ordering guarantees.
> - **Go:** the Go memory model also promises DRF-SC. Synchronisation comes from channels (a send happens-before the corresponding receive completes), `sync.Mutex`, `sync.WaitGroup`, `sync.Once` and the `sync/atomic` package. A racy Go program can do worse than read a stale value: interface values and slices are multi-word, so a racy read can see a torn value and crash. The Go proverb "Do not communicate by sharing memory; instead, share memory by communicating" expresses a preference for channels (Section 5.10).

### Guarantees and non-guarantees

| Tool | Atomicity | Visibility | Ordering |
|---|---|---|---|
| Plain variable | Usually for a single aligned word-sized read or write on common hardware, but the language may not promise it (torn reads of 64-bit values in Java, undefined behaviour in C++) | No guarantee | No guarantee |
| Atomic / `volatile` variable | Each single read or write; atomic read-modify-write methods (`incrementAndGet`, CAS) for compound updates | Yes | Acquire/release (or sequentially consistent) around the access |
| Lock (mutex) | Everything inside the critical section, relative to other code holding *the same* lock | Yes, for everything written before the unlock, to the next locker | Yes, via unlock → lock |

Non-guarantees worth stating out loud: atomicity of single accesses does not make a sequence atomic; `volatile`/atomic gives no mutual exclusion; and a happens-before edge exists only between operations on *the same* lock or *the same* atomic variable. Two threads that each lock a different mutex are not synchronised with each other at all.

### Failure and diagnosis

Memory-model bugs are among the hardest to find, because:

- they may never occur on x86 (TSO hides most reorderings) and appear only on ARM servers;
- they often appear only after JIT compilation, i.e. after warm-up under load;
- adding a `print` statement usually "fixes" them, because printing involves locks (output streams are synchronised) that create happens-before edges.

Symptoms: a thread that never notices a flag (hang on shutdown, a worker that never stops; a thread dump shows it RUNNABLE in a loop, not BLOCKED), an object observed with default field values (a null-reference error on a field that the constructor always sets), a counter that is stale for a long time. Evidence: the code review question "which happens-before edge orders this write before that read?"; for C, C++ and Go, a race detector (ThreadSanitizer, `go test -race`, Section 5.14) that reports the racing pair of accesses; for Java, a concurrency stress harness such as OpenJDK's jcstress, which runs litmus tests millions of times and counts the outcomes.

### Backend connection

Backend code is full of single-writer, many-reader state: a "draining" flag set during graceful shutdown (Chapter 4, Section 4.11) and read by every request handler; a configuration object reloaded periodically; a cached set of feature flags; a lazily created client. Each is a publication problem. The standard pattern is: build a new immutable object, then publish it with a single atomic reference write; readers read the reference once per request and use that snapshot. It needs no locks on the read path, and the happens-before edge on the reference makes every field visible.

---

## 5.4 Hardware Atomics: Test-and-Set, Compare-and-Swap, Fetch-and-Add

### Definition

An **atomic read-modify-write (RMW) instruction** is a single CPU instruction that reads a memory location, computes a new value and writes it back, such that no other core can access that location between the read and the write. The three classic forms are:

- **Test-and-set (TAS):** atomically write `true` to a location and return its old value.
- **Compare-and-swap (CAS)**, also called compare-and-exchange: atomically compare the location's current value with an expected value and, only if they are equal, replace it with a new value; report whether the swap happened (and usually the value seen).
- **Fetch-and-add (FAA):** atomically add a number to a location and return its old value.

### Why they exist

Section 5.2 showed that the critical-section problem can be solved with plain loads and stores only under assumptions that real hardware breaks, and even then only with awkward algorithms (Peterson's handles two threads). The hardware solution is to make the *read-modify-write itself* indivisible. Every lock, semaphore, atomic counter and concurrent data structure in every language ultimately rests on these instructions.

### Mechanism

Semantically, each is equivalent to the following function executed with no other memory access interleaved:

```pseudo
atomic test_and_set(addr):
    old = *addr
    *addr = true
    return old

atomic compare_and_swap(addr, expected, new):
    current = *addr
    if current == expected:
        *addr = new
        return true          // success
    return false             // failure: someone changed it first

atomic fetch_and_add(addr, delta):
    old = *addr
    *addr = old + delta
    return old
```

How the hardware makes it indivisible: on modern x86, a `LOCK`-prefixed instruction (`LOCK CMPXCHG` for CAS, `LOCK XADD` for fetch-and-add; `XCHG` with memory is implicitly locked and serves as test-and-set) obtains the cache line in the exclusive/modified state using the coherence protocol (Chapter 2, Section 2.3), then holds on to it, refusing to answer other cores' requests for that line until the read and write are both done. On x86 the locked instruction is also a full memory barrier.

ARM traditionally used a different mechanism: a pair of instructions called **load-linked / store-conditional (LL/SC)** (`LDXR`/`STXR`). Load-linked reads a value and starts monitoring its address; store-conditional writes only if no other core has written that address since, and reports failure otherwise. A CAS is built as a small LL/SC retry loop. ARMv8.1 added direct atomic instructions (the "Large System Extensions", including `CAS` and `LDADD`), which scale better under contention.

**Cost.** An uncontended atomic instruction on a line already in this core's cache commonly costs on the order of tens of cycles, more than a plain store but far less than a system call. Under contention the cost is dominated by the cache line bouncing between cores: every successful atomic operation by one core invalidates the line in all others (the same mechanism that makes false sharing slow, Chapter 2, Section 2.3). A single counter incremented by every core can become a serial bottleneck.

### Concrete: a CAS-based counter

```pseudo
shared atomic count = 0

increment():
    loop:
        old = atomic_load(count)
        new = old + 1
        if compare_and_swap(&count, old, new):
            return new
        // CAS failed: another thread changed count after we read it. Retry with the fresh value.
```

Replay the lost-update interleaving from Section 5.2 with this code:

| Step | Thread T1 | Thread T2 | `count` |
|---|---|---|---|
| 1 | `old = 0`, `new = 1` | | 0 |
| 2 | | `old = 0`, `new = 1` | 0 |
| 3 | `CAS(count, 0, 1)` → **success** | | 1 |
| 4 | | `CAS(count, 0, 1)` → **fails** (count is 1, not 0) | 1 |
| 5 | | retry: `old = 1`, `new = 2` | 1 |
| 6 | | `CAS(count, 1, 2)` → success | 2 |

The update is no longer lost: T2's attempt is rejected because the world changed under it, and it recomputes. This pattern (read, compute, CAS, retry on failure) is called a **CAS loop** and is the core idiom of **optimistic concurrency**: assume no conflict, detect it at the last moment, retry. (The same idea, applied to database rows with a version column, is optimistic locking in Chapter 17; applied to HTTP resources with `ETag`/`If-Match`, it is in Chapter 13.)

For a plain counter, fetch-and-add is better: `fetch_and_add(&count, 1)` always succeeds in one instruction, with no retry loop.

> **Java lens:** `java.util.concurrent.atomic` provides `AtomicInteger`, `AtomicLong`, `AtomicBoolean` and `AtomicReference<V>`. `incrementAndGet()` and `getAndAdd()` compile to fetch-and-add on x86; `compareAndSet(expected, new)` is CAS; `updateAndGet(f)` runs the CAS loop for you (so `f` must be side-effect free, because it may be called several times). Under heavy contention on one counter, `LongAdder` keeps several internal cells and has each thread add to a different one, summing them on read; writes stop contending, at the cost of a read that is not an atomic snapshot. `VarHandle` exposes per-access memory orderings, similar to C++'s.

### Concrete: a spinlock from test-and-set and CAS

A **spinlock** is a lock whose waiting threads repeatedly test the lock in a loop ("spin") instead of being put to sleep.

```pseudo
shared atomic locked = false

// Version 1: test-and-set
lock():
    while test_and_set(&locked) == true:
        // it was already true: someone else holds it; try again
        cpu_relax()             // e.g. x86 PAUSE: a hint that we are spinning
unlock():
    atomic_store_release(&locked, false)

// Version 2: the same with CAS
lock():
    while not compare_and_swap(&locked, false, true):
        cpu_relax()
```

Why it is correct: `test_and_set` returns `false` to exactly one thread (the one whose write changed `false` to `true`); every other thread sees `true` until the holder stores `false`. The atomic instruction gives mutual exclusion; acquire semantics on the successful TAS and release semantics on the unlock give the happens-before edge, so the next holder sees everything written in the critical section.

**The performance problem and the fix (test-and-test-and-set).** Every `test_and_set` is a write. With 8 threads spinning, each attempt pulls the cache line into the spinning core in exclusive state and invalidates it everywhere else, so the line bounces continuously between cores, slowing the lock holder too (it needs the line to release the lock). The standard improvement spins on a plain *read*, which can be served from each core's shared cached copy without traffic, and attempts the atomic write only when the lock looks free:

```pseudo
lock():
    loop:
        while atomic_load_relaxed(locked) == true:   // read-only spin: stays in local cache
            cpu_relax()
        if test_and_set(&locked) == false:           // looks free: now try to grab it
            return
        // lost the race to another thread; go back to read-only spinning
```

Adding **exponential backoff** (wait longer after each failed attempt) further reduces the stampede when the lock is released and every waiter tries at once. Queue-based spinlocks (ticket locks, MCS locks) go further, giving each waiter its own location to spin on and granting the lock in arrival order (fairness, Section 5.5). The Linux kernel uses queued spinlocks for this reason.

Section 5.5 explains when spinning is the right way to wait and when it is a disaster.

### The ABA problem

CAS answers the question "is the value still what I read?", not "has anything happened since I read it?". The **ABA problem** is the failure that occurs when a location changes from value A to B and back to A between a thread's read and its CAS: the CAS succeeds, although the state it was protecting has changed.

The classic victim is a lock-free stack (a **Treiber stack**), in which `top` points to the first node and `pop` is a CAS loop:

```pseudo
pop():
    loop:
        old_top = atomic_load(top)          // read A
        if old_top == null: return null
        next = old_top.next                 // read A's successor
        if compare_and_swap(&top, old_top, next):
            return old_top
```

**ABA timeline.** The stack is `top → A → B → C`.

```text
time  Thread T1                          Thread T2                         stack (top first)
----  ---------------------------------  --------------------------------  -----------------
 t1   old_top = A, next = B                                                 A → B → C
      (T1 is preempted here)
 t2                                      pop()  → returns A                 B → C
 t3                                      pop()  → returns B                 C
 t4                                      free(B); the memory of B is reused C
 t5                                      push(A) (A's node is recycled)     A → C
 t6   CAS(top, A, B): top == A? YES      (expected A, and top IS A again)
      top = B        → SUCCESS
 t7                                                                         B → ???  (B was freed!)
```

T1's CAS succeeded because `top` held the same address A as before, but the structure underneath changed: A's successor is now C, not B, and B has been freed. The stack now points into freed (or reused) memory. Values are equal; the *history* is different.

ABA is not limited to pointers. Any CAS on a value that can return to an earlier state while some meaning is attached to it can suffer from it: for example, a slot state that goes `FREE → BUSY → FREE` while a thread still holds an old reservation.

**Fixes:**

- **Version tags (stamped references).** Pair the value with a counter that increases on every change, and CAS the pair. A→B→A becomes (A,1)→(B,2)→(A,3); a CAS expecting (A,1) fails. Hardware support: x86-64's `CMPXCHG16B` (a 128-bit CAS) can swap a pointer and a counter together.
- **LL/SC**, where available, fails if the location was written at all, regardless of the value. (In practice implementations can fail spuriously, so code retries in a loop.)
- **Safe memory reclamation.** Many ABA cases come from freeing and reusing a node while another thread still holds a pointer to it. **Hazard pointers** (each thread publishes the pointers it is using, and a node is freed only when no thread has published it) and **epoch-based reclamation** (nodes are freed only after every thread has passed a checkpoint) prevent the reuse.
- **Garbage collection.** In a garbage-collected language, a node cannot be freed and reused while T1 still holds a reference to it, so the pointer form of ABA in the timeline above cannot happen. The value form (a counter or state that genuinely returns to an earlier value) still can.

> **Java lens:** `AtomicStampedReference<V>` pairs a reference with an `int` stamp and CASes both; `AtomicMarkableReference<V>` pairs it with a boolean.

### Guarantees and non-guarantees

An atomic RMW instruction guarantees that the read and write of *one* memory location (of the instruction's width, suitably aligned) are indivisible and, depending on the instruction and the language's memory ordering, that it orders surrounding accesses. It does **not** make two separate atomic operations atomic together (incrementing two counters atomically needs a lock or a single wider atomic), does not guarantee that a CAS loop ever succeeds (under contention a thread can keep losing; Section 5.9), does not detect ABA, and does not provide fairness.

### Failure and diagnosis

- **Spinlock CPU burn:** threads at 100% CPU doing no useful work. A CPU profile shows time inside the lock's spin loop (or `PAUSE` instructions); `perf top` shows a hot loop in a lock function.
- **CAS contention collapse:** throughput of a shared atomic counter falls as you add cores. A CPU profile shows the time in the CAS loop; hardware counters (`perf stat`) show many cache misses on one line. Fix: per-thread or striped counters (Section 5.9), or batch updates.
- **ABA:** crashes or corrupted lists in hand-written lock-free code in languages without garbage collection, usually rare and load-dependent. Evidence comes from memory-safety tools such as AddressSanitizer (which reports use-after-free) or from model checking. The practical prevention is to use a well-tested library rather than writing your own.

### Backend connection

Hardware atomics appear in backend code in four common forms: request and error counters for metrics; sequence and ID generators within a process; reference-swapped configuration (Section 5.3); and the internals of every lock, connection pool and concurrent queue your framework uses. The CAS idea (detect conflicting change and retry) reappears at every level of the stack: database optimistic locking with version columns (Chapter 17), conditional HTTP requests with `If-Match` (Chapters 7 and 13), and conditional writes in key-value stores (Chapter 23).

---

## 5.5 Locks

### Definition

A **lock** is a synchronisation object that enforces mutual exclusion: a thread **acquires** (locks) it before entering a critical section and **releases** (unlocks) it on leaving, and while one thread holds it, other threads that try to acquire it must wait. A **mutex** (from *mutual exclusion*) is a lock with an **owner**: only the thread that acquired it may release it. Chapter 2 (Section 2.9) gave the working definition; this section explains how a mutex works, how it waits, and the variations (spinning vs blocking, reentrant, fair, read–write) you have to choose between.

### Why it exists

Atomic instructions (Section 5.4) make one operation on one memory word indivisible. Most invariants span several words: a linked list's `head` and `size`, an account's `balance` and its list of transactions, a cache's map and its least-recently-used list. Keeping them consistent with CAS alone needs specialised lock-free algorithms (Section 5.9), which are hard to write and to prove correct. A lock lets you write ordinary sequential code inside the critical section and know that no other thread using the same lock can observe or modify the state in the middle.

```pseudo
shared lock m
shared list items
shared int size

add(x):
    lock(m)
    items.append(x)      // several memory writes...
    size = size + 1      // ...that must appear together
    unlock(m)
```

A lock also provides visibility and ordering: an unlock happens-before the next lock of the same mutex (Section 5.3), so everything written inside one critical section is visible to the next thread that enters.

### Mechanism: how to wait

What a lock does when the lock is already held is the most important design decision.

**Spinning.** The waiting thread loops, re-checking the lock (Section 5.4). It wastes CPU while waiting, but it notices the release within nanoseconds and never pays for a context switch.

**Blocking (sleeping).** The waiting thread asks the kernel to put it to sleep on a **wait queue** associated with the lock; it moves from the running state to the blocked state (Chapter 3, Section 3.4) and uses no CPU. When the holder releases the lock, it wakes one waiter, which the scheduler later runs. Blocking costs at least two context switches and a system call or two, commonly a few microseconds in total, plus the latency until the scheduler runs the woken thread (Chapter 3, Section 3.11).

Rule of thumb for choosing:

| Situation | Better waiting strategy |
|---|---|
| Critical section is a few dozen instructions, threads are on different cores, holder is never descheduled while holding | Spin |
| Critical section may take microseconds or longer, may do I/O, may page-fault | Block |
| Single core | Never spin: the holder cannot run (and release) while you spin |
| User-space code on a general-purpose OS (the holder can be preempted at any time) | Block, possibly after spinning briefly |
| Kernel code with preemption disabled and interrupts handled carefully | Spinlocks are common |

The danger of spinning in user space is **holder preemption**: if the scheduler preempts the thread that holds a spinlock (its time slice ends), every waiter burns its entire time slice spinning on a lock that cannot be released until the holder is scheduled again. Kernels avoid this by disabling preemption while a spinlock is held; user-space programs cannot.

**Adaptive locks** combine the two: spin for a short while (often only if the holder is currently running on another CPU), then block. Many production mutexes (in glibc as an option, in the HotSpot JVM's built-in monitors, in Go's `sync.Mutex`) spin briefly before sleeping.

### Mechanism: the futex, a lock that usually never enters the kernel

A naive blocking mutex would make a system call on every lock and unlock, costing hundreds of nanoseconds (Chapter 3, Section 3.2) even when no other thread is interested. Linux solves this with the **futex** ("fast user-space mutex"): a 32-bit integer in ordinary user memory, plus two system calls that operate on its address.

- `futex(addr, FUTEX_WAIT, expected)`: "put me to sleep on the wait queue for `addr`, **but only if** `*addr` still equals `expected`". The kernel performs the check and the enqueue atomically with respect to `FUTEX_WAKE`, which is what prevents a lost wake-up (a release that happens between "I saw it locked" and "I went to sleep").
- `futex(addr, FUTEX_WAKE, n)`: wake up to `n` threads sleeping on `addr`.

The kernel keeps a hash table from addresses to wait queues; a futex that nobody is waiting on costs the kernel nothing. A mutex built on a futex uses the integer's value to remember whether anyone is waiting:

```pseudo
// state: 0 = unlocked, 1 = locked with no waiters, 2 = locked and maybe waiters
shared atomic int state = 0

lock():
    c = compare_and_swap_returning_old(&state, 0, 1)
    if c == 0:
        return                         // FAST PATH: one atomic instruction, no syscall
    // SLOW PATH: mark "there are waiters", then sleep until we get it
    if c != 2:
        c = atomic_exchange(&state, 2)
    while c != 0:
        futex_wait(&state, 2)          // sleeps only if state is still 2
        c = atomic_exchange(&state, 2) // woke up: try to take it (still marking waiters)

unlock():
    if atomic_fetch_sub(&state, 1) != 1:   // old value was 2: there may be waiters
        atomic_store(&state, 0)
        futex_wake(&state, 1)              // wake one of them
    // old value was 1: nobody waiting, no syscall
```

The result: an **uncontended** lock and unlock cost one atomic instruction each and never enter the kernel. Only when threads actually collide does the slow path make system calls. glibc's `pthread_mutex_t`, and the locks inside the JVM, Go runtime and most other runtimes on Linux, are built on futexes. (Other operating systems provide equivalents: for example `WaitOnAddress` on Windows.)

### Reentrancy

A **reentrant** (or **recursive**) lock can be acquired again by the thread that already holds it, without deadlocking; it keeps an owner and a **hold count**, and is released only when the count returns to zero.

```pseudo
reentrant_lock(m):
    if m.owner == current_thread:
        m.count = m.count + 1
        return
    acquire_underlying(m)             // wait as usual
    m.owner = current_thread
    m.count = 1

reentrant_unlock(m):
    m.count = m.count - 1
    if m.count == 0:
        m.owner = none
        release_underlying(m)
```

Why it exists: object-oriented code commonly has one locked method calling another on the same object (`transfer()` calls `withdraw()`, both locked). With a non-reentrant lock the second acquisition waits for itself forever: a one-thread **self-deadlock**.

The argument against reentrancy: when the inner method is called, the outer method may be in the middle of changing the protected state, so the invariant the inner method assumes may not hold. A non-reentrant lock makes that situation fail loudly. POSIX mutexes are non-reentrant by default (`PTHREAD_MUTEX_RECURSIVE` is an option); Go's `sync.Mutex` is not reentrant; Java's `synchronized` and `ReentrantLock` are reentrant.

### Fairness

A **fair** lock grants the lock to waiting threads in the order they started waiting (FIFO). An **unfair** (or **barging**) lock lets a thread that arrives just as the lock is released take it immediately, even if others have been waiting longer.

Unfair locks have higher throughput, because handing the lock to a thread that is already running on a CPU avoids waking a sleeping thread and waiting for the scheduler to run it. During that wake-up gap (microseconds), nobody can use the lock. With strict FIFO hand-off, every acquisition under contention pays that gap, and a **lock convoy** can form: threads queue on a hot lock, each runs briefly, releases, and the lock sits unused while the next waiter is woken. The cost of unfairness is that a particular thread can lose the race repeatedly: **starvation** (Section 5.8). Most locks are unfair by default and offer a fair mode; Go's `sync.Mutex` switches to a FIFO "starvation mode" when a waiter has waited longer than about a millisecond.

### Read–write locks

A **read–write lock** (or **shared–exclusive lock**) allows either any number of concurrent readers or exactly one writer. A reader acquires it in **shared** mode; a writer acquires it in **exclusive** mode.

```text
            requested:   shared (read)    exclusive (write)
held:
  nothing                 grant            grant
  shared by N readers     grant            wait
  exclusive (writer)      wait             wait
```

Why: for read-mostly data with non-trivial read critical sections (looking up routing tables, reading a configuration map), letting readers proceed in parallel removes most of the serialisation a plain mutex would cause.

Why a read–write lock is not automatically better than a mutex:

- **Readers still write.** Acquiring the lock in shared mode must increment a reader count, which is an atomic write to a shared cache line. With many cores and very short read sections, that line bounces between cores (Section 5.4), and a plain mutex or a lock-free snapshot (Section 5.3's reference swap) can be faster.
- **Writer starvation.** If readers are always present, a writer may wait forever (the first readers–writers problem, Section 5.7). Implementations choose a policy: reader preference, writer preference (new readers wait if a writer is waiting) or fair.
- **Upgrade deadlock.** If two threads hold the lock in shared mode and both try to upgrade to exclusive mode, each waits for the other to release its shared hold: a deadlock (Section 5.8). Most read–write locks do not support upgrading for this reason; the thread must release the read lock and acquire the write lock, then re-check the state, because it may have changed in between.

A related technique, the **sequence lock** (seqlock, used in the Linux kernel for things like timekeeping), lets readers proceed without writing anything: a writer increments a sequence number before and after its update; a reader reads the number, reads the data, and reads the number again, and retries if it changed or was odd. Readers never block writers, at the cost of retries.

### Lock granularity

**Granularity** is how much state one lock protects. A **coarse-grained** lock protects a lot (a whole cache, a whole service object); a **fine-grained** lock protects a little (one bucket of a hash map, one account).

| | Coarse-grained | Fine-grained |
|---|---|---|
| Simplicity | One lock, easy to reason about | Many locks, easy to forget one |
| Parallelism | Serialises all operations | Unrelated operations proceed in parallel |
| Deadlock risk | Low (one lock) | Higher (operations that need several locks must order them, Section 5.8) |
| Overhead | One acquisition per operation | Possibly several acquisitions; more memory |

**Lock striping** is a middle ground: use a fixed array of *k* locks and protect item *i* with lock `hash(i) mod k`. Concurrent hash maps have used striping (one lock per segment of buckets) and per-bucket locking.

### The cost of lock contention

**Lock contention** occurs when threads frequently try to acquire a lock that another thread holds. Its cost is not the lock instruction; it is **serialisation**.

*Worked example.* A handler holds a global lock for 50 µs per request (it updates an in-memory structure). However many cores and threads you add, the lock can be held by at most one thread, so at most 1 s / 50 µs = **20,000 requests per second** can pass through it. If the service needs 30,000, adding servers' worth of cores to *this process* does not help. Worse, as utilisation of the lock approaches 100%, the queue of waiters grows sharply (queueing theory, Chapter 11), and each waiter that blocks adds context-switch overhead. If the critical section becomes 5 µs (move the expensive work outside the lock), the ceiling becomes 200,000 per second.

Practical rules:

1. **Keep critical sections short.** Compute outside the lock; inside, only read or swap the shared state.
2. **Never hold a lock across I/O or a call to another service.** A 50 µs critical section that sometimes includes a 200 ms database call becomes a 200 ms critical section for everybody.
3. **Don't call unknown code while holding a lock** (callbacks, listeners, overridable methods); it may block, or take other locks and cause deadlock.
4. **Reduce sharing** rather than optimise the lock: per-thread data, striping, immutable snapshots (Section 5.10).

### Locks across processes

The locks above protect memory shared by threads of one process. Processes on one machine can coordinate too (Chapter 4, Section 4.8, introduced shared memory and promised this):

- **Process-shared mutexes.** A futex lives at an address in memory; if that memory is a shared mapping, threads of different processes can use the same futex. POSIX exposes this as a mutex with the `PTHREAD_PROCESS_SHARED` attribute placed in shared memory. If a process dies while holding it, other processes would wait forever; a **robust** mutex reports `EOWNERDEAD` to the next locker, which must repair the shared state and mark it consistent.
- **File locks.** `flock()` locks a whole file; `fcntl()` record locks (POSIX locks) lock byte ranges. Both are **advisory**: they constrain only processes that also ask for the lock, not processes that simply read or write the file. The kernel releases them when the process exits, which makes them safer than a lock in shared memory. A notorious detail: POSIX `fcntl` locks are owned by the *process* and are released when the process closes *any* file descriptor for that file, even one opened elsewhere in the program by a library.
- **Lock files.** Creating a file with `O_CREAT | O_EXCL` succeeds for exactly one process, which can serve as a crude lock; a crash leaves a stale lock file behind, so tools that use this (for example PID files) must detect and clean up stale ones.

Locks across *machines* (several service instances) are a different problem with different failure modes: the network can partition, a lock holder can pause, and a lock service can lose its state. That is Chapter 23 (distributed locks and fencing tokens).

> **Java lens:** Every Java object has a built-in reentrant, unfair lock used by `synchronized`. It is released automatically when the block exits, including by an exception. `java.util.concurrent.locks.ReentrantLock` adds `tryLock()` (fail instead of waiting), `tryLock(timeout)`, `lockInterruptibly()`, an optional fair mode (`new ReentrantLock(true)`) and multiple condition variables (Section 5.6); it must be released in a `finally` block (`lock.lock(); try { ... } finally { lock.unlock(); }`). `ReentrantReadWriteLock` is a read–write lock (no upgrade from read to write). `StampedLock` is a non-reentrant lock with an *optimistic read* mode that works like a seqlock: read without locking, then call `validate(stamp)` to check that no writer intervened.

> **Other runtimes:** Go's `sync.Mutex` is not reentrant: locking it twice in one goroutine deadlocks. `sync.RWMutex` is a read–write lock. The idiom `mu.Lock(); defer mu.Unlock()` releases on every return path. Python's `threading.Lock` is non-reentrant and `threading.RLock` is reentrant; `with lock:` releases on exit.

### Guarantees and non-guarantees

A mutex guarantees that at most one thread at a time holds it, and that everything written before an unlock is visible to the next thread that acquires it. It does **not** guarantee: that code which does not take the lock respects it (locks are a convention); fairness (unless you ask for a fair lock); freedom from deadlock (Section 5.8); any bound on how long you wait (unless you use a timed acquire); or protection across processes or machines (unless specifically designed for it).

### Failure and diagnosis

| Misuse | Symptom | Evidence |
|---|---|---|
| Lock not released on an error path | First failure leaves the lock held; all later callers hang forever | Thread dump: many threads waiting on one lock; the "holder" is not in the critical section at all (or no live thread owns it) |
| Lock held during I/O | Latency of unrelated requests tracks the slowest downstream call | Thread dump: one thread holding the lock is in a socket read; others waiting on it |
| Different code paths using different locks for the same data | Races persist "despite locking" | Code review of every access path; race detector |
| Locking on an object that changes (e.g. a field that is reassigned) | Two threads hold "the lock" at once | Threads in the critical section simultaneously; lock identity differs |
| Hot global lock | Throughput plateaus as cores are added; CPU utilisation well below 100% | Lock contention profile (Section 5.14); many threads BLOCKED/waiting on one lock |
| Spinlock in user space under oversubscription | High CPU with little progress | CPU profile hot in a spin loop; more runnable threads than cores |

### Backend connection

In a request-handling service, locks protect in-process shared structures: local caches, rate-limiter buckets, connection-pool bookkeeping, metric registries. The two rules that prevent most production incidents are "hold locks for microseconds, not milliseconds" and "never hold a lock across a network call". Most application code should not need explicit locks at all: it should use concurrent data structures and immutable snapshots (Section 5.10) and keep shared state in the database, whose own concurrency control is the subject of Chapter 17.

---

## 5.6 Semaphores, Condition Variables, and Monitors

Locks answer "how do I stop two threads being here at once?". Two other questions come up constantly: "how do I let at most *N* threads use something?", and "how do I wait until some condition on shared state becomes true, without spinning?". Semaphores and condition variables answer them.

### Semaphores

#### Definition

A **semaphore** (introduced by Dijkstra) is a synchronisation object holding a non-negative integer, with two atomic operations:

- **wait** (also called **P**, **down** or **acquire**): if the value is greater than zero, decrement it and continue; otherwise block until it is greater than zero, then decrement it.
- **signal** (also called **V**, **up** or **release**): increment the value and, if any threads are blocked in `wait`, wake one of them.

A **counting semaphore** can take any non-negative value; it represents a number of available permits. A **binary semaphore** is restricted to 0 and 1.

#### Mechanism

```pseudo
semaphore:
    value: int
    waiters: queue of threads
    m: internal mutex

wait(s):
    lock(s.m)
    while s.value == 0:
        block current thread on s.waiters, releasing s.m while asleep   // atomically, see the lost wake-up below
    s.value = s.value - 1
    unlock(s.m)

signal(s):
    lock(s.m)
    s.value = s.value + 1
    wake one thread from s.waiters (if any)
    unlock(s.m)
```

A semaphore has **no owner**: any thread may call `signal`, including one that never called `wait`. That is what makes it a signalling tool as well as a counting tool.

#### Three uses

1. **Limiting concurrency to N.** Initialise to *N*; `wait` before using the resource, `signal` after. At most *N* threads are inside at any time.

   ```pseudo
   shared semaphore db_slots = 10        // at most 10 concurrent calls to the reporting database

   run_report():
       wait(db_slots)
       try:
           query_reporting_db()
       finally:
           signal(db_slots)              // always give the permit back
   ```

   This is the core of every connection pool (Chapter 18) and of the **bulkhead** resilience pattern (Chapter 24).

2. **Signalling an event.** Initialise to 0. Thread A calls `wait` and blocks; thread B calls `signal` when the event has happened. Because the count is remembered, it works even if B signals *before* A waits (A then passes straight through).

3. **Mutual exclusion.** A binary semaphore initialised to 1 behaves like a lock: `wait` to enter, `signal` to leave.

#### Mutex vs semaphore

| | Mutex | Semaphore |
|---|---|---|
| State | Locked / unlocked, plus an owner | A count |
| Owner | Yes: only the locking thread may unlock | No: any thread may signal |
| Purpose | Mutual exclusion around a critical section | Counting permits; signalling between threads |
| Allowed concurrently | 1 | N (the count) |
| Reentrancy | Possible (recursive mutex), because the owner is known | Not meaningful: a second `wait` by the same thread just consumes another permit |
| Priority inheritance (Section 5.8) | Possible, because the runtime knows who to boost | Not possible: no owner to boost |
| Error detection | Unlock by a non-owner can be detected and reported | A stray `signal` silently inflates the count |

A binary semaphore is therefore *not* the same as a mutex, even though it can be used like one: it lacks ownership, so it cannot support reentrancy, priority inheritance, or "you unlocked a lock you don't hold" errors. Use a mutex for mutual exclusion and a semaphore for counting and signalling.

#### Misuse

- **Permit leak:** a code path (usually an exception) skips the `signal`. Each leak permanently reduces capacity by one, until every caller blocks forever. The symptom is a slow decline in throughput over hours or days, then a total stall; the evidence is a gauge of available permits trending to zero.
- **Over-release:** calling `signal` more times than `wait` (for example, releasing in both a `catch` block and a `finally` block) silently raises the limit above *N*.
- **Wrong order with other locks:** acquiring a semaphore permit while holding a mutex that the permit-holders need is a deadlock (see the bounded buffer in Section 5.7).

### Condition variables

#### Definition

A **condition variable** is a synchronisation object that lets threads wait, without spinning, until some condition on shared state becomes true. It is always used with a mutex that protects that state, and has three operations:

- **wait(cv, m)**: *atomically* release mutex `m` and put the calling thread to sleep on `cv`; when woken, re-acquire `m` before returning.
- **signal(cv)** (also **notify**): wake one thread waiting on `cv`, if any.
- **broadcast(cv)** (also **notifyAll**): wake all threads waiting on `cv`.

The condition variable does not store the condition and has no memory: a `signal` with no waiters does nothing. The condition itself (the "predicate", e.g. "the queue is not empty") lives in ordinary shared variables protected by the mutex.

#### Why it exists, and why `wait` must release the mutex atomically

Suppose a consumer must wait until a queue is non-empty. Spinning (`while queue.empty(): pass`) burns CPU. Releasing the lock and sleeping in two separate steps loses wake-ups:

```text
Consumer                                    Producer
--------                                    --------
lock(m)
queue.empty()? yes
unlock(m)                                   
                                            lock(m)
                                            queue.push(item)
                                            wake any sleepers  -> nobody is asleep yet!
                                            unlock(m)
sleep()     -> sleeps forever (or until the next push), although an item is waiting
```

This is the **lost wake-up** problem. `wait(cv, m)` solves it by making "release the mutex and go to sleep" a single atomic step with respect to `signal`: a producer cannot get the mutex, push, and signal in the gap, because there is no gap. (The futex's "sleep only if the value is still X" check from Section 5.5 is how this is typically implemented.)

#### Mechanism: the canonical pattern and the "while, not if" rule

```pseudo
// Waiting side
lock(m)
while not condition():      // WHILE, not IF
    wait(cv, m)             // releases m while asleep; holds m again on return
// here: m is held AND condition() is true
do_work_that_needs_condition()
unlock(m)

// Signalling side
lock(m)
change_state_so_condition_may_be_true()
signal(cv)                  // or broadcast(cv)
unlock(m)
```

The waiter must re-check the condition in a loop after every wake-up, for three independent reasons:

1. **Mesa semantics.** In almost all real systems, `signal` only makes the waiter *runnable*; the signaller keeps the mutex and continues. Between the signal and the moment the woken thread re-acquires the mutex, another thread can acquire it and make the condition false again (it takes the item the waiter was woken for, sometimes called a **stolen wake-up**). The alternative, **Hoare semantics**, hands the mutex directly to the woken thread so the condition is guaranteed on return; it is rarely implemented because the extra context switches are costly. The names come from the Mesa programming language and from C. A. R. Hoare's original monitor paper.
2. **Spurious wake-ups.** POSIX and Java explicitly allow `wait` to return without any `signal`, which simplifies their implementation (for example, on some systems a wait interrupted by a signal handler returns early).
3. **Shared condition variables and broadcasts.** If several different conditions share one condition variable, or if `broadcast` wakes every waiter, a woken thread may be woken for a condition that is not its own, or one that another woken thread has already consumed.

With `if`, the thread proceeds on a false condition: it pops from an empty queue, or writes into a full buffer. With `while`, it goes back to sleep.

#### `signal` or `broadcast`?

Use `signal` when exactly one waiter can make progress and all waiters wait for the same condition (for example, one item was added and every consumer waits for "not empty"). Use `broadcast` when the state change may let several waiters proceed, or when waiters wait for *different* conditions on the same condition variable, so that `signal` might wake one that cannot proceed and leave the one that can asleep (a hang). `broadcast` is always safe with the `while` rule, but it wakes everyone to compete for the mutex, and all but one go back to sleep: a **thundering herd** (the same problem as for `accept`, Chapter 4, Section 4.5). The usual fix is separate condition variables for separate conditions (Section 5.7's bounded buffer uses two).

### Monitors

A **monitor** is a programming-language construct that bundles shared data, the operations on it, a mutex that is automatically held during every operation, and one or more condition variables. Only one thread can be "inside" the monitor at a time; a thread inside can wait on a condition, releasing the monitor while it waits.

Monitors make the lock implicit: you cannot forget to take it, and you cannot forget to release it on an error path. Java's `synchronized` methods plus `wait`/`notify` are a monitor; so are C#'s `lock` plus `Monitor.Wait`/`Pulse`. The weakness of Java's built-in monitor is that each object has only **one** implicit condition variable (its "wait set"). An object with two conditions ("not full" and "not empty") must make both kinds of waiters share it and therefore use `notifyAll`.

> **Java lens:** `obj.wait()`, `obj.notify()` and `obj.notifyAll()` may only be called while holding `obj`'s monitor (inside `synchronized (obj)`); otherwise they throw `IllegalMonitorStateException`. `wait()` can wake spuriously, so it must be called in a `while` loop. `Lock.newCondition()` creates separate `Condition` objects (`await`, `signal`, `signalAll`) on one `ReentrantLock`, which is how `ArrayBlockingQueue` implements "not full" and "not empty". `java.util.concurrent.Semaphore` is a counting semaphore (with `tryAcquire` and an optional fair mode). `CountDownLatch` (wait until N events have happened) and `CyclicBarrier` (wait until N threads have arrived) are higher-level signalling tools built on the same ideas.

> **Other runtimes:** Python provides `threading.Semaphore`, `threading.BoundedSemaphore` (raises an error on over-release) and `threading.Condition`. Go provides `sync.Cond`, but idiomatic Go usually signals with channels: a buffered channel of capacity *N* works as a counting semaphore (send to acquire, receive to release).

### Guarantees and non-guarantees

A semaphore guarantees that the number of successful `wait`s never exceeds the initial value plus the number of `signal`s. It does not guarantee which waiter wakes (unless it is fair), does not know who holds permits, and does not detect leaks. A condition variable guarantees that `wait` releases the mutex and sleeps atomically, and that the mutex is held again when `wait` returns. It does **not** guarantee that the condition is true on return, that a `signal` with no waiters is remembered, or which waiter `signal` wakes.

### Failure and diagnosis

- **Using `if` instead of `while`:** rare, load-dependent exceptions such as "queue empty" or index-out-of-bounds in code that "checked first".
- **Waiting without re-checking the condition, or signalling before setting the state:** a hang until the next unrelated signal. Thread dump shows threads waiting on a condition whose predicate is already true.
- **`signal` instead of `broadcast` with mixed conditions:** an occasional permanent hang with work available. Thread dump: consumers waiting on the shared condition variable while the buffer is non-empty.
- **Semaphore permit leak:** available permits trend down after errors; eventually all callers wait in `acquire`.

### Backend connection

These primitives are rarely written by hand in application code, but you use them constantly: a connection pool is a semaphore plus a list of connections; a blocking queue between request threads and background workers is a mutex plus two condition variables; "wait until the server has finished starting" is a latch. Knowing the mechanism lets you read a thread dump: eight threads all waiting to acquire the HTTP client's connection semaphore means the client's connection limit is the bottleneck, not the CPU.

---

## 5.7 Classic Synchronisation Problems

Three problems from the operating-systems literature distil the patterns that recur in real systems. Each is worth solving once, carefully, because interviewers use them and because their failure modes (deadlock, starvation) show up in production code.

### 5.7.1 Producer–consumer (the bounded buffer)

**Problem.** One or more **producers** generate items and put them into a buffer with a fixed capacity *N*; one or more **consumers** take items out. A producer must wait while the buffer is full, a consumer must wait while it is empty, and the buffer's internal state must never be corrupted by concurrent access.

**Why it matters.** This is the structure of a thread pool's work queue (Section 5.11), a logging pipeline, a message broker client's prefetch buffer, and any stage of a pipeline. The fixed capacity is what provides **backpressure**: when consumers fall behind, producers are slowed down instead of memory growing without bound (Chapter 11).

**Solution 1: one mutex and two condition variables.**

```pseudo
shared buffer[N]
shared int count = 0, in = 0, out = 0
shared mutex m
shared condition not_full, not_empty

put(item):                              // producer
    lock(m)
    while count == N:                   // full: wait for space
        wait(not_full, m)
    buffer[in] = item
    in = (in + 1) mod N
    count = count + 1
    signal(not_empty)                   // a consumer may now proceed
    unlock(m)

take():                                 // consumer
    lock(m)
    while count == 0:                   // empty: wait for an item
        wait(not_empty, m)
    item = buffer[out]
    out = (out + 1) mod N
    count = count - 1
    signal(not_full)                    // a producer may now proceed
    unlock(m)
    return item
```

Points to check in an interview: the `while` loops (Section 5.6); two separate condition variables, so that `signal` always wakes a thread that waits for the condition just made true (with one shared condition variable you would need `broadcast`); and the mutex held around all buffer index updates.

**Solution 2: three semaphores.**

```pseudo
shared buffer[N]
shared int in = 0, out = 0
shared semaphore empty_slots = N     // counts free slots
shared semaphore full_slots  = 0     // counts filled slots
shared semaphore mutex       = 1     // binary semaphore protecting in/out/buffer

put(item):
    wait(empty_slots)                // 1. reserve a free slot (blocks if full)
    wait(mutex)                      // 2. then lock the buffer
    buffer[in] = item
    in = (in + 1) mod N
    signal(mutex)
    signal(full_slots)               // one more item available

take():
    wait(full_slots)                 // 1. reserve an item (blocks if empty)
    wait(mutex)                      // 2. then lock the buffer
    item = buffer[out]
    out = (out + 1) mod N
    signal(mutex)
    signal(empty_slots)              // one more free slot
    return item
```

The counting semaphores *are* the conditions, so there is no explicit `while` loop: the count remembers how many slots or items exist.

**The deadlock if you swap the order.** Suppose `put` locks the buffer first and then waits for a free slot:

```pseudo
put_WRONG(item):
    wait(mutex)          // lock the buffer first...
    wait(empty_slots)    // ...then wait for space while HOLDING the lock
    ...
```

```text
Buffer is full (empty_slots = 0, full_slots = N).

Producer                                      Consumer
--------                                      --------
wait(mutex)        -> acquired
wait(empty_slots)  -> 0: BLOCKS, holding mutex
                                              wait(full_slots) -> N: ok
                                              wait(mutex)      -> held by producer: BLOCKS
Both wait forever: the consumer cannot free a slot without the mutex,
and the producer will not release the mutex until a slot is freed.
```

This is a textbook deadlock (Section 5.8) caused by **waiting for a condition while holding a lock that the condition's signaller needs**. Condition variables avoid it by design (`wait` releases the mutex); with semaphores, you must always acquire the counting semaphore *before* the mutex.

### 5.7.2 Readers–writers

**Problem.** A shared data structure is read by many threads and occasionally updated. Readers can safely run concurrently; a writer needs exclusive access. Design the synchronisation.

This is the problem that read–write locks (Section 5.5) solve; the interesting part is the **policy**, because each simple solution starves someone.

**Solution 1: reader preference (the "first readers–writers problem").**

```pseudo
shared int readers = 0
shared semaphore count_mutex = 1     // protects 'readers'
shared semaphore resource = 1        // held by the writer, or by the group of readers

reader():
    wait(count_mutex)
    readers = readers + 1
    if readers == 1:
        wait(resource)               // first reader in locks out writers
    signal(count_mutex)

    read_data()

    wait(count_mutex)
    readers = readers - 1
    if readers == 0:
        signal(resource)             // last reader out lets writers in
    signal(count_mutex)

writer():
    wait(resource)
    write_data()
    signal(resource)
```

**The starvation.** Readers hold `resource` as a group from the moment the first enters until the last leaves. If new readers keep arriving before the last one leaves, `readers` never returns to 0, and a waiting writer waits forever:

```text
time  ->  t0    t1    t2    t3    t4    t5    t6   ...
R1        [====read====]
R2              [====read====]
R3                    [====read====]
R4                          [====read====]
W               waits.......waits.......waits.......  (readers never drops to 0)
```

On a busy read-mostly structure (a configuration cache refreshed by one writer while thousands of requests read it), this means updates may never be applied.

**Solution 2: a fair version with a turnstile.** Add a semaphore that every thread, reader or writer, must pass through in arrival order. A waiting writer holds the turnstile, so readers that arrive after it queue behind it:

```pseudo
shared semaphore turnstile = 1        // arrival order: FIFO if the semaphore is fair

reader():
    wait(turnstile)
    signal(turnstile)                 // pass straight through unless a writer is holding it
    ... same as reader() above ...

writer():
    wait(turnstile)                   // block new readers from entering
    wait(resource)                    // wait for current readers to finish
    signal(turnstile)                 // let others queue up again
    write_data()
    signal(resource)
```

Readers already inside finish; new readers wait at the turnstile; the writer gets in when the last current reader leaves. (Solution 2 is fair only if the semaphore wakes waiters in FIFO order. A **writer-preference** solution, the "second readers–writers problem", goes further and lets waiting writers always go before waiting readers, which can starve readers instead.)

**The upgrade deadlock.** If a thread holding read access tries to upgrade to write access without releasing, and another reader does the same, both wait for `readers` to drop to 0 while each keeps it at least 1. Release and re-acquire instead, and re-validate after.

### 5.7.3 Dining philosophers

**Problem (Dijkstra).** Five philosophers sit at a round table. Between each pair of neighbours is one fork (five forks in total). A philosopher alternates between thinking and eating, and needs both the fork on their left and the fork on their right to eat. Design a protocol that avoids deadlock and starvation.

```text
                 P0
           F4          F0
        P4                  P1
          F3              F1
           P3        P2
                 F2
     Philosopher i uses fork i (right) and fork (i+4) mod 5 = (i-1) mod 5 (left).
```

The forks are locks; the philosophers are threads that each need two locks. This is the shape of any operation that needs two resources at once: transferring money between two accounts, moving an item between two inventories, joining two rooms in a chat system.

**Naive solution: pick up left, then right.**

```pseudo
shared mutex fork[5]

philosopher(i):
    loop:
        think()
        lock(fork[(i + 4) mod 5])   // left
        lock(fork[i])               // right
        eat()
        unlock(fork[i])
        unlock(fork[(i + 4) mod 5])
```

**The deadlock.** If all five become hungry at the same time and each picks up their left fork, every fork is held and every philosopher waits for their right fork, which their neighbour holds:

```text
P0 holds F4, waits for F0  (held by P1)
P1 holds F0, waits for F1  (held by P2)
P2 holds F1, waits for F2  (held by P3)
P3 holds F2, waits for F3  (held by P4)
P4 holds F3, waits for F4  (held by P0)   <- cycle: P0 -> P1 -> P2 -> P3 -> P4 -> P0
```

All four deadlock conditions (Section 5.8) hold, including **circular wait**.

**Fix 1: resource ordering (the standard fix).** Number the forks and make every philosopher pick up the lower-numbered fork first:

```pseudo
philosopher(i):
    a = min(i, (i + 4) mod 5)
    b = max(i, (i + 4) mod 5)
    loop:
        think()
        lock(fork[a])           // always the lower number first
        lock(fork[b])
        eat()
        unlock(fork[b])
        unlock(fork[a])
```

For philosophers 1 to 4 this is still "left then right", but philosopher 0's forks are F4 (left) and F0 (right), so philosopher 0 now picks up F0 first. P0 and P1 both reach for F0 first; one of them waits *without holding anything*, and the cycle cannot close. Section 5.8 explains why a global order always prevents cycles.

**Fix 2: limit the number of diners.** A counting semaphore initialised to 4 allows at most four philosophers to try to eat at once. With five forks and at most four contenders, at least one contender can always get both forks (pigeonhole principle), so there is no deadlock.

```pseudo
shared semaphore seats = 4
philosopher(i):
    loop:
        think()
        wait(seats)
        lock(left); lock(right); eat(); unlock(right); unlock(left)
        signal(seats)
```

**Fix 3: an arbiter (waiter).** A single mutex (the "waiter") must be held while picking up both forks, so a philosopher either takes both or none. This is simple and correct but serialises fork pickup.

**Fix 4: try and back off.** Pick up the left fork, *try* to pick up the right (`tryLock`); if it fails, put the left one down and try again later. This breaks hold-and-wait, but if all philosophers do it in lockstep they can pick up and put down forever: a **livelock** (Section 5.8). Randomised back-off makes this unlikely.

Starvation is a separate question: even without deadlock, a philosopher whose neighbours eat alternately could be unlucky forever. Fair locks or explicit queueing address it.

### Guarantees and non-guarantees

The bounded buffer, built either way, guarantees that no item is lost or duplicated, that producers wait while it is full and consumers while it is empty, and that its indices are never corrupted. It does **not** guarantee the order in which waiting producers or consumers are served (that depends on the fairness of the underlying lock or semaphore), or that consumers keep up: a full buffer slows producers, which is the point. The reader-preference solution guarantees maximum reader concurrency and does not guarantee that a writer ever runs; the turnstile version bounds writer waiting only if its semaphore is FIFO. Resource ordering guarantees the philosophers cannot deadlock; it does not guarantee that each philosopher eats, which needs fairness as well.

### Backend connection

These three problems are the shapes of everyday backend code. Producer–consumer is every work queue between request threads and background workers, and its capacity is the first form of backpressure (Chapter 11). Readers–writers is every read-mostly in-memory structure (configuration, routing tables, caches), where a careless policy lets constant reads starve the refresh. Dining philosophers is every operation that needs two resources at once (two accounts, two inventory rows, two connections from the same pool), and its fix, a global acquisition order, is the same one used to prevent database deadlocks (Chapter 17).

---

## 5.8 Liveness Failures: Deadlock, Livelock, Starvation, Priority Inversion

### Safety and liveness

A concurrent program can fail in two fundamentally different ways. A **safety** property says "nothing bad ever happens" (mutual exclusion is never violated; stock never goes negative). Race conditions violate safety. A **liveness** property says "something good eventually happens" (every request is eventually served; a waiting thread eventually acquires the lock). Locks fix safety problems and, used carelessly, create liveness problems. This section covers the four named liveness failures.

### Deadlock

#### Definition

A **deadlock** is a state in which each thread (or process, or transaction) in a set is waiting for a resource held by another member of the same set, so that none of them can ever proceed without outside intervention.

#### Why it happens

Deadlock is the price of holding one resource while waiting for another, which is unavoidable whenever an operation needs several resources (two locks, two database rows, two connections). The classic backend shape:

```text
Request A (transfer from account 1 to account 2):   lock(acct1) ... lock(acct2)
Request B (transfer from account 2 to account 1):   lock(acct2) ... lock(acct1)

time   Request A                       Request B
----   ---------                       ---------
t1     lock(acct1)   -> acquired
t2                                     lock(acct2)   -> acquired
t3     lock(acct2)   -> waits for B
t4                                     lock(acct1)   -> waits for A
       ... forever (or until a timeout) ...
```

Both requests hang. Their threads are blocked, using no CPU. If enough requests deadlock, the thread pool runs out of threads and the whole service stops responding, while CPU usage drops.

#### The four Coffman conditions

Coffman, Elphick and Shoshani (1971) showed that a deadlock among resources can occur only if **all four** of these conditions hold at the same time:

1. **Mutual exclusion:** at least one resource is held in a non-shareable mode (only one thread can hold it).
2. **Hold and wait:** a thread holds at least one resource while waiting to acquire another.
3. **No preemption:** a resource cannot be forcibly taken from the thread holding it; it is released only voluntarily.
4. **Circular wait:** there is a cycle of threads T1, T2, ..., Tn in which each Ti waits for a resource held by Ti+1, and Tn waits for a resource held by T1.

The first three are properties of the *system* (how its resources behave); the fourth is a property of a particular *state*. Because all four are necessary, breaking any one prevents deadlock. That observation organises the four strategies: **prevention**, **avoidance**, **detection and recovery**, and ignoring the problem.

#### Mechanism: the resource-allocation and wait-for graphs

The state can be drawn as a **resource-allocation graph**: a node for each thread and each resource; an edge resource → thread means "held by", thread → resource means "waiting for". If each resource has a single instance (true of locks), the graph can be simplified to a **wait-for graph**: a node per thread, and an edge T1 → T2 meaning "T1 waits for a resource that T2 holds".

**For single-instance resources, the system is deadlocked if and only if the wait-for graph contains a cycle.**

```text
Wait-for graph (four request threads):

     T1 -----> T2 -----> T3
     ^                    |
     |                    v
     +------------------- T4        T5 -----> T2

T1 waits for T2 (T2 holds lock L_orders), T2 waits for T3 (L_stock),
T3 waits for T4 (L_payments), T4 waits for T1 (L_users).
Cycle T1 -> T2 -> T3 -> T4 -> T1: these four are deadlocked.
T5 waits for T2 but is not on the cycle: it is not part of the deadlock,
but it is stuck behind it and will wait as long as the deadlock lasts.
```

For resources with several instances (a pool of 10 connections), a cycle in the resource-allocation graph is necessary but not sufficient: another instance might be released by a thread outside the cycle. Detection then needs a Banker's-style algorithm (below).

#### Prevention: make one condition impossible

| Condition broken | Technique | Cost |
|---|---|---|
| Mutual exclusion | Make the resource shareable: read-only data, immutable snapshots, lock-free structures | Not possible for genuinely exclusive resources |
| Hold and wait | Acquire all needed resources at once (or none), or release everything before requesting more | Must know the needs in advance; holds resources longer than needed; reduces concurrency |
| No preemption | If you cannot get the next resource, release what you hold and retry (`tryLock` with timeout); or let the system abort a holder | Retries waste work; possible livelock; aborted work must be safe to redo |
| Circular wait | **Global lock ordering**: number all resources; always acquire in increasing order | Requires discipline across the codebase; hard when the set of locks is discovered dynamically |

**Lock ordering is the standard prevention technique.** Why it works: if every thread acquires locks in increasing order, then in any wait-for chain T1 → T2 → ... each thread is waiting for a lock with a higher number than every lock it holds. Following a chain, lock numbers strictly increase, so the chain can never return to its start: no cycle. Applied to the transfer:

```pseudo
transfer(from, to, amount):
    first  = min_by_id(from, to)          // a global order: account id
    second = max_by_id(from, to)
    lock(first.lock)
    lock(second.lock)
    try:
        from.balance -= amount
        to.balance   += amount
    finally:
        unlock(second.lock)
        unlock(first.lock)
```

Both A (1 → 2) and B (2 → 1) now lock account 1 first. Whichever gets it first proceeds; the other waits *holding nothing*. The same technique prevents database deadlocks: update rows in a consistent order (for example, by primary key) in every transaction (Chapter 17).

#### Avoidance: the Banker's algorithm

**Deadlock avoidance** lets threads request resources freely, but grants a request only if the resulting state is **safe**: a state in which there is at least one order (a **safe sequence**) in which every thread can obtain its maximum remaining needs and finish, each releasing its resources for the next. A safe state cannot lead to deadlock if threads behave as declared. An **unsafe** state is not necessarily deadlocked; it is one from which the system can no longer *guarantee* that a deadlock will be avoided.

The **Banker's algorithm** (Dijkstra) implements this. Each thread declares in advance the maximum number of each resource type it may need. The system keeps:

- **Available[r]**: free instances of each resource type;
- **Max[t][r]**: each thread's declared maximum;
- **Allocation[t][r]**: what each thread currently holds;
- **Need[t][r] = Max − Allocation**: what each thread may still ask for.

**Safety check:**

```pseudo
is_safe():
    work = copy(Available)
    finished[t] = false for all t
    repeat:
        find a thread t with not finished[t] and Need[t] <= work   (component-wise)
        if found:
            work = work + Allocation[t]     // t can finish and return everything it holds
            finished[t] = true
    until no such thread is found
    return all finished
```

**Request:** when thread *t* asks for a vector `Request`, reject it as an error if it exceeds `Need[t]`; make *t* wait if it exceeds `Available`; otherwise *pretend* to grant it (Available −= Request, Allocation[t] += Request, Need[t] −= Request) and run the safety check. If the new state is safe, grant it; if not, undo the pretence and make *t* wait.

**Worked example.** Two resource types: R1 = database connections (10 in total), R2 = worker slots on a batch cluster (5 in total). Three jobs:

| Job | Max (R1, R2) | Allocation | Need = Max − Allocation |
|---|---|---|---|
| J1 | (7, 3) | (2, 1) | (5, 2) |
| J2 | (3, 2) | (2, 1) | (1, 1) |
| J3 | (6, 2) | (3, 1) | (3, 1) |

Allocated in total: (7, 3), so **Available = (10, 5) − (7, 3) = (3, 2)**.

*Is the current state safe?*

| Step | Work before | Candidate with Need ≤ Work | Work after (+ its Allocation) |
|---|---|---|---|
| 1 | (3, 2) | J2: (1, 1) ≤ (3, 2) ✓ | (3, 2) + (2, 1) = (5, 3) |
| 2 | (5, 3) | J1: (5, 2) ≤ (5, 3) ✓ | (5, 3) + (2, 1) = (7, 4) |
| 3 | (7, 4) | J3: (3, 1) ≤ (7, 4) ✓ | (7, 4) + (3, 1) = (10, 5) |

All finish: the state is **safe**, with safe sequence ⟨J2, J1, J3⟩ (⟨J2, J3, J1⟩ also works).

*Request A: J3 asks for (2, 1).* (2, 1) ≤ Need(J3) = (3, 1) and ≤ Available = (3, 2). Pretend: Available = (1, 1), Allocation(J3) = (5, 2), Need(J3) = (1, 0). Safety check: Work (1, 1): J2's need (1, 1) fits → Work (3, 2); J3's need (1, 0) fits → Work (8, 4); J1's need (5, 2) fits → Work (10, 5). **Safe: grant.**

*Request B (instead of A): J1 asks for (3, 1).* (3, 1) ≤ Need(J1) = (5, 2) and ≤ Available = (3, 2). Pretend: Available = (0, 1), Allocation(J1) = (5, 2), Need(J1) = (2, 1). Safety check: Work = (0, 1). J1 needs (2, 1), J2 needs (1, 1), J3 needs (3, 1): none fits. **Unsafe: J1 must wait**, even though the resources are physically available. If it were granted and every job then asked for its declared maximum, no job could finish.

**Why production systems rarely use it.** It requires each thread to declare its maximum needs in advance, a fixed number of threads and resources, and a safety check (O(threads² × resource types)) on every request. Request handlers do not know in advance which locks or rows they will touch. The algorithm is valuable for the *idea*: some states are safe, some are unsafe, and admission control can keep a system in safe states. A connection pool that refuses to give a single request a second connection while it holds one is a crude version of the same idea (Chapter 18 discusses pool deadlock).

#### Detection and recovery

**Detection** lets deadlocks happen, periodically (or when a wait has lasted suspiciously long) builds the wait-for graph and searches it for a cycle, for example with a depth-first search that reports a back edge:

```pseudo
has_cycle(graph):
    for each node n: state[n] = UNVISITED
    for each node n:
        if state[n] == UNVISITED and dfs(n): return true
    return false

dfs(n):
    state[n] = IN_PROGRESS
    for each m in graph.waits_for(n):
        if state[m] == IN_PROGRESS: return true       // back edge: cycle found
        if state[m] == UNVISITED and dfs(m): return true
    state[n] = DONE
    return false
```

**Recovery** then breaks the cycle: abort one member (the **victim**), releasing its resources, or preempt a resource from it, which requires that its work can be rolled back. Victim selection usually prefers the cheapest to redo (the youngest, or the one holding the fewest resources).

This is exactly what relational databases do: PostgreSQL and InnoDB detect cycles among transactions waiting for row locks and abort one with a deadlock error, which the application must catch and retry (Chapter 17). The JVM detects deadlocks among its monitors and `ReentrantLock`s when you take a thread dump (Section 5.14), but it does not recover: the threads stay deadlocked until the process is restarted.

#### Ignoring the problem (and timeouts)

Many systems take the "ostrich" approach: assume deadlocks are rare and rely on a **timeout**. A lock acquisition with a timeout (`tryLock(2 s)`), a request deadline, or a database `lock_timeout` turns a permanent deadlock into a failed request. This is not a solution (the requests still fail, and a timeout does not tell you why), but it is what keeps most servers alive when an unforeseen deadlock occurs. Without any timeout, one deadlock can permanently consume threads until the service stops.

#### Guarantees and non-guarantees

Lock ordering guarantees no deadlock *among the locks that follow the order*; one code path that takes them in another order reintroduces the risk. The Banker's algorithm guarantees safety only if threads never exceed their declared maximums. Detection guarantees that cycles are found, not that the victim's work is safely undone (that is the application's problem: retrying must be safe).

#### Failure and diagnosis

Symptom: a set of requests hangs; CPU usage *drops*; throughput falls to zero for the affected operations or for the whole service once the pool is exhausted; no errors are logged until timeouts fire. Evidence: a thread dump showing threads blocked on locks held by each other (the JVM prints "Found one Java-level deadlock" with the cycle, Section 5.14); in Go, if *every* goroutine is blocked the runtime aborts with `fatal error: all goroutines are asleep - deadlock!`, while a partial deadlock is visible only in a goroutine dump; in databases, deadlock errors in the logs (Chapter 17).

### Livelock

A **livelock** is a state in which threads are not blocked, keep changing state in response to each other, and still make no progress. Each thread is busy, but no work is completed.

```text
Two threads, each needing locks X and Y, using "try, and back off if you can't":

time   T1                                  T2
t1     lock(X) ok                          lock(Y) ok
t2     tryLock(Y) fails -> unlock(X)       tryLock(X) fails -> unlock(Y)
t3     wait 10 ms                          wait 10 ms
t4     lock(X) ok                          lock(Y) ok
t5     tryLock(Y) fails -> unlock(X)       tryLock(X) fails -> unlock(Y)
...    forever, in lockstep
```

The everyday picture is two people in a corridor who both step aside in the same direction, repeatedly. The fix is to break the symmetry: **randomised** back-off (wait a random time, as Ethernet's collision handling does), a fixed priority between the threads, or lock ordering, which removes the need to back off. Unlike deadlock, livelock burns CPU: the symptom is high CPU with zero throughput, and a CPU profile shows time in acquire/release/retry code. Retry storms between services are the distributed version (Chapter 24 covers jitter).

### Starvation

**Starvation** is a condition in which a thread that is ready to make progress is perpetually denied a resource it needs, even though the resource keeps becoming available to others. Unlike deadlock, the system as a whole makes progress; one participant does not.

Causes seen in this chapter and the previous one: unfair (barging) locks where one thread keeps losing the race; reader-preference read–write locks starving writers (Section 5.7.2); strict priority scheduling starving low-priority threads (Chapter 3, Section 3.10). Fixes: fair (FIFO) locks or fair modes; writer-preference or turnstile policies; **aging**, which raises the priority of a waiting task the longer it waits (Chapter 3, Section 3.10). The symptom is tail latency: most requests are fine, but some requests (or one type of operation, such as configuration updates) take a very long time or never complete.

### Priority inversion

**Priority inversion** is a situation in which a high-priority task is forced to wait for a lower-priority task, and that lower-priority task is itself prevented from running by medium-priority tasks, so that the high-priority task is effectively running at the lowest priority.

```text
Priorities: H (high) > M (medium) > L (low).  L and H share mutex m.

time  ->   t0      t1        t2        t3 ............................ t9
L          lock(m) [critical section] --- preempted by M ---------------- resumes, unlock(m)
H                            wakes,
                             lock(m) -> BLOCKS (L holds m)
M                                      runs (higher than L) ......... finishes
                                       M keeps the CPU, so L cannot finish and release m,
                                       so H waits for M, a task of LOWER priority than H.
```

The best-known incident is NASA's Mars Pathfinder lander in 1997, which kept resetting itself: a low-priority meteorological task held a mutex needed by a high-priority bus-management task, medium-priority communication tasks preempted the low-priority one, and a watchdog timer detected that the high-priority task had not run in time and reset the system. The fix, uploaded from Earth, was to enable priority inheritance on that mutex.

Fixes:

- **Priority inheritance:** while a low-priority thread holds a mutex that a higher-priority thread is waiting for, it temporarily runs at the waiter's priority, so medium-priority work cannot preempt it; it drops back on unlock. This requires knowing the mutex's *owner*, which is one more reason why ownership (Section 5.6) matters.
- **Priority ceiling:** each mutex has a ceiling equal to the highest priority of any thread that may lock it; a thread that holds it runs at that ceiling.
- **Don't share locks across priority classes**, or make the critical sections non-blocking.

In general-purpose backend servers, thread priorities are rarely used and the Linux scheduler shares CPU time by weight rather than strict priority (Chapter 3, Section 3.10), so classic priority inversion is mostly a real-time-systems problem. The same *shape* is common, though: a latency-critical request waits for a lock, connection or database row held by a slow background job that is itself starved of CPU or I/O by other work. Recognising "my important work is waiting on unimportant work that is waiting on something else" is the transferable skill.

---

## 5.9 Lock-Free and Wait-Free Data Structures

### Definition

Concurrent algorithms are classified by their **progress guarantee**, from weakest to strongest:

- **Blocking:** a thread can be delayed indefinitely by another thread that is delayed (for example, a lock holder that has been descheduled or has crashed).
- **Obstruction-free:** a thread completes its operation in a bounded number of steps if it eventually runs alone (all other threads pause).
- **Lock-free:** at every moment, *some* thread completes an operation in a finite number of steps, no matter how the threads are scheduled or delayed. The system as a whole always makes progress, though an individual thread may starve.
- **Wait-free:** *every* thread completes each operation in a bounded number of its own steps, regardless of other threads. No thread starves.

"Lock-free" therefore does not just mean "does not use a lock object". It means that the suspension of any thread at any point cannot prevent the others from making progress.

### Why they exist

Locks have failure modes that are inherent to blocking: a preempted, page-faulting or crashed lock holder stalls every waiter; locks can deadlock; and priority inversion is possible. Code that must never block (interrupt handlers, signal handlers, real-time paths, the internals of a garbage collector or a runtime scheduler) cannot use them. Lock-free structures remove those failure modes, and in some highly contended workloads they also scale better.

### Mechanism

Lock-free algorithms are built from CAS loops (Section 5.4): read the current state, compute a new state privately, and publish it with one CAS; if the CAS fails, someone else succeeded (so the system made progress), and you retry. That last point is exactly why a CAS loop is lock-free: a CAS can only fail because another thread's CAS succeeded.

**A lock-free stack (Treiber stack) push:**

```pseudo
push(value):
    node = new Node(value)
    loop:
        old_top = atomic_load(top)
        node.next = old_top
        if compare_and_swap(&top, old_top, node):
            return
```

`pop` is the CAS loop shown in Section 5.4, with its ABA hazard. A lock-free FIFO queue (the **Michael–Scott queue**, the basis of many library queues) uses two pointers, `head` and `tail`, and a technique called **helping**: if a thread finds `tail` lagging behind (another thread linked a node but has not yet advanced `tail`), it advances `tail` itself before trying its own operation, so that a delayed thread can never block the others.

**Wait-free** algorithms usually need more machinery: each thread announces its operation in a shared array, and all threads help complete pending announced operations, so that every operation finishes in bounded steps. Some simple operations are wait-free in hardware: fetch-and-add on x86 always completes in one instruction, so an atomic counter incremented with `fetch_and_add` is wait-free, while the same counter incremented with a CAS loop is only lock-free (a thread can lose the CAS race repeatedly).

```text
Progress under a stalled thread (T2 is descheduled at the worst possible moment):

Lock-based counter:   T2 holds lock, stalls  -> T1, T3, T4 all wait: system progress = 0
CAS-loop counter:     T2 stalls before its CAS -> T1, T3, T4 continue; T2's later CAS
                      fails and retries: system progress continues
FAA counter:          each increment is one instruction: every thread progresses
```

### Atomic counters at scale

Counters are the lock-free structure backend engineers meet most often (request counts, bytes sent, error totals). Three implementations, in increasing scalability:

| Implementation | Progress | Behaviour under heavy contention from many cores |
|---|---|---|
| Mutex-protected integer | Blocking | Serialised; waiters may sleep and wake |
| Single atomic integer (FAA or CAS) | Wait-free (FAA) / lock-free (CAS) | One cache line bounces between all cores; throughput can *decrease* as cores are added |
| Striped counter (one cell per thread or per core, summed on read) | Wait-free writes | Writes hit different cache lines (padded to avoid false sharing); reads are slower and not an atomic snapshot |

### Why lock-free is not automatically faster

It is common to assume that removing locks makes code faster. Often it does not:

1. **Contention costs are about cache lines, not locks.** A lock-free structure under contention still has many cores writing the same few cache lines (`top`, `head`, `tail`). Each successful CAS invalidates the line everywhere else. That traffic, not the lock, is the dominant cost of a contended lock too.
2. **Failed CAS attempts are wasted work.** Under heavy contention, threads can spend most of their time computing new states that are then rejected. A blocking lock instead puts waiters to sleep and lets the holder run without interference.
3. **More atomic operations per operation.** A well-implemented uncontended mutex costs one atomic instruction to lock and one store or atomic to unlock (Section 5.5). A lock-free queue operation may need two or more CASes, plus memory-reclamation bookkeeping (hazard pointers or epochs) in languages without garbage collection.
4. **Memory reclamation and allocation.** Lock-free linked structures typically allocate a node per operation; a lock-based array buffer may not allocate at all.
5. **Correctness is much harder.** Lock-free algorithms are notoriously subtle (ABA, memory ordering on weak hardware, reclamation). Published algorithms have had bugs found years later.

The real advantages are **progress guarantees** (no deadlock, no stall when a thread is preempted, no priority inversion) and **predictable tail latency** in specific contended scenarios. The decision should be made by measurement under realistic contention, not by assumption. For application code, the right move is almost always to use a well-tested library structure (a concurrent queue or map) and let its authors worry about the algorithm.

### Guarantees and non-guarantees

Lock-free guarantees system-wide progress; it does not guarantee that any particular thread makes progress (starvation is possible), does not bound latency (except wait-free), does not make a sequence of operations on the structure atomic (a lock-free map's `get` followed by `put` is still check-then-act), and does not by itself guarantee higher throughput.

### Failure and diagnosis

Hand-written lock-free code fails with rare corruption (lost nodes, duplicated items, crashes) that appears under load or only on weakly ordered hardware. Library lock-free structures fail by contention: high CPU in CAS loops and poor scaling, visible in CPU profiles and hardware counters. The remedies are the same as for a hot lock: reduce sharing (sharding, striping, batching), not a cleverer algorithm.

### Backend connection

You will use lock-free structures constantly without writing any: the queues inside thread pools and HTTP clients, concurrent maps used as caches, atomic counters in metrics libraries, the run queues of runtimes such as Go's and the JVM's work-stealing pools (Section 5.11). Knowing their guarantees prevents two mistakes: assuming that a concurrent map makes multi-step logic atomic, and assuming that replacing a lock with atomics will fix a scaling problem that is really caused by every core writing the same data.

---

## 5.10 Thread-Safety Strategies

### Definition

A piece of code (a function, an object, a module) is **thread-safe** if it behaves correctly when used from several threads at once, under any scheduling or interleaving, without the callers having to add synchronisation of their own. "Correctly" means its invariants and its documented behaviour hold exactly as they would if the calls were made one at a time in some order.

### Why it matters

Sections 5.4 to 5.9 gave the low-level tools. In practice, most concurrency bugs come not from misusing a lock but from never deciding how a piece of shared state is supposed to be protected. A thread-safety *strategy* is that decision, made deliberately for each piece of state. There are five, and the first two avoid the problem rather than solve it.

### Mechanism: five strategies, from cheapest to most expensive to reason about

**1. Confinement: don't share.** State that only one thread can reach needs no synchronisation at all.

- **Stack confinement:** local variables and objects reachable only from them live in one thread's stack frame (Chapter 2, Section 2.6) and are private to that thread.
- **Thread confinement:** an object is used by only one thread by design, for example a per-thread buffer, or every object touched by a single event-loop thread (Section 5.13).
- **Thread-local storage:** a variable with a separate copy per thread (Chapter 3, Section 3.7), used for per-thread caches of non-thread-safe objects (formatters, random-number generators) and for context such as the current request ID.
- **Ownership transfer:** an object is handed from one thread to another through a synchronising channel or queue, and the sender stops using it. At every moment exactly one thread owns it, and the hand-off creates the happens-before edge (Section 5.3).

```pseudo
// Request-scoped state: confined to the thread (or task) handling the request
handle(request):
    order = parse(request)            // a new object, reachable only from this call
    order.total = compute_total(order)  // no other thread can see 'order': no lock needed
    save(order)
```

The pitfall of thread-local storage in a **thread pool**: pool threads are reused across tasks, so a value set by one request is still there when the same thread runs the next request, unless it is cleared. That can leak one user's data into another user's request, and it keeps objects alive for as long as the thread lives (a memory leak). Chapter 11 covers request-scoped context in servers.

**2. Immutability: don't mutate.** An object whose state cannot change after construction can be shared freely by any number of threads, provided it is published safely (Section 5.3). To "change" it, you build a new object and publish a reference to the new one:

```pseudo
shared atomic ref current_config      // points to an immutable Config

reload():
    new_config = Config(parse(read_file()))   // build privately
    atomic_store_release(current_config, new_config)   // publish in one step

handle(request):
    cfg = atomic_load_acquire(current_config) // take one consistent snapshot
    use(cfg.timeout_ms, cfg.max_retries)      // both values from the same version
```

This **copy-on-write** pattern gives readers a consistent snapshot with no lock, and is ideal for read-mostly data. Its costs are a new allocation per update, and lost updates if two *writers* each copy the old version and publish (two writers need a lock, or a CAS loop on the reference).

**3. Synchronisation: protect the state.** When state must be shared and mutated, guard it with a lock or atomics (Sections 5.4 and 5.5). The discipline that makes this work: for each piece of shared mutable state, name the lock that guards it, take that lock on every access (reads included), and keep every invariant that spans several variables under one lock.

**4. Delegation: use a concurrent data structure.** Let a well-tested library structure do the synchronisation: a concurrent map, a blocking queue, an atomic variable, a copy-on-write list. The structure guarantees that each of its operations is atomic. It does not make *your sequence* of operations atomic, which is why good concurrent maps offer atomic compound operations:

```pseudo
// WRONG: check-then-act across two atomic calls (Section 5.2)
if not cache.contains(key):
    cache.put(key, load(key))     // two threads can both load and both put

// RIGHT: one atomic operation does the check and the act
value = cache.compute_if_absent(key, k -> load(k))
// most concurrent maps also offer put_if_absent and merge
```

A caution: atomic compound operations usually hold an internal lock (on the bucket) while running your function, so the function must be short and must not modify the same map; a slow `load` (a database call) inside `compute_if_absent` blocks every other access to that bucket.

**5. Message passing: share by communicating.** Instead of several threads touching shared state, give the state to one owner (a thread, goroutine or actor) and have others send it messages: requests to read or change the state. The owner processes messages one at a time, so its state is confined (strategy 1) and needs no lock. This is the model of Go channels and of actors (Section 5.13).

```pseudo
// One owner goroutine/actor holds the counter; others send messages
owner():
    count = 0
    loop:
        msg = receive(inbox)
        if msg.type == INCREMENT: count = count + 1
        if msg.type == GET:       send(msg.reply_to, count)
```

The costs: every interaction is a message (a queue operation and often a context switch), the owner is a serial bottleneck, and a request–reply message between two owners that each wait for the other is a deadlock like any other.

### Concrete: choosing a strategy

| State | Good strategy | Why |
|---|---|---|
| Data for one request | Confinement | Not shared at all |
| Configuration, feature flags, routing tables | Immutability + atomic reference swap | Read on every request, changed rarely |
| In-memory cache | Delegation (concurrent map with atomic compound operations) | Many readers and writers, independent keys |
| Rate-limiter bucket per client | Synchronisation per bucket (fine-grained lock) or atomics | Compound update (refill + take) per key |
| Metrics counters | Atomics or striped counters | Single-word updates, very hot |
| Background job state machine | Message passing to one owner | Complex state, sequential logic, serial by nature |

### Guarantees and non-guarantees

"Thread-safe" applies to the documented operations of a component, one at a time. It does not make compositions of those operations atomic, does not extend to objects you get out of a thread-safe container (a thread-safe map can hold non-thread-safe values), and does not cover state shared across processes or machines.

### Failure and diagnosis

The common failures are *escape*: an object believed to be confined is in fact reachable by another thread (stored in a static field, captured by a callback that runs on another thread, returned from a "read-only" getter that exposes an internal mutable list); and *compound misuse* of concurrent collections. Evidence comes from code review ("who else can reach this object?") and from race detectors.

### Backend connection

A well-designed request handler is mostly strategy 1 (everything it touches is request-scoped), reads shared configuration through strategy 2, uses caches and pools through strategy 4, and keeps the shared, durable state in the database, whose concurrency control (Chapter 17) is the backend's real strategy 3. Explicit locks in handler code are a sign that state that belongs in the database, or in a library structure, is being managed by hand.

---

## 5.11 Thread Pools and Executors

### Definition

A **thread pool** is a fixed or bounded set of pre-created **worker threads** that repeatedly take tasks from a shared **work queue** and run them. An **executor** is the abstraction that separates *submitting* a task ("run this") from *how and when* it runs (on which thread, in what order, with what limits). A thread pool is the most common implementation of an executor.

### Why it exists

Creating a kernel thread costs a system call, a stack allocation (commonly reserved at several hundred kilobytes to a few megabytes of virtual memory) and scheduler bookkeeping (Chapter 3, Section 3.7), typically tens of microseconds. Creating one per task wastes that time, and more importantly it puts **no bound on concurrency**: a traffic spike creates thousands of threads, each with memory and scheduling cost, each competing for the same CPUs and the same database. A pool fixes both problems: threads are reused, and the number running at once is limited. The queue absorbs bursts, and the executor gives one place to decide what happens when the system is overloaded.

### Mechanism

```pseudo
pool:
    queue: bounded blocking queue of tasks      // producer-consumer, Section 5.7.1
    workers: list of threads
    shutting_down = false (atomic)

start(n):
    repeat n times:
        workers.add(start_thread(worker_loop))

worker_loop():
    loop:
        task = queue.take()             // blocks while the queue is empty
        if task == POISON_PILL: return  // shutdown signal
        try:
            task.run()
        catch error:
            record(error)               // never let one task's error kill the worker silently

submit(task):
    if shutting_down: reject(task)
    if not queue.offer(task):           // queue full
        rejection_policy(task)          // see below

shutdown():
    shutting_down = true
    repeat len(workers) times: queue.put(POISON_PILL)
    join all workers                    // waits for queued tasks to finish
```

```text
  submitters                      work queue (bounded)                 workers
  ----------                      --------------------                 -------
  request thread --submit-->  [ t7 | t6 | t5 | t4 |    |    ]  --take-->  W1 running t1
  request thread --submit-->                                   --take-->  W2 running t2
  timer          --submit-->      full? -> rejection policy    --take-->  W3 running t3
```

A pool has three parameters that matter: the **number of threads**, the **queue type and capacity**, and the **rejection policy**.

### Sizing for CPU-bound and I/O-bound work

Recall the distinction from Chapter 3, Section 3.12. A task is **CPU-bound** if it spends its time computing, **I/O-bound** if it spends most of its time waiting (for the network, a disk, a database).

**CPU-bound tasks:** use about as many threads as there are CPU cores available to the process (N_cpu, or N_cpu + 1 so a core is not idle while a thread takes a page fault). More threads cannot do more computation; they only add context switches and cache pollution. In a container, "cores available" means the cgroup CPU limit, not the host's core count (Chapter 4, Section 4.10): a pool of 64 threads on a 64-core host whose container is limited to 2 CPUs will be throttled heavily.

**I/O-bound tasks:** threads spend most of their time blocked, so more threads than cores are needed to keep the cores busy. A common estimate (popularised by Brian Goetz's *Java Concurrency in Practice*) is:

```text
N_threads = N_cpu × U_target × (1 + W / C)

  N_cpu     cores available
  U_target  desired CPU utilisation (0 to 1)
  W / C     ratio of wait time to compute time per task
```

*Worked example.* 4 cores, target utilisation 0.8, each task computes for 5 ms and waits 45 ms for a database (W/C = 9): N = 4 × 0.8 × 10 = **32 threads**.

Two corrections stop this formula from misleading you:

1. **The downstream resource is usually the real limit.** If those 32 threads each need a database connection while waiting, and the pool of database connections has 10, then 22 threads are always waiting for a connection, not for the database. The useful concurrency is min(threads, connections, what the database can serve). The formula sizes for *your* CPU; it says nothing about the capacity of what you call. Chapter 11 sizes pools from throughput and latency with Little's Law; Chapter 18 sizes database connection pools.
2. **W/C changes under load.** When the database slows down, W grows, and the formula asks for more threads, which send more concurrent queries to the database that is already slow. A bounded pool is a deliberate cap, not something to grow automatically.

### Queues and rejection policies

| Queue type | Behaviour | Risk |
|---|---|---|
| **Unbounded** | Never rejects; queue grows during overload | Memory grows without bound; latency grows without bound (tasks wait minutes and are useless when they finally run); overload is hidden until the process runs out of memory |
| **Bounded** | Rejects (via the policy) when full | Must choose a capacity and a policy |
| **Synchronous hand-off** (capacity 0) | A submit succeeds only if a worker is free to take it right now | Rejects whenever all workers are busy; usually paired with a pool that can grow |

When the pool is saturated (all workers busy, queue full), the **rejection policy** decides what happens to a new task:

| Policy | What happens | When it is appropriate |
|---|---|---|
| **Abort** | `submit` fails with an error | When the caller can return a fast failure (an HTTP 503) rather than wait: fail fast under overload |
| **Caller runs** | The submitting thread runs the task itself | Natural **backpressure**: the submitter is slowed down, so it submits less. Dangerous if the submitter is an event-loop or request-accepting thread, which then stops accepting |
| **Discard** | The new task is silently dropped | Only for tasks that are genuinely optional (best-effort metrics sampling) |
| **Discard oldest** | The oldest queued task is dropped, the new one queued | When fresher work is worth more than older work (latest sensor reading) |

Silent discarding is a common source of "the job sometimes doesn't run" bugs; at minimum, count rejections in a metric.

### Thread-starvation deadlock

A pool can deadlock with no locks at all. If a task running on the pool submits a subtask *to the same pool* and then blocks waiting for its result, and all workers are running such parent tasks, every worker waits for a subtask that sits in the queue with no free worker to run it:

```text
Pool with 2 workers.
W1 runs Parent A: submits Child A to the pool, waits for Child A's result
W2 runs Parent B: submits Child B to the pool, waits for Child B's result
Queue: [Child A, Child B]   <- no worker is free to run them; parents wait forever
```

This is a resource deadlock (Section 5.8) in which the resource is "a worker thread". Fixes: never block a pool thread waiting for work queued on the same pool; use separate pools for parents and children; or use composition (Section 5.12) instead of blocking waits.

### Work stealing

A single shared queue is itself a contention point (Section 5.9). A **work-stealing** pool gives each worker its own double-ended queue: a worker pushes and pops its own tasks at one end (cheap, rarely contended), and an idle worker *steals* tasks from the other end of a busy worker's queue. This scales well for many small tasks, especially recursive divide-and-conquer work. Java's `ForkJoinPool` and the Go runtime scheduler (Section 5.13) both use work stealing.

> **Java lens:** `ExecutorService` is the executor interface; `ThreadPoolExecutor` is the configurable pool (core size, maximum size, keep-alive time, queue, `ThreadFactory`, `RejectedExecutionHandler`). Its rejection handlers are `AbortPolicy` (the default; throws `RejectedExecutionException`), `CallerRunsPolicy`, `DiscardPolicy` and `DiscardOldestPolicy`. A frequently misunderstood rule: `ThreadPoolExecutor` creates threads beyond the core size only when the **queue is full**, so with an unbounded queue the pool never grows beyond its core size. The factory methods hide risky choices: `Executors.newFixedThreadPool(n)` uses an unbounded `LinkedBlockingQueue`; `Executors.newCachedThreadPool()` uses a synchronous hand-off and an unbounded number of threads. A task submitted with `submit()` that throws stores its exception in the returned `Future`; if nobody calls `get()`, the error is never seen (with `execute()` it goes to the thread's uncaught-exception handler). `ForkJoinPool` is the work-stealing pool; `ForkJoinPool.commonPool()` is shared by parallel streams and, by default, by `CompletableFuture`'s `...Async` methods. Since Java 21, `Executors.newVirtualThreadPerTaskExecutor()` creates a new virtual thread per task instead of pooling (Section 5.13).

> **Other runtimes:** Python's `concurrent.futures.ThreadPoolExecutor` and `ProcessPoolExecutor` (processes, to get around the GIL, Section 5.13). Go has no built-in thread pool, because goroutines are cheap; to bound concurrency, Go code typically starts a fixed number of worker goroutines reading from a channel, or uses a buffered channel as a semaphore. Node.js runs JavaScript on one thread and uses a small internal thread pool (libuv's) for blocking work such as file-system calls (Section 5.13).

### Guarantees and non-guarantees

A pool guarantees an upper bound on the number of threads running its tasks, and that every accepted task is either run or (on a forced shutdown) handed back to you. It does not guarantee execution order (a multi-worker pool can run tasks out of submission order), a bound on queueing delay (unless the queue is bounded), that exceptions are reported, that tasks finish on shutdown unless you wait for them, or that the downstream services the tasks call can handle the concurrency.

### Failure and diagnosis

| Failure | Symptom | Evidence |
|---|---|---|
| Unbounded queue under overload | Memory climbs; latency climbs; eventually out-of-memory | Queue-length metric trending up; heap dump full of queued tasks |
| Pool too small for I/O-bound work | Low CPU, high latency, all workers busy | Thread dump: all workers in socket reads; queue non-empty |
| Pool too large | High context-switch rate, throttling, downstream overload | `vmstat` `cs` high (Chapter 3, Section 3.19); database at its connection limit |
| Thread-starvation deadlock | Pool hangs completely, CPU near zero | Thread dump: every worker waiting on a `Future` whose task is in the same pool's queue |
| Swallowed task exceptions | Jobs "silently don't happen" | No error logs; `Future`s never inspected; a rejection or error counter would have shown it |
| Pool created per request | Thread count grows without limit; thread creation starts failing | Thread count metric; thread dump with thousands of near-identical auto-generated pool thread names |

### Backend connection

A server has many pools, each a deliberate bound: the request-handling pool, database connection pools, HTTP client connection pools, background job executors, scheduled-task executors. Separating them is a resilience technique (a **bulkhead**, Chapter 24): a slow dependency can exhaust only the pool used to call it, not the threads that serve health checks. Naming pool threads ("orders-db-worker-3") makes thread dumps readable. Sizing and queueing for servers specifically are Chapter 11.

---

## 5.12 Futures, Promises, and async/await

### Definition

A **future** is an object that represents the result of a computation that may not have finished yet: a placeholder that will eventually hold a value or an error. A **promise** is the writable side of the same thing: the object through which the producer of the result *completes* the future, exactly once, with a value or an error. (Some languages use the two words interchangeably; JavaScript calls the combined object a `Promise`.) A **continuation** is the code to run once the future completes. **async/await** is language syntax that lets you write code that waits for futures in a straight-line style, while the compiler or runtime turns each `await` into a suspension point and the rest of the function into a continuation.

### Why they exist

Section 5.11 lets you *start* work on another thread. You then need a way to get the result back, to find out whether it failed, and to start dependent work when it is done. The naive approaches are bad: a shared variable plus a flag (a visibility problem, Section 5.3), or a blocking `join` that ties up the calling thread while it waits, which wastes a thread per outstanding operation. Futures package the result, the error and the "done" signal (with the correct happens-before edge) into one object. Continuations and async/await go further and let a program have thousands of operations in flight without a thread blocked on each one.

### Mechanism: states and completion

```text
                 complete(value)
   PENDING  ------------------------->  FULFILLED (value)
      |
      |          fail(error)
      +---------------------------->    REJECTED / FAILED (error)

   Transitions happen at most once. Later attempts to complete are ignored (or are errors).
   Continuations registered before completion run when it completes;
   continuations registered after completion run (or are scheduled) immediately.
```

Internally a future is a small state machine protected by a CAS on its state field (Section 5.4): the first `complete` wins, and it then runs or schedules the list of registered continuations. Completing a future *happens-before* any continuation or any `get` that observes the result, so the value is safely published.

There are two ways to consume a future:

- **Blocking:** `value = future.get()` waits on the calling thread until completion. Simple, but it occupies a thread while waiting, and it is how thread-starvation deadlock (Section 5.11) happens.
- **Non-blocking (continuation):** `future.then(f)` registers `f` to run when the value is ready, and returns immediately with a *new* future for `f`'s result.

### Composition

Continuations compose into pipelines without blocking:

```pseudo
// get user, then their orders, then price them, with error handling
fetch_user(id)                                  // Future<User>
    .then_compose(user -> fetch_orders(user))   // Future<List<Order>>  (flatten: f returns a future)
    .then(orders -> total_price(orders))        // Future<Money>        (map: f returns a value)
    .on_error(err -> Money.zero())              // recover from any earlier failure
    .then(total -> respond(total))

// run independent operations concurrently, then combine
all_of(fetch_profile(id), fetch_recommendations(id), fetch_cart(id))
    .then((profile, recs, cart) -> render(profile, recs, cart))
```

The difference between "map" (`then` with a function returning a value) and "flatten" (`then_compose` with a function returning another future) matters: using map with a function that returns a future gives you a future of a future.

### Callback hell

Before futures, asynchronous APIs took callbacks directly. Dependent steps nest, and error handling has to be repeated at every level:

```pseudo
fetch_user(id, (err, user) ->
    if err: return respond_error(err)
    fetch_orders(user, (err, orders) ->
        if err: return respond_error(err)
        fetch_prices(orders, (err, prices) ->
            if err: return respond_error(err)
            save_total(sum(prices), (err) ->
                if err: return respond_error(err)
                respond_ok()
            )
        )
    )
)
```

This "pyramid of doom" or **callback hell** is hard to read, makes errors easy to drop (forget one `if err`), and makes loops, early returns and `try/finally` cleanup hard to express. Futures flatten the nesting into a chain; async/await turns it back into ordinary-looking code.

### async/await: what the compiler does

```pseudo
async function checkout(id):
    user   = await fetch_user(id)
    orders = await fetch_orders(user)
    try:
        total = await price(orders)
    catch PricingError:
        total = 0
    return total
```

Calling `checkout` does not run it to completion. It starts running until the first `await` of a future that is not yet complete, then **suspends**: it saves its local variables and its position, registers "resume me here" as a continuation on the awaited future, and returns its own (pending) future to the caller. The thread is free to run other work. When `fetch_user` completes, the continuation is scheduled; the function resumes after the `await` with `user` set, and so on.

For most languages with `async`/`await` (C#, JavaScript, Python, Rust, Kotlin), the compiler implements this by turning the function into a **state machine** object whose fields hold the local variables and whose `state` field records which `await` it is suspended at:

```pseudo
class CheckoutStateMachine:
    state = 0; id; user; orders; total; result_promise

    resume(value_or_error):
        switch state:
            case 0:
                state = 1; fetch_user(id).on_complete(this.resume); return
            case 1:
                user = value_or_error.get_or_throw()
                state = 2; fetch_orders(user).on_complete(this.resume); return
            case 2:
                orders = value_or_error.get_or_throw()
                state = 3; price(orders).on_complete(this.resume); return
            case 3:
                total = value_or_error is PricingError ? 0 : value_or_error.get_or_throw()
                result_promise.complete(total)
```

This is a **stackless coroutine** (Chapter 3, Section 3.8): the suspended function's state lives in a heap object, not on a thread's stack, so it costs a small allocation rather than a whole thread stack.

### async/await versus threads

| | Blocking threads | async/await |
|---|---|---|
| Who decides when a task pauses | The OS scheduler, at any instruction (preemptive) | The task itself, only at `await` (cooperative) |
| Cost of a waiting task | A whole thread (stack, kernel structures) | A small heap object (the state machine) |
| Concurrent waits per process | Thousands, limited by memory and scheduling | Hundreds of thousands, commonly |
| Races between two `await`s | Possible anywhere | Within one event-loop thread, code between two `await`s runs without interruption by other tasks on that loop; state can still change *across* an `await` |
| Blocking call by mistake | Blocks only that thread | Blocks the whole event loop (or executor thread) and every task on it |
| CPU-bound work | Parallel across cores | Does not add parallelism; long computation without an `await` starves other tasks |
| Stack traces and debugging | Natural | Fragmented across continuations (runtimes add "async stack traces" to help) |

The key point: async/await reduces the **cost of waiting**. It does not add CPU, database connections or downstream capacity. A service that is slow because its database is saturated is not made faster by rewriting it with async/await.

**Function colouring.** In most async/await languages, an `async` function can only be awaited from another `async` function, and calling a blocking function from async code blocks the loop. Code splits into two "colours" that do not mix freely, and an async function deep in the call stack forces every caller to become async. (The term comes from Bob Nystrom's essay "What Color is Your Function?".) Runtimes with stackful coroutines (Go's goroutines, Java's virtual threads, Section 5.13) avoid colouring: ordinary blocking-style code suspends cheaply.

### Cancellation, timeouts and structured concurrency

A future represents a result, not the work. Abandoning a future (because a timeout fired, or because the user disconnected) does not necessarily stop the work: the database query keeps running, and its result is thrown away. Cancellation must be propagated explicitly (a cancellation token, a context object with a deadline, interrupting the task). And remember that a timed-out operation's outcome is **unknown**, not failed (Chapter 1, Section 1.2): the write may have happened.

**Structured concurrency** is the discipline that every concurrent task is started inside a scope that waits for all of its tasks to finish before the scope ends, cancels the others if one fails, and propagates their errors. It prevents leaked background tasks and lost errors, just as structured programming prevented arbitrary `goto`s. Examples: Python's `asyncio.TaskGroup` (Python 3.11+), Kotlin's coroutine scopes, Go's `errgroup` package, and Java's `StructuredTaskScope` (a preview API in recent Java versions at the time of writing).

> **Java lens:** `Future<V>` (from `ExecutorService.submit`) supports only blocking `get()` and `cancel()`. `CompletableFuture<V>` is both a future and a promise (`complete`, `completeExceptionally`) and supports composition: `thenApply` (map), `thenCompose` (flatten), `thenCombine`, `allOf`/`anyOf`, `exceptionally`/`handle` (errors), `orTimeout` and `completeOnTimeout` (Java 9+). Methods ending in `Async` run the continuation on an executor (the common `ForkJoinPool` unless you pass one), others may run it on whatever thread completed the previous stage. Running blocking I/O on the common pool is a classic mistake: it is sized for CPU-bound work, so a few blocked tasks starve every other user of it. Java has no `async`/`await` keywords; virtual threads (Section 5.13) are its answer to the same problem.

> **Other runtimes:** JavaScript `Promise` with `.then/.catch` and `async`/`await`; an `async` function always returns a promise, and an unhandled rejection is reported by the runtime (in recent Node.js versions it terminates the process by default). Python `asyncio`: `async def` defines a coroutine, `await` suspends it, `asyncio.gather` runs several concurrently, and `asyncio.to_thread` (or `loop.run_in_executor`) moves a blocking call to a thread so it does not block the loop. Go has no futures in the language; a goroutine plus a channel for the result plays the same role.

### Guarantees and non-guarantees

A future is completed at most once, and its completion happens-before any observer of its result. It does not guarantee that the work will finish, that cancelling the future stops the work, that errors will be observed by anyone, or which thread runs a continuation (unless you specify an executor).

### Failure and diagnosis

- **Blocking inside async code** (a synchronous database driver, a `sleep`, heavy JSON parsing): every task on the loop stalls; event-loop lag (Chapter 4, Section 4.5) spikes; latency of all requests rises together. Evidence: an event-loop-lag metric; a profiler showing the loop thread inside a blocking call.
- **Unobserved failures:** a future that failed but was never awaited; work silently not done. Evidence: missing side effects with no error logs; runtime warnings about unhandled rejections or exceptions never retrieved.
- **Leaked tasks:** fire-and-forget tasks that outlive the request, keep resources and run after the client has gone. Evidence: growing numbers of pending tasks or goroutines.
- **Blocking `get()` on pool threads:** thread-starvation deadlock (Section 5.11).

### Backend connection

Calling several downstream services concurrently (fan-out) and combining the results is the most common use: three independent 100 ms calls take about 100 ms in parallel instead of 300 ms sequentially. Every such fan-out needs a timeout per call, a decision about what to do when one fails (fail the whole request, or degrade), and cancellation of the rest when the request is abandoned. Those patterns are developed in Chapters 11 and 24.

---

## 5.13 Execution Models

### Definition

An **execution model** is the way a runtime maps many concurrent tasks onto the operating system's threads and the machine's cores: how a task is represented, who schedules it, and what happens when it waits. Chapter 3 (Section 3.8) listed the threading models (1:1 kernel threads, N:1 green threads, M:N, coroutines); this section shows how each works as a programming model and what it costs.

### Why the choice matters

Every backend runtime has to answer the question from Section 5.1: how do we keep thousands of mostly-waiting requests in progress cheaply, while still using all cores for computation? The answers trade the cost of a waiting task against programming convenience and against the damage a single misbehaving task can do.

### Model 1: thread-per-task

Each task (often each request or each connection) gets its own kernel thread, from a pool (Section 5.11) or created on demand. When the task waits for I/O, its thread blocks in a system call and the kernel runs another thread.

```text
   requests          kernel threads (1:1)              kernel scheduler
   --------          --------------------              ----------------
   req 1  -------->  T1: parse -> query DB [blocked in read()] ...
   req 2  -------->  T2: parse -> compute -> respond           <-- runs on core 0
   req 3  -------->  T3: parse -> call API [blocked in read()] ...
   req 4  -------->  T4: [waiting in pool queue]                <-- no free thread
                                                                    cores: 0, 1
```

**Strengths:** simple, sequential code; blocking calls are fine; CPU-bound work runs in parallel; stack traces are natural; preemption means one slow task cannot starve the others. **Weaknesses:** each waiting task costs a kernel thread (memory for its stack, scheduling overhead, context switches of a few microseconds each); a pool bounds concurrency, so at high concurrency (tens of thousands of slow connections) you run out of threads. This is the model of classic servlet containers and of PostgreSQL's process-per-connection and MySQL's thread-per-connection designs (Chapter 3, Section 3.9).

### Model 2: the event loop

A single thread runs a loop that waits for I/O readiness on all sockets at once (with `epoll` or `kqueue`, Chapter 4, Section 4.5) and runs short callbacks (or resumes coroutines) when their I/O is ready. Tasks never block; they register what they are waiting for and return to the loop.

```text
                      +---------------------------------------------+
                      |           EVENT LOOP (one thread)           |
                      |                                             |
   timers ----------->|   1. run expired timer callbacks            |
                      |   2. run queued callbacks / ready tasks  <--+---- task queue
                      |      (each runs until it returns or awaits) |     [cb7, cb8, cb9]
                      |   3. ask the I/O poller: what is ready?     |
                      |      epoll_wait(timeout = next timer)  -----+---> kernel
                      |   4. for each ready socket: queue its    <--+---- ready FDs
                      |      callback                               |     (sock 12 readable,
                      |   5. repeat                                 |      sock 40 writable)
                      +---------------------------------------------+
                                       |
                    blocking work (file I/O, DNS lookups, CPU-heavy crypto)
                                       v
                      +---------------------------------------------+
                      |  helper thread pool: runs it, then queues   |
                      |  the completion callback back to the loop   |
                      +---------------------------------------------+
```

**Strengths:** a waiting task costs only a small callback or coroutine object, so one thread can handle tens of thousands of connections; there is no preemption on the loop thread, so code between two awaits runs without interruption by other tasks (no data races among tasks on the same loop). **Weaknesses:** one thread uses one core (run one loop per core, or several processes, to use more); any blocking call or long computation stalls *every* task on the loop; code must be written in callback or async/await style (function colouring, Section 5.12).

> **Node.js lens:** Node runs JavaScript on one thread with an event loop provided by the libuv library. Each loop iteration goes through phases: timers (`setTimeout`), pending callbacks, the poll phase (waiting for I/O with `epoll`/`kqueue` and running I/O callbacks), the check phase (`setImmediate`) and close callbacks. Between callbacks, Node drains two higher-priority queues: the `process.nextTick` queue and then the promise **microtask** queue, so promise continuations run before the loop moves on. Network I/O is non-blocking on the loop thread; file-system operations, `dns.lookup`, and some crypto and compression functions run on libuv's thread pool (by default 4 threads in libuv at the time of writing, configurable with `UV_THREADPOOL_SIZE`). CPU-heavy work belongs in `worker_threads` or a separate process. A synchronous call such as `fs.readFileSync` or a long `JSON.parse` blocks the whole server.

### Model 3: M:N scheduling (green threads, virtual threads, goroutines)

The runtime multiplexes many lightweight user-level tasks (M of them) onto a small pool of kernel threads (N, usually about one per core). Each task has its own stack (it is a **stackful** coroutine), so code is written in ordinary blocking style. When a task performs an operation that would block (a socket read, a lock, a channel receive, a sleep), the runtime **parks** it: it saves its stack, registers the socket with its internal poller (`epoll`), and runs another task on the same kernel thread. When the socket becomes ready, the task is put back on a run queue.

```text
   lightweight tasks (M = 10,000)                runtime scheduler                 kernel threads (N = 4)
   ------------------------------                -----------------                 ----------------------
   G1  [running]  ---------------------------->  mounted on                 --->   carrier/M 1 (core 0)
   G2  [running]  ---------------------------->  mounted on                 --->   carrier/M 2 (core 1)
   G3  [running]  ---------------------------->  mounted on                 --->   carrier/M 3 (core 2)
   G4  [running]  ---------------------------->  mounted on                 --->   carrier/M 4 (core 3)
   G5  [runnable] -- in run queue (work stealing between per-thread queues)
   G6..G9000 [parked: waiting on socket]  <--  netpoller (epoll) wakes them when ready
   G9001.. [parked: waiting on lock/channel]
```

**Strengths:** the programming model of threads (sequential, blocking-style code, no colouring) with a cost per waiting task close to that of an event loop; parallelism across all cores. **Weaknesses:** the runtime must intercept every blocking operation, and an operation it cannot intercept (a blocking system call it does not know about, native code) blocks the underlying kernel thread; tasks still share memory, so all the synchronisation of this chapter still applies; and, as always, cheap waiting does not add CPU, connections or downstream capacity.

> **Go lens:** goroutines are scheduled by the Go runtime's **G–M–P** scheduler: G is a goroutine, M an OS thread ("machine"), P a "processor" (a scheduling context with its own run queue; there are `GOMAXPROCS` of them, by default the number of CPUs, and recent Go versions also take a container's cgroup CPU limit into account). A goroutine starts with a small stack (a few kilobytes) that grows and shrinks as needed, so a million goroutines is feasible. Network I/O goes through the runtime's netpoller (`epoll` on Linux). When a goroutine enters a blocking system call (e.g. a file read), the runtime hands its P to another M so other goroutines keep running. Since Go 1.14, the runtime can preempt a goroutine in a tight loop asynchronously (using signals), so one CPU-bound goroutine cannot monopolise a P. Channels (`ch <- v`, `v := <-ch`, `select`) are the preferred way to communicate (Section 5.10).

> **Java lens:** **virtual threads** (final in Java 21, JEP 444) are `Thread` objects scheduled by the JVM onto a small pool of **carrier** (platform) threads, by default a `ForkJoinPool` with one carrier per core. When a virtual thread blocks on a socket, a `java.util.concurrent` lock or a `sleep`, the JVM **unmounts** it (copies its stack frames to the heap) and mounts another; when the operation can proceed, the virtual thread is re-mounted, possibly on a different carrier. Existing blocking code (JDBC, `InputStream`) works unchanged. Points to know: (1) a virtual thread is **pinned** to its carrier, so blocking blocks the carrier, while it runs native code, and, in Java 21 to 23, while it is inside a `synchronized` block; Java 24 (JEP 491) removed the `synchronized` limitation. The JFR event `jdk.VirtualThreadPinned` reports pinning. (2) Do not pool virtual threads; create one per task (`Executors.newVirtualThreadPerTaskExecutor()`), and limit access to scarce resources with a semaphore or a connection pool instead. (3) `ThreadLocal`s still work, but with a million threads, per-thread caches multiply memory use. (4) They help I/O-bound work, not CPU-bound work.

### Model 4: actors (brief)

An **actor** is an independent unit of computation with private state and a **mailbox**: actors interact only by sending each other asynchronous messages, and each actor processes its messages one at a time. Because an actor's state is never touched by anyone else, it needs no locks (message passing, Section 5.10). Erlang/Elixir processes and the Akka toolkit (on the JVM) are the best-known implementations; the Erlang runtime schedules millions of actors M:N with preemption and isolates their heaps, so one actor crashing does not corrupt others, and supervisors restart failed actors. Actors move the hard problems rather than removing them: mailboxes can grow without bound (backpressure is needed), request–reply cycles between actors can deadlock, and message ordering is guaranteed only between a given pair of actors (in typical implementations).

### Comparison table

| | OS (kernel) thread | Green thread (N:1, classic) | Stackless coroutine (async/await) | Java virtual thread | Goroutine |
|---|---|---|---|---|---|
| Scheduled by | Kernel, preemptive | Runtime, usually cooperative, on one kernel thread | Event loop/executor, cooperative at each `await` | JVM, on carrier threads (M:N); switches at blocking operations | Go runtime (M:N), switches at blocking operations, plus asynchronous preemption |
| Memory per task | Stack reserved commonly in the hundreds of KB to a few MB (grows on demand physically) | Small runtime-managed stack | One heap object holding the locals live across `await`s | Stack frames stored on the heap while unmounted; grows as needed | Starts at a few KB, grows and shrinks |
| Creation and switch cost | Microseconds; switch involves the kernel | Cheap, in user space | Very cheap (allocation, function call) | Cheap, in user space | Cheap, in user space |
| Typical count per process | Thousands | Many, but only one runs at a time | Hundreds of thousands | Millions | Millions |
| Parallelism across cores | Yes | No (N:1) | Only with several loops or executors | Yes | Yes |
| When a task makes a blocking call | Only that thread blocks | **All** green threads on that kernel thread block | The whole loop blocks (unless moved to a thread pool) | Virtual thread is unmounted (except when pinned, then the carrier blocks) | Goroutine parks; for blocking syscalls the runtime moves other goroutines to another thread |
| Code style | Sequential, blocking | Sequential | `async`/`await`, coloured functions | Sequential, blocking | Sequential, blocking; channels |
| Need for locks between tasks | Yes | Fewer (no parallelism), but still at yield points | Not between tasks on one loop, except across `await`; yes across loops/threads | Yes | Yes (or channels) |

### The Global Interpreter Lock (GIL)

#### Definition

The **Global Interpreter Lock (GIL)** is a mutex inside the standard CPython interpreter that a thread must hold in order to execute Python bytecode. Only one thread per interpreter process can run Python code at any moment, even on a machine with many cores. (Chapter 2, Section 2.9 introduced the term.) Other runtimes have similar locks; Ruby's standard interpreter (CRuby) has the equivalent Global VM Lock.

#### Why it exists

CPython manages memory with reference counting (Chapter 2, Section 2.7): every object has a count that is updated on every reference creation and deletion, constantly, from every thread. Making each update atomic or individually locked would slow down single-threaded code considerably. Interpreter internals (memory allocator, object dictionaries, many C extensions) also assumed a single thread. One global lock made the interpreter and its C extensions thread-safe cheaply and kept single-threaded performance high, at the cost of parallelism.

#### Mechanism

- A thread running Python code holds the GIL. Every so often (the **switch interval**, by default 5 ms, `sys.getswitchinterval()`), if another thread is waiting, the running thread is asked to release the GIL so the waiting thread can take it.
- A thread **releases the GIL around blocking I/O** (socket reads, file reads, `time.sleep`) and while running C code that does not touch Python objects. Well-written C extensions (NumPy's array operations, compression and cryptography libraries) release it during heavy computation.

```text
Two CPU-bound Python threads on a 4-core machine:

core 0 | T1 runs |         | T1 runs |         | T1 runs |
core 1 |         | T2 runs |         | T2 runs |         |
       +-----------------------------------------------------> time
         only one holds the GIL at a time: total throughput ~= one core (or less, from switching)

Two I/O-bound Python threads:

T1 | run | waiting on socket (GIL released) ........ | run |
T2 |     | run | waiting on DB (GIL released) ........... | run |
         the waits overlap: threading helps
```

#### Consequences

1. **Threads give no CPU parallelism for pure-Python code.** Splitting a CPU-bound Python computation across threads does not make it faster and can make it slower (GIL hand-off overhead). Use processes (`multiprocessing`, `ProcessPoolExecutor`, or several server worker processes behind one listener, as Python web servers commonly do), or move the hot loop into a C extension that releases the GIL.
2. **Threads do help I/O-bound code**, because the GIL is released while waiting. A threaded Python web server or crawler can overlap many network waits.
3. **The GIL does not make your code thread-safe.** It guarantees that individual bytecode instructions (and some C-implemented operations, such as `list.append`) are not interleaved; it does not make multi-bytecode operations atomic. `count += 1` compiles to several bytecodes (load, add, store), and a thread switch can occur between them, so the lost update of Section 5.2 happens in Python too. Use `threading.Lock`.
4. **Free-threaded Python.** PEP 703 made the GIL optional: Python 3.13 introduced an experimental free-threaded build, and Python 3.14 made it an officially supported (but still optional) build. The standard build still has a GIL at the time of writing, and C extensions must be adapted to run without it.

> **Python lens (asyncio):** `asyncio` is an event loop (Model 2) in a single thread. Coroutines (`async def`) suspend at `await`; one blocking call (`time.sleep`, `requests.get`, a synchronous database driver) blocks every coroutine. Use asynchronous libraries, or `asyncio.to_thread` for blocking calls. The GIL is not involved in asyncio's scheduling, because only one thread is running Python code anyway.

### Guarantees and non-guarantees

No execution model removes the need to protect shared mutable state across tasks that run in parallel or that interleave at suspension points. An event loop guarantees that one task's code between suspension points is not interleaved with other tasks on the same loop; it does not guarantee anything across `await`s, across loops, or across processes. M:N runtimes guarantee cheap waiting for operations they intercept; they do not guarantee that every blocking operation is intercepted. The GIL guarantees one bytecode at a time; it does not make compound operations atomic.

### Failure and diagnosis

| Model | Characteristic failure | Symptom and evidence |
|---|---|---|
| Thread-per-task | Pool exhaustion when a dependency slows | All threads blocked in socket reads (thread dump); queue grows; CPU low |
| Event loop | A blocking call or CPU-heavy task on the loop | Event-loop lag metric spikes; all requests slow together; profiler shows the loop thread in one function |
| Virtual threads | Pinning or exhaustion of a downstream resource | JFR `jdk.VirtualThreadPinned` events; carriers all busy; or millions of virtual threads waiting on a 20-connection pool |
| Goroutines | Goroutine leak (blocked forever on a channel nobody writes to) | Goroutine count grows steadily (`runtime.NumGoroutine`, `/debug/pprof/goroutine`) |
| Python threads | CPU-bound work in threads | One core at 100%, others idle; adding threads does not help |

### Backend connection

The execution model determines how a server behaves under load: how many requests it can have in flight, what happens when a dependency becomes slow, and which mistakes are fatal (a blocking call on an event loop; a missing timeout in a thread-per-request server). The models also differ in what limits concurrency: with cheap tasks (event loops, virtual threads, goroutines), the limit is no longer the number of threads, so you must impose limits explicitly on downstream resources. Chapter 11 develops how servers choose and size execution models, using queueing theory.

---

## 5.14 Testing and Diagnosing Concurrency Bugs

### Why concurrency bugs are hard to find

A concurrency bug is a property of an *interleaving*, and the number of possible interleavings is enormous. Two threads that each execute *n* atomic steps can be interleaved in C(2n, n) ways: for *n* = 10, that is 184,756 different orders; for realistic code with hundreds of steps and several threads, the number is astronomically large. A test run explores one interleaving, chosen by the scheduler, the timing of interrupts, the cache state and the CPU model. Three properties follow:

- **Non-determinism:** the same test passes thousands of times and fails once; the failure cannot be reproduced on demand.
- **The probe effect:** observing the program (logging, a debugger, a breakpoint) changes its timing and often adds synchronisation, so the bug disappears when you look for it (the Heisenbug of Section 5.2).
- **Environment dependence:** bugs that need many cores, weak memory ordering (ARM), JIT-optimised code or production load do not appear on a developer laptop.

So the strategy is twofold: make concurrency bugs *less likely to exist* (by design: confinement, immutability, library structures, a small number of well-reviewed synchronised components), and *more likely to show up* when they do exist (stress tests and tools).

### Testing techniques

**1. Stress tests with maximal contention.** Run the operation from many threads at once, many times, and check an invariant at the end. Start all threads at the same instant with a barrier or latch, so that they actually collide instead of running one after another:

```pseudo
test_counter_is_thread_safe():
    counter = new Counter()
    start = new Latch(1)
    threads = []
    repeat 16 times:
        threads.add(start_thread(() ->
            start.await()                      // all threads wait here...
            repeat 100_000 times: counter.increment()
        ))
    start.count_down()                         // ...and are released together
    join all threads
    assert counter.value() == 16 * 100_000     // the invariant
```

Run it on a machine with more cores than a laptop, on ARM as well as x86 if you deploy to both, many times, and with optimisation enabled (after JIT warm-up). A passing stress test is evidence, not proof.

**2. Dynamic race detectors.** Tools that instrument memory accesses and synchronisation operations at run time, track happens-before relationships (Section 5.3), and report two conflicting accesses that are not ordered by happens-before, even if the run did not happen to produce a wrong result. **ThreadSanitizer** (TSan, for C and C++ with Clang and GCC, `-fsanitize=thread`) and Go's built-in race detector (`go test -race`, `go run -race`, built on the same technology) are the standard examples. Go's report begins with `WARNING: DATA RACE` and shows the stack traces of both accesses and where each goroutine was created. Limits: they find only races in code paths that actually execute during the run; they slow programs down considerably (Go's documentation cites memory use increased 5–10× and execution time 2–20×), so they are used in tests and canaries rather than everywhere in production; and they detect *data races*, not higher-level race conditions such as check-then-act done with properly synchronised calls.

**3. Concurrency test harnesses.** Tools that run tiny concurrent tests (litmus tests, Section 5.3) millions of times and report how often each outcome occurred, flagging outcomes the specification forbids. OpenJDK's **jcstress** is the Java example. Model checkers and **deterministic simulation** go further: they control the scheduler and systematically explore (or randomly sample with a seed that can be replayed) many interleavings, making failures reproducible. These are used for concurrency libraries and distributed systems rather than everyday application code.

**4. Injecting delays.** Inserting a short sleep or a yield between the check and the act (in a test build) widens the race window and turns a one-in-a-million failure into a reliable one, which is useful for demonstrating a suspected bug and verifying its fix.

### Thread dumps

A **thread dump** is a snapshot of every thread in a process: its name, its state, its current stack trace, and, in managed runtimes, the locks it holds and the lock it is waiting for. It is the single most useful tool for diagnosing hangs, deadlocks, pool exhaustion and contention.

How to take one:

| Runtime | Command |
|---|---|
| JVM | `jstack <pid>`, `jcmd <pid> Thread.print`, or `kill -3 <pid>` (SIGQUIT: the dump is written to the JVM's standard output) |
| Go | `kill -QUIT <pid>` (prints all goroutine stacks and exits), or, without stopping, the `/debug/pprof/goroutine?debug=2` endpoint if `net/http/pprof` is enabled |
| Python | `py-spy dump --pid <pid>` (external sampling tool); `faulthandler.dump_traceback()` from inside |
| Native (C, C++, Rust) | `gdb -p <pid> -batch -ex "thread apply all bt"`, or `eu-stack -p <pid>` |

#### Java lens: reading a JVM thread dump

JVM dumps carry the most lock information of the runtimes above, so they make a good worked example. The JVM's thread states, and what they usually mean:

| State | Meaning | Typical cause in a server |
|---|---|---|
| `RUNNABLE` | Running on a CPU, ready to run, or **inside a native call such as a socket read** | CPU work, or blocked in I/O (the JVM reports threads blocked in native socket reads as `RUNNABLE`) |
| `BLOCKED` | Waiting to enter a `synchronized` block held by another thread | Monitor contention or deadlock |
| `WAITING` | Waiting indefinitely: `Object.wait()`, `LockSupport.park()` (used by `ReentrantLock`, `Semaphore`, queues, `Future.get()`) | Idle pool workers (`take()` on an empty queue), lock or permit waits, blocking on a future |
| `TIMED_WAITING` | Same, with a timeout: `sleep`, `wait(ms)`, `parkNanos` | Sleeps, timed waits, idle workers with keep-alive |

**An abridged deadlock dump.** (Real output includes thread IDs, addresses and more frames; the exact format varies between JDK versions.)

```text
"transfer-worker-7" #41 ... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.bank.TransferService.transfer(TransferService.java:58)
        - waiting to lock <0x00000007156a2e10> (a com.example.bank.Account)
        - locked <0x00000007156a2d88> (a com.example.bank.Account)
        ...

"transfer-worker-3" #37 ... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.bank.TransferService.transfer(TransferService.java:58)
        - waiting to lock <0x00000007156a2d88> (a com.example.bank.Account)
        - locked <0x00000007156a2e10> (a com.example.bank.Account)
        ...

Found one Java-level deadlock:
=============================
"transfer-worker-7":
  waiting to lock monitor ... (object 0x00000007156a2e10, a com.example.bank.Account),
  which is held by "transfer-worker-3"
"transfer-worker-3":
  waiting to lock monitor ... (object 0x00000007156a2d88, a com.example.bank.Account),
  which is held by "transfer-worker-7"
```

Reading it: worker 7 holds account `...2d88` and wants `...2e10`; worker 3 holds `...2e10` and wants `...2d88`. That is the two-account transfer of Section 5.8, and the fix is lock ordering by account ID. The JVM finds this cycle for monitors and `java.util.concurrent` locks; it cannot see deadlocks involving other resources (a database row lock, a connection pool, a future waiting on a full pool).

#### Triage method (any runtime)

Take three dumps a few seconds apart, then:

1. **Group threads by identical stack** (tools and simple scripts can do this). One stack shared by 190 of 200 request threads is your answer to "where are they all?".
2. **Separate stuck from busy.** A thread with the same stack in all three dumps is stuck (or doing one very long thing); a running thread whose stack changes is working.
3. **For threads waiting on a lock, find the holder.** Search the dump for the thread that holds the lock they are waiting for, and see what *it* is doing. Often it is in a socket read: a lock held during I/O (Section 5.5).
4. **For threads waiting on a pool or semaphore,** compare the number of waiters with the pool's size: a connection pool of 10 with 190 waiters is pool exhaustion (Chapter 18), not a CPU problem.

### Lock-contention profiling

A thread dump shows contention at one instant; a **lock-contention profile** measures where threads spend time waiting for locks over a period, aggregated by lock and by call stack.

- **JVM:** Java Flight Recorder (JFR) records `jdk.JavaMonitorEnter` (time blocked entering `synchronized`) and `jdk.ThreadPark` (time parked in `java.util.concurrent` locks) events with stacks; async-profiler has a lock-profiling mode (`-e lock`).
- **Go:** the **mutex profile** (enable sampling with `runtime.SetMutexProfileFraction`) shows where contended mutexes are released, weighted by the delay they caused; the **block profile** (`runtime.SetBlockProfileRate`) shows where goroutines block on channels, mutexes and selects. Both are served by `net/http/pprof`.
- **Native/Linux:** **off-CPU analysis** (tools from the BPF ecosystem, such as `offcputime` in BCC) shows where threads spend time *not* running and why; `perf lock` (where supported) analyses kernel lock events. Futex contention shows up as many `futex` system calls in `strace -c` (Chapter 4, Section 4.12).

A complementary metric view: for every pool and lock that matters, export *active*, *waiting* and *wait time* as metrics. Rising wait time with flat throughput is contention.

### Guarantees and non-guarantees

Stress tests and race detectors can prove the *presence* of bugs, never their absence. A clean race-detector run means no data race occurred *in the executed paths* during *that run*. A thread dump without a "deadlock found" message does not mean there is no deadlock: deadlocks through non-lock resources are invisible to the detector.

### Backend connection

In production, concurrency problems usually present as latency or availability problems, not as obvious errors: "requests hang", "p99 jumped", "throughput plateaued at 30% CPU". The response playbook is: check whether the CPU is busy or idle (Chapter 3, Section 3.19); take thread dumps and group them; look for many threads on one lock, one pool or one downstream call; confirm with contention profiles and pool metrics. Building that habit is the goal of the Diagnosis Lab at the end of this chapter, and the full production toolkit (metrics, traces, profiling) is in Chapter 26.

---

## Chapter Summary

Concurrency means several tasks in progress at once, with their steps interleaved in an order nobody controls; parallelism means several computations executing at the same instant. Backend software is concurrent mainly to overlap waiting (a request that waits 48 ms for a database leaves the CPU free for other requests), and parallel to use several cores. Correctness problems come from concurrency itself: they exist even on one core, because the scheduler can preempt a thread between any two instructions.

When tasks share mutable state, a **race condition** occurs if some interleaving produces a wrong result. The two shapes to recognise are **read-modify-write** (`count++` is load, add, store; two threads can both load the old value, and one update is lost) and **check-then-act** (two requests both see one item in stock and both sell it). The cure is to make the compound operation atomic, or to stop sharing the state. A correct critical-section protocol guarantees mutual exclusion, progress and bounded waiting.

Underneath, memory is not one big array. Each core puts writes in a **store buffer** before they reach the coherent caches; weakly ordered CPUs (ARM, POWER) reorder further; and compilers keep values in registers, hoist loads out of loops and reorder independent stores. Single-threaded code cannot tell, but other threads can: a stop flag is never seen, a published object is seen half-built, Peterson's algorithm lets both threads in. A **memory model** is the contract that tames this. Modern languages promise that **data-race-free programs behave sequentially consistently**, and **happens-before** says when a program is data-race-free: program order, plus synchronising edges (unlock → lock of the same mutex, atomic/volatile write → read that sees it, thread start and join, channel send → receive), closed under transitivity. Locks and atomics emit the **memory barriers** that enforce those edges. Visibility and atomicity are separate problems: `volatile` gives the first, not the second.

Everything is built on **hardware atomic instructions**: test-and-set, compare-and-swap and fetch-and-add, which hold a cache line exclusively while they read and write it. A CAS loop (read, compute, CAS, retry) is the core of optimistic concurrency; CAS cannot tell that a value went A → B → A (the **ABA problem**), which version tags, LL/SC or safe memory reclamation fix. A **spinlock** is a TAS or CAS loop, good only for tiny critical sections on dedicated cores. A **mutex** blocks instead; on Linux it is built on the **futex**, so an uncontended lock costs one atomic instruction and the kernel is involved only when threads collide. Locks vary in reentrancy, fairness (unfair is faster; fair prevents starvation) and mode (read–write locks admit many readers but can starve writers and deadlock on upgrade). Their cost under contention is serialisation: a 50 µs critical section caps throughput at 20,000 operations per second however many cores you add. Keep critical sections short and never hold a lock across I/O.

A **semaphore** is a counter of permits with no owner, used to limit concurrency to N and to signal events; a **condition variable** lets a thread sleep until a predicate on shared state becomes true, releasing its mutex atomically so that no wake-up is lost, and must always be waited on in a `while` loop because of Mesa semantics, spurious wake-ups and shared conditions. A **monitor** bundles a lock and condition variables with the data. These tools solve the classic problems: the **bounded buffer** (mutex plus two conditions, or three semaphores acquired counting-first), **readers–writers** (whose simple solution starves writers; a turnstile makes it fair) and **dining philosophers** (whose naive solution deadlocks; ordering the forks fixes it).

Locks bring **liveness failures**. **Deadlock** needs all four Coffman conditions (mutual exclusion, hold and wait, no preemption, circular wait); you **prevent** it by breaking one (global lock ordering breaks circular wait), **avoid** it by granting only requests that leave a **safe state** (the Banker's algorithm), or **detect** it by finding a cycle in the **wait-for graph** and **recover** by aborting a victim, as databases do. **Livelock** is busy non-progress; **starvation** is one participant never served; **priority inversion** is a high-priority task waiting behind a low-priority one that medium-priority work keeps from running, fixed by priority inheritance. **Lock-free** algorithms guarantee that some thread always progresses and **wait-free** ones that every thread does, but neither is automatically faster: contention is about cache lines, not about lock objects.

In practice, thread safety is a strategy chosen per piece of state: **confine** it, make it **immutable** and swap references, **synchronise** it, **delegate** to a concurrent structure (using its atomic compound operations), or give it to one owner and **pass messages**. **Thread pools** bound concurrency and reuse threads; size them near the core count for CPU-bound work and larger for I/O-bound work, but the downstream resource is usually the real limit, the queue must be bounded, and the rejection policy is your overload behaviour. **Futures** carry results and errors across threads; continuations and **async/await** let a waiting task cost a small heap object instead of a thread. The **execution models** (thread-per-task, event loop, M:N virtual threads and goroutines, actors) trade the cost of waiting against programming convenience and the harm one misbehaving task can do; none adds CPU, connections or downstream capacity. CPython's **GIL** lets only one thread run Python bytecode at a time: threads help I/O-bound Python code, not CPU-bound code, and the GIL does not make `count += 1` atomic. Concurrency bugs are non-deterministic, so you design them out, then hunt for the rest with stress tests, race detectors, thread dumps and contention profiles.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Concurrency (unsynchronised) | Each thread's own program order, as seen by itself | Any particular interleaving; visibility or order of writes to other threads |
| Thread-safe component | Each documented operation is atomic and correct under concurrency | Atomicity of sequences of operations (check-then-act); safety of objects stored in it |
| Memory model (DRF-SC) | Sequential consistency for programs without data races | Anything definite for racy programs (stale values, torn reads; undefined behaviour in C/C++) |
| `volatile` / atomic variable | Visibility and ordering of each access; atomic single reads/writes; atomic RMW methods | Atomicity of `x++` written as read-then-write; mutual exclusion; ordering with other variables unless they are accessed before/after in program order |
| CAS | Indivisible compare-and-replace of one location | Detection of ABA; success within any number of retries; atomicity across two locations |
| Fetch-and-add | One-instruction, wait-free (on x86) atomic increment | Scalability under heavy contention (cache line bouncing) |
| Spinlock | Mutual exclusion with very low hand-off latency | Efficiency if the holder is preempted or the section is long; usefulness on one core |
| Mutex | One holder at a time; unlock → lock happens-before | Fairness; freedom from deadlock; protection of code that skips the lock; cross-process or cross-machine exclusion |
| Read–write lock | Many readers or one writer | Faster than a mutex for short sections; writer progress (policy-dependent); safe upgrade |
| Semaphore | Concurrent holders ≤ initial count + signals | Ownership; detection of leaked or extra permits; which waiter wakes |
| Condition variable | Atomic release-and-sleep; mutex held on return | That the condition is true on return; memory of a signal with no waiters |
| Lock ordering | No deadlock among locks acquired in the order | Protection if any path breaks the order; no deadlock on non-lock resources |
| Banker's algorithm | The system stays in safe states if declared maxima hold | Usefulness when needs are unknown in advance; low overhead |
| Deadlock detection | Cycles are found and broken | That the aborted victim's work is safely retried |
| Lock-free structure | System-wide progress; no deadlock; tolerance of stalled threads | Per-thread progress; higher throughput; atomic multi-step use |
| Thread pool | Bounded threads; accepted tasks run or are returned | Order; bounded queueing delay with an unbounded queue; reported exceptions; downstream capacity |
| Future | Completed at most once; completion happens-before observation | That the work finishes; that cancellation stops the work; that errors are observed |
| Event loop | No interleaving between suspension points on one loop | Progress if a task blocks; parallelism; atomicity across `await` |
| Virtual threads / goroutines | Cheap waiting for intercepted blocking operations | More CPU, connections or downstream capacity; freedom from pinning or leaks; data-race freedom |
| GIL | One Python bytecode at a time per interpreter | Atomicity of compound Python operations; CPU parallelism for Python code |
| Race detector / stress test | Reports races that occur in executed paths | Absence of races elsewhere or in other runs |

---

## Interview Drill

**Q1. What is the difference between concurrency and parallelism?**
A strong answer must mention:
- Concurrency: several tasks in progress over the same period, steps interleaved; a property of program structure.
- Parallelism: simultaneous execution on several cores or machines; a property of execution.
- Concurrency is possible on one core (time slicing); parallelism requires multiple execution units.
- Servers are concurrent mainly to overlap I/O waits, and parallel to use cores; races arise from concurrency, even on one core.
- Amdahl's law limits parallel speed-up by the serial fraction (locks contribute to it).
*Common weak answer:* "They are the same thing, running things at the same time." This misses that single-core concurrency exists and is where race conditions start.

**Q2. What is a race condition, and how do you prevent one?**
A strong answer must mention:
- Correctness depends on the interleaving of operations on shared mutable state.
- The two shapes: read-modify-write (`count++` = load/add/store, lost update) and check-then-act (oversell, lazy init, TOCTOU), with an interleaving drawn.
- Prevention: make the compound operation atomic (atomic instruction, lock around the whole sequence, atomic library operation such as `computeIfAbsent`), or eliminate sharing (confinement, immutability, message passing).
- Every access path must use the same protection.
- Across instances of a service, an in-process lock is useless; use an atomic conditional update in the database (Chapter 17).
*Common weak answer:* "Use `synchronized`/a lock." Correct as a tool, but without the mechanism (which steps interleave) and without the cross-instance caveat it is a keyword answer.

**Q3. What is a critical section, and what must a solution to the critical-section problem guarantee?**
A strong answer must mention:
- Code that touches shared state and must not be executed by more than one thread at once.
- Mutual exclusion, progress (threads outside cannot block entry; the decision cannot be postponed forever), bounded waiting (a bound on how often others enter first).
- No assumptions about thread speeds or CPU counts.
- Peterson's algorithm satisfies them on a sequentially consistent machine but fails on real hardware without fences (store→load reordering).
*Common weak answer:* "Only one thread at a time." That is only the first requirement; strict alternation satisfies it and still violates progress.

**Q4. What is a memory model? What is a visibility problem?**
A strong answer must mention:
- The contract defining which values a read may return in a multithreaded program; it binds programmer, compiler and hardware.
- Causes of visibility/ordering surprises: store buffers (store→load reordering, even on x86), weak hardware ordering (ARM/POWER), compiler optimisations (register caching, hoisting loads out of loops, reordering stores).
- Example: a stop flag in a plain variable that the worker never sees because the load is hoisted; fix with an atomic/volatile variable.
- DRF-SC: data-race-free programs behave sequentially consistently; a data race is conflicting accesses not ordered by happens-before.
- Caches are coherent (MESI), so "stale cache" alone is not the full explanation.
*Common weak answer:* "Each thread has its own copy of the variable in its cache." Caches are coherent; the real causes are store buffers, reordering and compiler optimisations, and the answer omits the contract.

**Q5. What is happens-before?**
A strong answer must mention:
- A partial order: if A happens-before B, A's effects (and everything before A) are visible to B.
- Sources: program order within a thread; unlock → later lock of the same mutex; atomic/volatile write → read that sees it; thread start; thread join; channel send → receive.
- Transitivity: one synchronising edge publishes everything written before it (safe publication).
- Conflicting accesses not ordered by happens-before are a data race.
- Edges exist only through the *same* lock or variable.
*Common weak answer:* "It means A runs before B in time." Happens-before is about guaranteed visibility, not wall-clock order; two operations can occur in time order with no happens-before edge.

**Q6. What does `volatile` do in Java, and what doesn't it do?**
A strong answer must mention:
- Every read sees the latest write in happens-before order; a volatile write happens-before subsequent reads of it; no register caching; barriers emitted.
- It gives visibility and ordering, and atomic reads/writes of `long`/`double`.
- It does not make `x++` atomic and gives no mutual exclusion; use `AtomicInteger` or a lock.
- Needed for double-checked locking to be correct.
- Contrast: C/C++ `volatile` is not a threading tool; use `std::atomic`.
*Common weak answer:* "It makes the variable thread-safe." It does not make compound operations atomic.

**Q7. What is compare-and-swap, and what does lock-free mean?**
A strong answer must mention:
- CAS atomically replaces a location's value only if it equals an expected value, reporting success; implemented with `LOCK CMPXCHG` on x86 or LL/SC / LSE on ARM, holding the cache line exclusively.
- CAS loop: read, compute, CAS, retry; a failure means another thread succeeded.
- Lock-free: some thread always completes in finite steps even if others stall; wait-free: every thread does, in bounded steps.
- A CAS-loop counter is lock-free; fetch-and-add is wait-free.
- Lock-free is not automatically faster: cache-line contention, wasted retries, more atomics, reclamation costs.
*Common weak answer:* "Lock-free means no locks, so it is faster." It is a progress guarantee, not a performance guarantee.

**Q8. What is the ABA problem?**
A strong answer must mention:
- CAS checks equality of value, not absence of change; a value going A → B → A lets a stale CAS succeed.
- Timeline on a Treiber stack pop: read top = A and next = B, preempted; others pop A, pop B, free B, push A; CAS(top, A, B) succeeds and installs a freed node.
- Fixes: version-tagged references (double-width CAS, `AtomicStampedReference`), LL/SC, hazard pointers or epoch-based reclamation.
- Garbage collection prevents the pointer-reuse form, not value ABA.
*Common weak answer:* "When a value changes twice." Without the stale-CAS mechanism and a fix, it is just a label.

**Q9. What is the difference between a mutex and a semaphore? Between a mutex and a spinlock?**
A strong answer must mention:
- Mutex: owner, one holder, for mutual exclusion; enables reentrancy, priority inheritance and non-owner-unlock errors.
- Semaphore: a count of permits, no owner, any thread may signal; for limiting concurrency to N and signalling; binary semaphore ≠ mutex.
- Spinlock waits by busy-looping (TAS/CAS); good for tiny sections on dedicated cores; terrible if the holder is preempted, on one core, or for long sections.
- Blocking mutex sleeps; on Linux a futex makes the uncontended path one atomic instruction with no syscall; adaptive mutexes spin briefly then sleep.
*Common weak answer:* "A semaphore is a mutex that allows N threads." It omits ownership and the signalling use.

**Q10. What is a condition variable, and why must you wait in a `while` loop?**
A strong answer must mention:
- Lets a thread sleep until a predicate on shared state is true; used with the mutex protecting that state.
- `wait` atomically releases the mutex and sleeps (preventing lost wake-ups) and reacquires it before returning.
- `while` because of Mesa semantics (another thread may take the state first), spurious wake-ups, and broadcasts or shared condition variables.
- `signal` vs `broadcast`: one waiter for one condition vs several or mixed conditions; thundering herd with broadcast.
*Common weak answer:* "Because of spurious wake-ups." Only one of three reasons, and it misses the stolen wake-up under Mesa semantics.

**Q11. How do you solve the producer–consumer problem?**
A strong answer must mention:
- Bounded buffer: producers wait while full, consumers while empty; capacity provides backpressure.
- Mutex + two condition variables (`not_full`, `not_empty`) with `while` loops, signalling the opposite condition after each change.
- Semaphore version: `empty = N`, `full = 0`, `mutex = 1`; acquire the counting semaphore before the mutex.
- Acquiring the mutex first then waiting for a slot deadlocks (holding the lock the consumer needs).
- Real-world form: a bounded blocking queue in a thread pool.
*Common weak answer:* "Use a queue." Doesn't show how waiting, signalling and mutual exclusion are achieved.

**Q12. What are the four conditions for deadlock? How do you prevent, avoid and detect deadlock?**
A strong answer must mention:
- Mutual exclusion, hold and wait, no preemption, circular wait: all four are necessary.
- Prevention: break one; global lock ordering (e.g. lock accounts by ID) breaks circular wait; acquire-all-at-once breaks hold and wait; `tryLock` with release breaks no preemption.
- Avoidance: Banker's algorithm grants a request only if the resulting state has a safe sequence; needs declared maxima, so rare in servers; unsafe ≠ deadlocked.
- Detection: cycle in the wait-for graph (sufficient for single-instance resources); recovery by aborting a victim (as databases do), which must be retried safely.
- In practice, timeouts as a last line of defence.
*Common weak answer:* Lists the four conditions but cannot explain why lock ordering makes a cycle impossible, or how the Banker's safety check works.

**Q13. What is the difference between deadlock, livelock and starvation? What is priority inversion?**
A strong answer must mention:
- Deadlock: a set of threads blocked forever on each other; CPU idle.
- Livelock: threads active, reacting to each other, no progress (symmetric back-off); CPU busy; fix with randomised back-off.
- Starvation: one thread never gets the resource while others progress (unfair locks, reader preference, strict priority); fix with fairness or aging.
- Priority inversion: high waits on a lock held by low, medium preempts low; Mars Pathfinder; fixed with priority inheritance or ceiling.
*Common weak answer:* "They are all when threads get stuck." Misses the differences in whether threads are blocked, busy or selectively unlucky.

**Q14. How do you size a thread pool?**
A strong answer must mention:
- CPU-bound: about the number of available cores (the cgroup limit in a container).
- I/O-bound: N_cpu × U × (1 + W/C), with a worked number.
- The downstream resource (database connections, a dependency's capacity) is usually the real limit; more threads than connections just queue.
- Bounded queue and an explicit rejection policy (abort for fail-fast, caller-runs for backpressure); unbounded queues hide overload.
- Separate pools for different kinds of work; beware thread-starvation deadlock.
*Common weak answer:* "As many threads as possible" or "2 × cores." Ignores the CPU-bound/I/O-bound distinction and downstream limits.

**Q15. What is the difference between async/await and threads? How does an event loop work?**
A strong answer must mention:
- Event loop: one thread; waits for readiness on all sockets (`epoll`), runs ready callbacks and timers; tasks must not block.
- async/await: the compiler turns a function into a state machine (stackless coroutine); `await` suspends and registers a continuation; the thread runs other tasks meanwhile.
- Threads: preemptive, cost a stack each, blocking is fine, parallel on cores.
- async is cooperative: a blocking call or long computation stalls every task on the loop; function colouring.
- Neither adds CPU or downstream capacity; async reduces the cost of waiting.
*Common weak answer:* "Async is faster than threads." It is cheaper per waiting task, not faster per unit of work, and is worse for CPU-bound work.

**Q16. Compare OS threads, green threads, coroutines, virtual threads and goroutines.**
A strong answer must mention:
- OS threads: 1:1, kernel-scheduled, preemptive, relatively large stacks, microsecond switches.
- Green threads (N:1): user-space, one kernel thread, no parallelism, one blocking call blocks all.
- Stackless coroutines: state machines with `await`; cooperative; coloured functions.
- Virtual threads and goroutines: M:N stackful tasks; runtime parks them on blocking operations via an internal poller; cheap enough for millions.
- Caveats: pinning (virtual threads), goroutine leaks, unintercepted blocking calls; all still need synchronisation for shared state.
*Common weak answer:* "Virtual threads are just faster threads." They are cheaper to block, not faster at computing.

**Q17. What is the GIL, and what are its consequences?**
A strong answer must mention:
- A mutex in CPython that must be held to execute Python bytecode; one thread runs Python code at a time per process.
- Exists because of reference counting and non-thread-safe interpreter internals and C extensions.
- Released on blocking I/O and by C extensions during heavy work: threads help I/O-bound code, not CPU-bound code.
- Workarounds: multiprocessing / multiple worker processes, C extensions, asyncio for I/O concurrency.
- The GIL does not make `count += 1` atomic; free-threaded builds (PEP 703, 3.13 experimental, 3.14 supported but optional).
*Common weak answer:* "Python can't do concurrency." It can (threads for I/O, asyncio, processes); it can't run Python bytecode in parallel threads in the standard build.

**Q18. How do you diagnose a hung or slow concurrent service?**
A strong answer must mention:
- First: is CPU busy (spinning, livelock, CPU-bound) or idle (blocked: deadlock, pool exhaustion, slow dependency)?
- Take several thread dumps seconds apart; group by stack; separate stuck from busy; find lock holders.
- The JVM reports monitor/`j.u.c` lock deadlocks; non-lock resources (pools, DB rows, futures) need manual reasoning.
- Lock-contention profiles (JFR, async-profiler, Go mutex/block profiles) and pool metrics (active, waiting, wait time).
- Race detectors and stress tests for correctness bugs; they show presence, not absence.
*Common weak answer:* "Add more logging and restart." Logging changes timing, and a restart destroys the evidence that a thread dump would capture.

---

## Scenario Exercises

1. An in-memory rate limiter allows 100 requests per minute per API key. Under a load test of 500 concurrent requests for one key, it lets about 130 through in the first minute. The code reads the key's counter from a concurrent map, checks it against 100, and writes back the incremented value. Explain exactly how 130 get through and give two correct designs. What changes when the service runs on four instances?
2. A Java service has a background thread that refreshes a pricing table every 30 seconds by mutating a shared `HashMap` in place (clearing it and refilling it). Request threads read from it without locking. Most of the time it works, but a few times a day requests see missing prices, and once a request thread was found consuming 100% CPU in `HashMap.get`. Explain both symptoms and give the simplest correct design.
3. After moving a service from x86 instances to ARM instances, a worker sometimes processes a job whose `payload` field is `null`, although the producer always sets `payload` before adding the job's ID to a plain array that workers poll. It never happened on x86. Why, and how do you fix it?
4. A service handles each HTTP request on a pool of 50 threads. Each request calls a reporting function that runs on a separate 8-thread executor and blocks on `future.get()` for the result. The reporting function itself fans out three subqueries on the *same* 8-thread executor and blocks for their results. Under moderate load the reporting endpoint hangs completely and CPU drops to almost zero. Explain the mechanism and two fixes.
5. A Node.js API's p99 latency jumps from 30 ms to 900 ms whenever one customer uploads a large CSV, which the API parses with a synchronous library in the request handler. Other endpoints, unrelated to CSV, slow down too. Explain why, and propose two fixes. Would moving the service to a 16-core machine help?
6. A team rewrites a Python image-thumbnail service from one thread to 8 threads to use their 8-core machine. Throughput does not change. A second team rewrites a Python service that calls three slow HTTP APIs per request from one thread to 8 threads, and throughput rises about sevenfold. Explain both results, and what the first team should do.
7. A team migrates a Java service to virtual threads, creating one per request. Under a traffic spike, the number of concurrent requests rises from about 200 to 20,000, and the database starts rejecting connections and timing out; overall throughput is *lower* than before the migration. What happened, and what was missing?

## Diagnosis Lab

**Lab 1: The money-transfer service that freezes**
- *Symptom:* a payments service occasionally stops processing transfers. The health check (on a separate thread pool) still passes. Transfer requests time out at the load balancer after 30 seconds. CPU usage on the instance drops to about 2%. Restarting fixes it for a few hours or days.
- *Hypotheses:* (a) the database is slow or locked (Chapter 17); (b) the request pool is exhausted waiting for a dependency; (c) an in-process deadlock.
- *Evidence:* database metrics show no increase in query latency and no lock waits, so (a) is unlikely. Three `jcmd <pid> Thread.print` dumps taken 5 s apart show all 64 `transfer-worker` threads with identical stacks in all three, every one `BLOCKED` in `TransferService.transfer`, waiting to lock various `Account` monitors; at the end of the dump, "Found one Java-level deadlock" names `transfer-worker-12` (holds account 881, wants 1204) and `transfer-worker-40` (holds 1204, wants 881). The other 62 threads wait for one of those two accounts or for accounts held by threads waiting on them.
- *Root cause:* `transfer(from, to)` locks `from` then `to`. Two simultaneous transfers in opposite directions between the same pair of accounts (a refund issued while a payment was being processed) created a circular wait. Every later transfer touching those accounts queued behind them until the pool was exhausted.
- *Fix:* acquire the two account locks in a global order (lower account ID first). Better still, move the balance update into the database with a single transaction that updates the two rows in primary-key order, so that multi-instance deployments are also correct.
- *Prevention:* a code rule that any code taking more than one lock documents and follows the order; `tryLock` with a timeout on multi-lock paths so a missed case turns into a failed request with a clear log line rather than a frozen service; a scheduled check that calls the JVM's deadlock detection (`ThreadMXBean.findDeadlockedThreads()`) and alerts; an alert on "request pool fully busy for more than N seconds".

**Lab 2: The cache that made everything slower**
- *Symptom:* after adding an in-process cache for product data, a catalogue service's throughput plateaus at about 9,000 requests/second on a 16-core machine, with CPU usage around 35%. Before the cache, it handled 7,000 requests/second, with the database as the bottleneck. Adding more instances helps linearly; adding cores to one instance does not.
- *Hypotheses:* (a) GC pauses (Chapter 2); (b) lock contention in the cache; (c) the network or the load balancer.
- *Evidence:* GC logs show short, infrequent pauses: not (a). Thread dumps show 20–40 of the 64 request threads `BLOCKED` on the same monitor, `ProductCache.get`, at any moment. A JFR recording shows `jdk.JavaMonitorEnter` events with a total blocked time per minute larger than the total running time, all on `ProductCache`. Code: `get` is a `synchronized` method that also updates an LRU list (moving the entry to the front) on every read, and, on a miss, loads from the database *while holding the lock*.
- *Root cause:* a coarse-grained lock serialising every read, and held across a database call on misses. On average the lock is held about 100 µs per request (most hits are short, but misses hold it for milliseconds), capping throughput near 1 s / 100 µs ≈ 10,000 requests per second regardless of cores (Section 5.5).
- *Fix:* replace the hand-written cache with a concurrent cache library that uses a concurrent map and approximate, lock-amortised recency tracking; load misses with an atomic compute-if-absent per key, outside any global lock, so that only requests for the same missing key wait for each other. Throughput then becomes CPU-bound again.
- *Prevention:* "no I/O under a lock" in code review; lock-contention profiling in load tests; export wait-time metrics for locks and pools. Cache design itself is Chapter 21.

**Lab 3: The worker that would not stop**
- *Symptom:* during every deploy, one background-job service takes the full 30-second grace period to shut down and is then killed with `SIGKILL` (Chapter 4, Section 4.11). Jobs in progress at the time are sometimes processed twice after restart. In local testing, shutdown takes under a second.
- *Hypotheses:* (a) the worker is blocked on a long job or a slow I/O call that ignores shutdown; (b) the shutdown signal is not reaching the process (PID 1 problem, Chapter 4); (c) the worker never observes the shutdown flag.
- *Evidence:* logs show the `SIGTERM` handler running and setting `running = false`, so (b) is ruled out. A thread dump taken 10 s after `SIGTERM` shows the worker thread `RUNNABLE` in `JobWorker.loop`, with a stack in pure Java code (polling an in-memory queue), not in I/O; three dumps show it busy, so it is looping, not blocked: not (a). The field `running` is a plain `boolean`. Locally the process runs for seconds; in production for hours, long enough for the loop to be JIT-compiled.
- *Root cause:* a visibility bug (Section 5.3). The JIT hoisted the read of the non-volatile `running` out of the hot loop, so the worker never saw the write made by the signal-handling thread. In short local runs the loop was still interpreted and re-read the field.
- *Fix:* make `running` a `volatile` (or `AtomicBoolean`) field, so every iteration re-reads it and the write happens-before the read that sees it. Also make job processing idempotent, because a `SIGKILL` can always interrupt a job (the duplicates were a separate correctness bug; idempotent consumers are in Chapter 25).
- *Prevention:* treat every cross-thread flag as a synchronisation variable; static analysis rules that flag non-volatile fields written and read by different threads; a deploy test that measures shutdown time after a long warm-up.

---

## Solution Sketches

1. Many requests read the counter (say 87) before any of them writes back 88: a read-modify-write race across two separately atomic map operations, so increments are lost and the check passes too often (check-then-act). Correct designs: (a) one atomic compound operation per key, such as an atomic integer stored in the map and incremented with fetch-and-add, then checking the returned value; or the map's atomic `compute`/`merge`; (b) a per-key lock around read, check and increment. With four instances, each has its own memory, so each allows 100: the limit becomes 400. The counter must move to shared storage with an atomic operation (for example an atomic increment with expiry in Redis, Chapter 21; rate-limiting algorithms are in Chapter 24).
2. Readers run concurrently with `clear()` and refilling, so they can see an empty or half-filled map (missing prices); `HashMap` is not thread-safe, and a read concurrent with an internal resize can observe a broken structure (in older implementations, even a cycle that makes `get` loop forever; in current ones, lost entries or exceptions). There is also no happens-before edge, so readers may see partially constructed entries. Simplest fix: build a new map privately on each refresh and publish it through a `volatile` (or `AtomicReference`) field; readers take one reference per request (immutability + safe publication, Sections 5.3 and 5.10).
3. Message passing without synchronisation: the producer's two stores (`payload`, then the ID) can become visible out of order on ARM's weak memory model; x86's TSO preserves store→store and load→load order, so the hardware did not expose the bug there, although the compiler was always allowed to. Fix: publish through a synchronising structure (a concurrent queue, a lock, or an atomic with release/acquire semantics), so the payload write happens-before the worker's read.
4. Thread-starvation deadlock: up to 8 reporting tasks occupy all 8 executor threads, each blocked waiting for subqueries that are queued on the same executor with no free thread to run them. CPU is idle because everything waits. Fixes: run subqueries on a separate executor; compose futures with continuations (`thenCompose`/`allOf`) instead of blocking `get()` on pool threads; or run the subqueries directly on the calling thread. Also add timeouts so a hang becomes an error.
5. The synchronous parse runs on Node's single event-loop thread; while it runs (hundreds of milliseconds), no other callback can run, so every request on that instance waits (event-loop lag). More cores do not help, since one loop uses one core. Fixes: parse in a `worker_thread` or a separate service/queue; or use a streaming, incremental parser that yields to the loop between chunks; and limit the size of uploads processed inline.
6. Thumbnailing is CPU-bound pure Python (or a library that holds the GIL), so only one thread executes Python bytecode at a time: no speed-up. The HTTP service is I/O-bound; threads release the GIL while waiting on sockets, so the waits overlap. The first team should use multiple processes (`ProcessPoolExecutor` or several worker processes), or an imaging library that releases the GIL in its C code.
7. Virtual threads removed the thread pool, which had been (accidentally) bounding concurrency at about 200. Cheap threads let 20,000 requests reach the database at once, exceeding its connection capacity, so connection acquisition timed out and the database thrashed. Virtual threads reduce the cost of waiting; they add no database capacity. Missing: an explicit bound on concurrency to the database (a correctly sized connection pool with an acquisition timeout, or a semaphore), plus load shedding at the edge when the bound is reached (Chapters 11, 18 and 24).

---

**END OF CHAPTER 5**
