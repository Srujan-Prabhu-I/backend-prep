# Chapter 2: How a Program Runs: Hardware, Memory Hierarchy, and Language Runtimes

## Where We Are

Chapter 1 gave you the map: a web application is a set of processes on different machines, talking over networks. Before we can explain any of those processes (how an operating system schedules them, how a server handles thousands of requests, why a database cares about disks), we need to know what a running program physically *is*. That means the CPU executing instructions, memory and storage holding data at very different speeds, and the **language runtime** that sits between your source code and the hardware.

This chapter is the bottom layer of the book. Almost every performance discussion later ("why is this query slow?", "why does p99 latency spike every few minutes?", "why does this process use 3 GB when the heap is 2 GB?") ends in one of the mechanisms explained here.

**After this chapter you can:**
- explain what a CPU does when it runs a program, and what cores and hardware threads are;
- explain the memory hierarchy, quote order-of-magnitude latencies, and use them to estimate whether a design is fast or slow;
- explain cache lines, locality and false sharing;
- explain how HDDs and SSDs work and why "random vs sequential I/O" depends on the device;
- distinguish compilation, interpretation, bytecode VMs, JIT and AOT;
- draw a program's memory layout and explain the stack vs the heap;
- explain manual memory management, reference counting and tracing garbage collection, including why GC causes latency spikes;
- compare the JVM, CPython, Node.js and Go as execution environments.

**Interview questions this chapter prepares you for:**
1. What happens, at the hardware level, when a program runs?
2. What is the memory hierarchy, and why does it exist?
3. Why is accessing RAM slower than accessing CPU cache? Roughly how much slower, and why do latency numbers matter for design?
4. What is a cache line? What is false sharing?
5. HDD vs SSD: how do they work, and why is random I/O slow on an HDD?
6. Compiled vs interpreted languages? What is a JIT?
7. Stack vs heap: what lives where, and why?
8. What is a stack overflow? What is a memory leak?
9. How does garbage collection work? What is the generational hypothesis?
10. What is a stop-the-world pause, and how does it affect a web service?
11. Reference counting vs tracing GC?
12. Why does a process use more memory than its configured heap?
13. What is the GIL? Why is Node.js called single-threaded?

**Scope.** Here: hardware and the *program's* view of memory and execution. The operating system's view (processes, virtual memory, page tables, scheduling) is Chapter 3. Files, the page cache and fsync are Chapter 4. Threads, locks and concurrency are Chapter 5. This chapter mentions threads only where the hardware or runtime forces it.

---

## 2.1 The CPU: What "Running" Means

### Definition

A **CPU (central processing unit)** is the hardware component that executes a program's **instructions**: small, primitive operations such as "load a value from memory into a register", "add two registers", "compare", and "jump to another instruction if the comparison was true". A program, at the hardware level, is nothing more than a long sequence of such instructions stored in memory.

### Why it matters

Every line of code you write in any language eventually becomes these instructions. "My code is slow" always means one of two things: the CPU is executing too many instructions, or the CPU is **waiting**, usually for data from memory, disk or network. This chapter teaches you to tell the difference.

### Mechanism: the fetch–decode–execute cycle

A CPU core repeats one loop, billions of times per second:

```text
        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        ▼                                                          │
  ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┴─┐
  │  FETCH    │────►│  DECODE   │────►│  EXECUTE  │────►│ WRITE BACK │
  │ instr. at │     │ what op?  │     │ ALU / load│     │ result to  │
  │ address PC│     │ which regs│     │ / store / │     │ register or│
  │           │     │           │     │ branch    │     │ memory; PC │
  └───────────┘     └───────────┘     └───────────┘     │ advances   │
                                                        └────────────┘
```

- The **program counter (PC)** is a special register holding the address of the next instruction.
- **Registers** are a few dozen tiny storage slots *inside* the core, each holding one value (for example 64 bits). Arithmetic happens only on registers, so data in memory must be **loaded** into a register first and **stored** back afterwards.
- The **ALU (arithmetic logic unit)** performs the arithmetic and comparisons.

Consider a single line such as `total = total + price`. On a typical machine it becomes roughly:

```text
LOAD  r1, [address of total]     ; bring total from memory into register r1
LOAD  r2, [address of price]     ; bring price into r2
ADD   r1, r1, r2                 ; r1 = r1 + r2
STORE [address of total], r1     ; write the result back to memory
```

One line of source code becomes four instructions, and three of them touch memory. Keep that in mind; it is the root of Section 2.3 and of the race conditions in Chapter 5.

### Clock speed, pipelining and why "instructions per second" is not simple

The **clock** is a signal that ticks at a fixed frequency (e.g. 3 GHz = 3 billion ticks per second). Modern cores **pipeline** instructions: while one instruction executes, the next is being decoded and the one after is being fetched, like an assembly line. They also execute several independent instructions per tick, and even execute instructions **out of order** when an earlier one is waiting for data. They **predict branches** (guess which way an `if` will go) to keep the pipeline full; a wrong guess throws away partially done work.

The consequence: a core can complete several instructions per clock tick *when its data is ready*, and almost none while it waits for memory. **Waiting for data, not doing arithmetic, is usually what limits speed.**

### Cores and hardware threads

- A **core** is an independent execution unit with its own registers, pipeline and small caches. A "16-core" CPU can execute 16 instruction streams truly simultaneously. That is **parallelism** (Chapter 5).
- A software **thread** is one independent sequence of instructions inside a running program, with its own program counter, registers and stack. The operating system decides which thread runs on which core at each moment (Chapter 3 defines threads properly). A program with several threads can use several cores at once.
- **Simultaneous multithreading (SMT)**, marketed by Intel as Hyper-Threading, lets one core hold the state of two instruction streams (two sets of registers) and interleave them, so when one stream stalls waiting for memory, the other uses the execution units. The OS sees each as a separate logical CPU. Two hardware threads on one core do **not** give twice the throughput. They share the execution units and caches, and the gain depends on how often each stream stalls.

> **Backend connection.** When a cloud provider sells you "4 vCPUs", those are often hardware threads, not full cores. A CPU-heavy workload (JSON serialization, encryption, image processing) may get much less than 4× the throughput of one vCPU.

### Failure and diagnosis

- *Symptom:* a service is slow. The first question is whether the CPU is busy or waiting.
- *Evidence:* if per-core CPU utilisation (from `top` or your platform's CPU metrics) is near 100% while latency rises, the work is CPU-bound: too many instructions. If CPU is low while latency is high, the program is waiting for memory, disk, network or locks, and more CPU will not help. On Linux, `perf stat` reports instructions per cycle (IPC); a low IPC on a busy core means it is mostly stalled on memory (Section 2.3).
- *Fix:* for CPU-bound work, do less work (a better algorithm, less serialization) or add cores. For waiting, find what is being waited on (Chapters 4, 6 and 26).

### Guarantees and non-guarantees

- **Guaranteed:** a single core executes *your* instruction stream so that its results are as if instructions ran in program order.
- **Not guaranteed:** that *other* cores observe your memory writes in that order or immediately. Out-of-order execution and caches make this visible to other threads. This is why Chapter 5 needs a "memory model".

---

## 2.2 The Memory Hierarchy

### Definition

The **memory hierarchy** is the arrangement of storage technologies in a computer into levels. Each level going down is **larger, cheaper per byte and slower** than the one above: registers → CPU caches (L1, L2, L3) → main memory (RAM) → solid-state drive (SSD) → hard disk drive (HDD) → remote storage across a network.

**Main memory**, usually called **RAM** (random-access memory), is the computer's working memory. It holds the instructions and data of running programs; any byte can be read or written directly by its address in roughly the same time, which is what "random access" means; and it is **volatile**: its contents are lost when power is lost. Modern main memory is built from DRAM chips (described below). **Storage** (SSDs and HDDs) is the persistent level beneath it, read and written in blocks rather than bytes (Section 2.4).

### Why it exists

We want memory that is huge, cheap, fast and persistent, and no single technology is all four. Fast memory (the SRAM used for caches) is expensive per bit and takes more chip area, so you can only have megabytes of it. DRAM (main memory) is cheaper and denser but slower. Flash and magnetic disks are cheaper still and **persistent** (they survive power loss) but much slower. The hierarchy combines them: keep the data you are using *right now* in the small fast levels, and everything else in the big slow ones.

Why is DRAM slower than the SRAM in caches, and not just cheaper? An SRAM cell is a small circuit of transistors that holds its bit as long as power is on and can be read directly, and the caches sit on the same chip as the core. A DRAM cell is a single capacitor holding a tiny charge. Reading it means opening a whole row of cells and sensing and amplifying those charges, and because the charges leak, every row must be refreshed periodically. DRAM also sits on separate chips: a request leaves the processor, passes through the memory controller and across the memory bus, and the data travels back the same way. Each level is slower because of how it physically stores a bit and how far the data has to travel.

This only works because programs exhibit **locality** (Section 2.3): they tend to reuse the same data soon, and to use data located near data they just used.

### The numbers

The figures below are **orders of magnitude**, not exact values. They vary with hardware generation and workload, so treat them as estimation tools, not specifications.

```text
             ▲ faster, smaller, more expensive per byte
             │
   ┌─────────┴─────────┐   Registers          < 1 ns          bytes–KB (inside core)
   │     Registers     │
   ├───────────────────┤   L1 cache           ~1 ns           tens of KB per core
   │        L1         │
   ├───────────────────┤   L2 cache           ~3–5 ns         hundreds of KB–few MB per core
   │        L2         │
   ├───────────────────┤   L3 cache           ~10–20 ns       several–hundreds of MB, shared
   │        L3         │
   ├───────────────────┤   Main memory (DRAM) ~60–100 ns      GBs–TBs
   │        RAM        │
   ├───────────────────┤   NVMe SSD read      ~10–100 µs      TBs (persistent)
   │        SSD        │
   ├───────────────────┤   HDD random read    ~5–10 ms        TBs (persistent)
   │        HDD        │
   ├───────────────────┤   Network: same data centre RTT ~0.1–0.5 ms
   │      Network      │            cross-continent RTT   ~50–150 ms
   └─────────┬─────────┘
             │
             ▼ slower, larger, cheaper per byte
```

The numbers are hard to feel because humans have no intuition for nanoseconds. Scale them so that **1 ns becomes 1 second**:

| Operation | Real time (approx.) | If 1 ns were 1 second |
|---|---|---|
| L1 cache hit | 1 ns | 1 second |
| L3 cache hit | ~15 ns | ~15 seconds |
| RAM access | ~100 ns | ~1.5 minutes |
| NVMe SSD random read | ~20–100 µs | ~6 hours – 1 day |
| Round trip within a data centre | ~0.5 ms | ~6 days |
| HDD seek | ~5–10 ms | ~2–4 months |
| Round trip across a continent | ~100 ms | ~3 years |

### Using the numbers: a worked estimate

A request handler makes **10 sequential database queries**, each crossing the data-centre network (~0.5 ms round trip) and taking ~1 ms to execute:

```text
10 × (0.5 ms network + 1 ms query) = 15 ms
```

Meanwhile, a loop over 100,000 in-memory objects doing a few nanoseconds of work each takes ~0.5 ms. **The ten queries cost about 30× more than processing 100,000 objects in memory, and a third of that time is pure network waiting.** This is why backend performance work is mostly about removing round trips and I/O (Chapters 6, 16 and 18), and only occasionally about making loops faster.

### Failure and diagnosis: falling off a level

The hierarchy degrades gradually and then suddenly. While the data a program actively uses (its **working set**) fits in a level, almost every access hits that level. Once the working set outgrows it, a large share of accesses fall through to the next level down, which can be 10–1000× slower, so latency jumps rather than drifting.

- *Symptom:* a database that served a query in about a millisecond becomes much slower as a table grows, with no change in code or traffic.
- *Evidence:* the database's cache hit ratio drops, and disk reads per second rise (on Linux, `iostat -x` shows reads per second and average wait time per device).
- *Explanation:* the hot data used to fit in RAM and now does not, so reads go to the storage device. The database's buffer pool and its hit ratio are Chapter 15; the OS page cache is Chapter 4.

### Guarantees and non-guarantees

- The hierarchy is **managed automatically** for caches (by hardware) and partly for RAM (by the OS, Chapter 3). Your code reads an address, and the hardware finds the data in the fastest level holding it.
- **No guarantee** that your data *is* in a fast level. Performance depends on access patterns you control.
- Only the levels from SSD downward are **persistent**. Anything in registers, caches or RAM is lost on power failure, which is the whole reason databases write logs to disk (Chapter 15).

---

## 2.3 CPU Caches, Locality and False Sharing

### Definition

A **CPU cache** is a small, fast memory inside the processor that automatically holds copies of recently used main-memory data. Data moves between RAM and cache in fixed-size blocks called **cache lines**, commonly **64 bytes** on mainstream x86-64 and many ARM processors (some designs use 128).

### Why it exists

RAM is roughly 100× slower than L1 cache. Without caches, a core that can complete several instructions per nanosecond would spend almost all its time waiting for memory.

### Mechanism

When the core reads an address:

```text
 read address A
      │
      ▼
 Is the cache line containing A in L1? ── yes ──► HIT: value in ~1 ns
      │ no
      ▼
 In L2? ── yes ──► copy line into L1, return  (~4 ns)
      │ no
      ▼
 In L3? ── yes ──► copy line into L2/L1, return (~15 ns)
      │ no
      ▼
 MISS: fetch the whole 64-byte line from RAM (~100 ns),
       place it in the caches (evicting some older line), return
```

Because a whole line is fetched, reading `array[0]` also brings `array[1]` … `array[15]` (for 4-byte integers) into the cache **for free**.

**Locality** is the property that makes caches effective:
- **Temporal locality:** data used recently is likely to be used again soon (a loop counter, a hot configuration object).
- **Spatial locality:** data near recently used data is likely to be used soon (the next element of an array).

### Concrete: why data layout changes speed

```pseudo
// Array of 10 million integers stored contiguously
sum = 0
for i in 0 .. N-1:
    sum = sum + values[i]           // sequential: 1 miss per 16 elements

// Linked list of 10 million nodes scattered across the heap
sum = 0
node = head
while node != null:
    sum = sum + node.value          // each node may be in a different line:
    node = node.next                //   potentially 1 miss per element
```

Both loops do the same arithmetic. The array version mostly hits the cache (and hardware **prefetchers** detect the sequential pattern and fetch lines ahead of time). The linked-list version can miss on almost every node. On large data sets the difference is commonly several times, sometimes more. Big-O notation says both are O(N); the memory hierarchy says they are not equally fast.

### Cache coherence and false sharing

Each core has its own L1/L2 caches. If two cores cache the same line and one writes to it, the other's copy becomes stale. **Cache coherence protocols** (the MESI family is the textbook example) keep cores consistent: before a core writes to a line, it must gain exclusive ownership, which **invalidates** the copies in other cores' caches. The other cores must then re-fetch the line.

**False sharing** happens when two threads on different cores write to **different variables that happen to sit in the same cache line**. Logically they share nothing; physically, every write by one core invalidates the line in the other core, and the line ping-pongs between them.

```text
              one 64-byte cache line
   ┌──────────────────────────────────────────────────┐
   │ counterA (8 bytes) │ counterB (8 bytes) │ ...     │
   └──────────────────────────────────────────────────┘
        ▲                       ▲
        │ writes                │ writes
   ┌────┴─────┐            ┌────┴─────┐
   │  Core 0  │            │  Core 1  │
   │ thread 1 │            │ thread 2 │
   └──────────┘            └──────────┘

   t1: Core 0 writes counterA → takes exclusive ownership; Core 1's copy invalidated
   t2: Core 1 writes counterB → must re-fetch line, takes ownership; Core 0 invalidated
   t3: Core 0 writes counterA → re-fetch again ...   (repeat millions of times)
```

**Symptom:** a multi-threaded program that should scale linearly with cores gets *slower* as you add threads, even though threads touch "separate" data. **Fix:** pad or align hot, independently written variables so they occupy different cache lines, or have each thread accumulate locally and combine at the end.

> **Java lens.** The JDK marks some of its own hot fields with an internal `@Contended` annotation, which makes the JVM pad them onto separate cache lines. Application code can use it only with an extra JVM flag, so most code pads manually or uses per-thread accumulators.
>
> **Other runtimes.** C++ offers `alignas` together with the constant `std::hardware_destructive_interference_size` for the same purpose.

> **Backend connection.** You rarely tune cache lines in business code. But they explain why shared counters, metrics libraries and concurrent data structures (Chapter 5) are designed the way they are, and why "add more threads" can backfire.

### Guarantees and non-guarantees

- **Guaranteed:** caches are transparent. A program gets correct values whether or not the data is cached, and coherence ensures that all cores agree on the order of writes to any *single* memory location.
- **Not guaranteed:** that your data is in the cache when you need it; that variables you consider independent sit on different lines (false sharing); or that writes to *different* addresses become visible to other cores in the order your program made them. That last point is the memory model's job (Chapter 5).

### Failure and diagnosis

- *Symptom:* CPU at 100%, but throughput is lower than expected and adding cores doesn't help.
- *Evidence:* hardware performance counters (on Linux, `perf stat` reports cache misses and instructions per cycle). Low instructions-per-cycle with high cache misses means the CPU is mostly waiting on memory.
- *Fix:* improve locality (contiguous data, fewer pointer hops), and eliminate false sharing.

---

## 2.4 Storage Devices: HDDs and SSDs

Everything that must survive a power loss (database files, logs, uploaded files) ends up on a persistent storage device. The two dominant technologies behave very differently, and databases (Chapter 15) are designed around those differences.

### Hard disk drives (HDD)

**Definition.** An **HDD** stores data magnetically on spinning platters. A mechanical arm moves a read/write head across the platter surface.

**Mechanism.** Reading a block requires three steps:

```text
          ┌────────────────── platter (spinning, e.g. 7,200 RPM) ─────────┐
          │                         track                                  │
          │          ●  ← target sector moving toward the head             │
          │                                                                │
          └────────────────────────────────────────────────────────────────┘
                 ▲
     arm ────────┘  head
```

1. **Seek:** move the arm to the correct track (several milliseconds).
2. **Rotational latency:** wait for the target sector to rotate under the head. At 7,200 RPM one revolution takes ~8.3 ms, so the average wait is ~4 ms.
3. **Transfer:** read the bits as they pass, which is fast once positioned.

So a **random** read costs roughly 5–10 ms of mechanical movement, which limits a typical HDD to on the order of **100–200 random operations per second (IOPS)**. A **sequential** read (the next sectors on the same track) pays the seek once and then streams at full speed. On HDDs, **sequential I/O can be 100× or more faster than random I/O**. That gap is what log-structured designs and write-ahead logs were invented to exploit.

### Solid-state drives (SSD)

**Definition.** An **SSD** stores data in NAND flash memory chips with no moving parts. A controller with its own processor and firmware manages them.

**Mechanism and its quirks:**
- Flash is read and written in **pages** (commonly 4–16 KB), but can only be **erased** in much larger **blocks** (many pages; often megabytes).
- A page **cannot be overwritten in place**. It must be erased first, and only whole blocks can be erased.
- So the controller's **flash translation layer (FTL)** writes updated data to a fresh page, marks the old page invalid, and keeps a mapping from logical addresses to physical pages. Later, **garbage collection** inside the SSD copies the still-valid pages out of a block so the block can be erased.
- This internal copying causes **write amplification**: one logical write can cause several physical writes. Flash cells also wear out after a limited number of erase cycles, so the controller spreads writes evenly (**wear levelling**).

**Performance.** No seek, so a random read takes tens of microseconds (NVMe drives) and a fast SSD sustains hundreds of thousands of IOPS. **Random vs sequential still matters, but the gap is far smaller**, often single-digit multiples rather than 100×. Sustained heavy random writes can trigger internal garbage collection and cause latency spikes.

### Comparison

| Property | HDD | SSD (NVMe) |
|---|---|---|
| Random read latency | ~5–10 ms | ~10–100 µs |
| Random IOPS (order of magnitude) | 10²  | 10⁵–10⁶ |
| Sequential vs random gap | Very large (≥100×) | Small (single-digit ×) |
| Overwrite in place | Yes | No (erase-before-write, FTL remaps) |
| Wear | Mechanical failure | Limited erase cycles |
| Typical backend use today | Cheap bulk/archival storage | Databases, most servers |

### Durability caveat: the device lies a little

Both kinds of device have **volatile write caches** (DRAM on the drive). A drive may report a write as "complete" when it is only in that cache. Correct durability requires the OS to issue a flush command, and on SSDs a power-loss-protected cache or honest flush handling. How software forces data to stable storage (`fsync`) is Chapter 4; how databases build on it is Chapter 15.

### Failure and diagnosis

- *Symptom:* database or file operations slow down while CPU is low, often in bursts.
- *Evidence:* on Linux, `iostat -x 1` reports, per device and per second, reads and writes completed, the average wait per request and utilisation. A device near its IOPS limit, or with waits far above its normal latency, is saturated. An HDD under random access saturates at a few hundred operations per second; an SSD under sustained heavy writes can show periodic latency spikes from internal garbage collection.
- *Fix:* reduce random I/O (better indexes, Chapter 16; batching writes), move to faster storage, or buy more IOPS on cloud volumes. The OS side of I/O and its tools are Chapter 4.

### Guarantees and non-guarantees

- **Guaranteed:** data written *and flushed* to a healthy device survives power loss.
- **Not guaranteed:** that a write that "returned successfully" is on stable media (it may be in OS or device caches); uniform latency (SSD internal GC, HDD seeks); infinite endurance.

> **Cloud note.** Cloud "disks" are frequently **network-attached block storage**. Every I/O crosses a network, so latency is typically higher than a local NVMe drive, and IOPS are often a purchased quota. A database that is fast on a laptop SSD may hit its IOPS limit in the cloud (diagnosis in Chapter 4).

---

## 2.5 From Source Code to Execution

### Definitions

- A **compiler** translates source code, *before* the program runs, into another form, typically **machine code** (instructions for a specific CPU architecture) or **bytecode**.
- An **interpreter** executes a program by reading its code (source or bytecode) and performing each operation itself, *while* the program runs, without first producing machine code for the whole program.
- **Bytecode** is a compact, CPU-independent instruction set designed for a **virtual machine (VM)**: a program (such as the JVM) that executes bytecode by interpreting it and/or compiling it to machine code.
- A **just-in-time (JIT) compiler** compiles code to machine code **during execution**, usually only the parts that run frequently ("hot" code), using information observed at runtime.
- **Ahead-of-time (AOT) compilation** produces machine code before the program runs.

### Why multiple approaches exist

There is a three-way trade-off between **startup time**, **peak speed** and **portability/flexibility**:

- AOT-compiled native code (C, C++, Rust, Go) starts instantly and runs fast, but a binary is tied to one CPU architecture and OS, and the compiler can only optimise using what it knows before runtime.
- Interpreters start instantly and are flexible (dynamic typing, run anywhere the interpreter runs), but executing each operation through an interpreter loop costs many machine instructions per source operation.
- Bytecode + JIT (Java, C#, JavaScript engines) starts in the interpreter, then compiles hot code using **runtime profiling** (actual types seen, which branches are taken, which methods are really called). It can reach high peak speed, and can sometimes optimise better than AOT because it knows how the program actually behaves.

### Mechanism: how a JIT runtime executes your code

```text
 source ──compile──► bytecode ──load──► VM
                                          │
                       ┌──────────────────┴─────────────────┐
                       ▼                                    │
               INTERPRET bytecode                           │
               + count calls/loops (profiling)              │
                       │ method becomes "hot"               │
                       ▼                                    │
               JIT-COMPILE to machine code using            │
               observed types/branches (speculative         │
               optimisations, e.g. inlining)                │
                       │                                    │
                       ▼                                    │
               RUN NATIVE CODE ── assumption violated? ─────┘
                                  (e.g. new type appears)
                                  → DEOPTIMISE back to interpreter
```

- **Tiered compilation:** many VMs use a quick, lightly optimising compiler first and a slower, heavily optimising one for the hottest code. The HotSpot JVM has C1 and C2 tiers; V8 interprets with Ignition and optimises hot code with TurboFan, with intermediate tiers in recent versions.
- **Speculation and deoptimisation:** if the JIT assumed "this variable is always an integer" and a string arrives, the compiled code is discarded and execution falls back to the interpreter until it recompiles.

### Comparison

| Approach | Examples | Startup | Peak speed | Notes |
|---|---|---|---|---|
| AOT to native | C, C++, Rust, Go | Fast | High | Binary per platform |
| Interpreter | CPython (default), Ruby (classic) | Fast | Lower | Simple, flexible |
| Bytecode VM + JIT | JVM (Java, Kotlin), .NET, V8 (JavaScript) | Slower (warm-up) | High after warm-up | Profile-guided optimisation |
| AOT for VM languages | GraalVM Native Image, .NET Native AOT | Fast | Good, often below peak JIT | Gives up some runtime flexibility |

"Compiled vs interpreted" describes an **implementation**, not a language. Python has interpreters (CPython) and JIT implementations (PyPy). Java is normally JIT-compiled but can be AOT-compiled.

### Backend connection: warm-up

A JIT-based service is slowest right after it starts: code is being interpreted and compiled, and caches are cold. If a freshly deployed instance immediately receives full traffic, its first requests can be much slower than steady state, which shows up as a latency bump after every deploy. Mitigations include sending synthetic **warm-up** traffic before marking an instance ready (readiness checks, Chapter 24), ramping traffic gradually (deployment strategies, Chapter 26), or using AOT/startup snapshots for fast-scaling services. The evidence is latency broken down per instance: only new instances are slow, and only for their first minutes. JIT runtimes can also log their compilation activity, which is heavy during exactly that window.

### Guarantees and non-guarantees

- **Guaranteed:** the program's observable behaviour is the same whichever execution strategy runs it (a JIT must preserve semantics).
- **Not guaranteed:** consistent performance over time. JIT compilation, deoptimisation and warm-up change the speed of the *same* code as the process runs, which matters when you benchmark (always measure after warm-up).

---

## 2.6 A Program's Memory: Stack and Heap

### Definition

When a program runs, its memory is organised into regions with different purposes and lifetimes. The two your code interacts with most are:

- The **stack**: a region holding one **stack frame** per active function call. A frame contains that call's local variables, arguments, and the return address (where to continue when the function returns). Frames are pushed on call and popped on return, strictly last-in-first-out.
- The **heap**: a region for data whose size or lifetime is not tied to a single function call. It is allocated on request (`new`, `malloc`, creating an object or list) and released later, either explicitly or by a garbage collector.

### The full layout

Here is a typical layout of one process's address space (exact arrangement varies by OS and architecture; the OS side is covered in Chapter 3):

```text
 high addresses
 ┌──────────────────────────────┐
 │           STACK              │  one frame per active call; grows ↓ (typically)
 │  main() frame                │
 │  handleRequest() frame       │
 │  computeTotal() frame        │
 ├──────────────── ↓ ───────────┤
 │                              │
 │   (unused address space)     │
 │                              │
 ├──────────────────────────────┤
 │  MEMORY-MAPPED REGION        │  shared libraries, mapped files, large allocations
 ├──────────────── ↑ ───────────┤
 │           HEAP               │  dynamically allocated objects; grows ↑
 ├──────────────────────────────┤
 │  BSS  (zero-initialised      │  global/static variables without initial values
 │        globals)              │
 ├──────────────────────────────┤
 │  DATA (initialised globals)  │  global/static variables with initial values
 ├──────────────────────────────┤
 │  TEXT (machine code)         │  the program's instructions, read-only
 └──────────────────────────────┘
 low addresses
```

In multi-threaded programs, **each thread has its own stack**, while the heap, globals and code are shared by all threads in the process. That is exactly why heap data needs synchronisation and stack locals don't (Chapter 5).

### Concrete: stack frames during recursion

```pseudo
function factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

result = factorial(3)
```

```text
 call factorial(3)    call factorial(2)    call factorial(1)    returns unwind
 ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
 │              │     │              │     │ fact n=1     │     │              │
 │              │     │ fact n=2     │     │ fact n=2     │     │              │
 │ fact n=3     │     │ fact n=3     │     │ fact n=3     │     │              │
 │ main         │     │ main         │     │ main         │     │ main result=6│
 └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                             returns 1 → n=2 frame computes 2*1=2
                                             → returns 2 → n=3 frame computes 3*2=6
```

Each frame holds its own `n`. Allocation is just moving a **stack pointer** register: extremely cheap, and memory is reclaimed automatically on return.

### Stack overflow

Each thread's stack has a fixed maximum size set by the OS or runtime (commonly on the order of 1–8 MB, configurable). Unbounded or very deep recursion keeps pushing frames until the limit is hit. The result is a **stack overflow** (in Java a `StackOverflowError`, in Python a `RecursionError` raised by its own recursion-depth limit, in C a crash).

*Typical backend causes:* recursive processing of deeply nested input (a maliciously nested JSON document), cyclic object graphs serialised recursively (entity A references B which references A), and accidental infinite recursion. *Defences:* depth limits on parsers, iterative algorithms, and breaking cycles before serialisation (Chapter 12). *Evidence:* the error's stack trace shows the same few frames repeating hundreds or thousands of times, which identifies the recursive path.

### The heap

Heap allocation is more expensive than stack allocation. The allocator must find a free block of the right size and track it, and something must eventually free it. Many runtimes make the common case fast: managed runtimes give each thread a private chunk (a **thread-local allocation buffer**) and allocate by bumping a pointer. The cost moves to **reclamation**, which is where garbage collection (Section 2.8) comes in.

Heap memory can become **fragmented**: there may be enough free bytes in total but no single contiguous block large enough for a request. Some garbage collectors fix this by **compacting** (moving live objects together).

### Stack vs heap at a glance

| | Stack | Heap |
|---|---|---|
| Holds | Local variables, arguments, return addresses | Objects, arrays and data outliving a call or of dynamic size |
| Allocation | Move a pointer (very cheap) | Allocator call or pointer bump; reclamation costs |
| Lifetime | Ends when the function returns | Until freed or garbage-collected |
| Owner | One per thread (private) | Shared by all threads in the process |
| Size | Small, fixed limit per thread | Large, bounded by configuration/OS |
| Typical failure | Stack overflow | Leaks, fragmentation, out-of-memory, GC pauses |

**Common misconception:** "primitives go on the stack, objects on the heap" is a simplification of how some languages *usually* behave. A local variable *holding a reference* lives on the stack while the object it points to lives on the heap. Compilers may also place objects on the stack when they can prove the object never escapes the function (**escape analysis**).

---

## 2.7 Memory Management Models

### Definition

**Memory management** is the policy and mechanism by which a program obtains heap memory and returns it when the data is no longer needed. Data is **live** while the program can still reach and use it, and becomes **garbage** once it is unreachable. The core problem is deciding, safely and efficiently, *when* memory can be reused.

### Why it is hard

Free too early and the program uses memory that now holds something else (**corruption**, a security hole). Free too late or never and memory usage grows until the process dies (**leak**). In a large program with shared data structures, no single function knows when the last user of an object is done.

There are three main strategies.

### 1. Manual management

The programmer calls an allocator (`malloc`) and must explicitly release (`free`) each allocation exactly once.

```pseudo
buffer = malloc(1024)
use(buffer)
free(buffer)
use(buffer)        // BUG: use-after-free (reads/writes memory now owned by someone else)
free(buffer)       // BUG: double free (can corrupt the allocator's own data structures)
```

- **Guarantees:** full control and predictable timing (no collector pauses).
- **Failure modes:** leaks (forgetting `free`), use-after-free, double free and buffer overflows. These have historically been a dominant source of serious security vulnerabilities in C/C++ software.
- **Ownership systems:** Rust moves the checking to compile time. Every value has one owner and is freed automatically when the owner goes out of scope, and the compiler rejects programs that could use memory after it is freed. You get no GC and no manual `free`, at the cost of stricter rules for the programmer.

### 2. Reference counting

Each object stores a **count of references pointing to it**. Creating a reference increments the count; dropping one decrements it. When the count reaches zero, the object is freed immediately (and the counts of objects *it* references are decremented, possibly cascading).

```text
 a = new Order()      Order.refcount = 1
 b = a                Order.refcount = 2
 a = null             Order.refcount = 1
 b = null             Order.refcount = 0 → freed immediately
```

- **Strengths:** memory is reclaimed promptly and incrementally, with no big pauses in the simple case.
- **Weaknesses:**
  - **Cycles leak.** If A references B and B references A, and nothing else references either, both counts stay at 1 forever.
    ```text
      (no outside refs)
      ┌───┐ ──► ┌───┐
      │ A │     │ B │      A.count = 1, B.count = 1 → never freed
      └───┘ ◄── └───┘
    ```
  - Every reference assignment costs a count update. In multi-threaded programs those updates must be **atomic** (performed as one indivisible step, so two cores updating the same count cannot lose an update), which is expensive (Chapter 5).
  - Freeing the root of a huge structure can cascade into a long pause.
- **Where it's used:** CPython (with an additional cycle detector, Section 2.9), Swift/Objective-C (ARC), and C++ `shared_ptr`.

### 3. Tracing garbage collection

A **tracing garbage collector** periodically determines which objects are reachable by following references from a set of **roots**, and reclaims everything else. Cycles are not a problem: an unreachable cycle is simply never visited. The JVM, .NET, Go and JavaScript engines use tracing GC. Section 2.8 explains how it works.

### A memory leak in a garbage-collected language

GC frees *unreachable* objects. It cannot free objects that are reachable but that you will never use again. So leaks still happen in Java, Go, Python and JavaScript, as **unintended reachability**:

```pseudo
// A "cache" that is never bounded or evicted
global requestCache = new Map()

function handle(request):
    requestCache.put(request.id, request)   // every request stays reachable forever
    ...
```

Common real-world causes: unbounded in-process caches or maps keyed by user/request IDs; listeners or callbacks registered and never removed; per-thread storage on pooled threads that is never cleared (Chapter 11); collections growing inside long-lived singletons. The symptom is memory that climbs steadily over hours or days, increasingly frequent and longer GCs, then an out-of-memory crash.

---

## 2.8 How Tracing Garbage Collection Works

### Definition

**Garbage collection (GC)** is automatic memory management in which the runtime reclaims heap objects that the program can no longer reach. A **tracing** collector finds live objects by starting from the **roots** and following references transitively. Everything not found is garbage.

**Roots** are the references the program can use directly without following any other reference: local variables and arguments in every thread's stack frames, global/static variables, CPU registers, and some runtime-internal references.

### Mechanism 1: mark and sweep

```text
 ROOTS:  [stack var x]   [stack var y]   [global config]
            │               │                 │
            ▼               ▼                 ▼
          ┌───┐           ┌───┐             ┌───┐
          │ A │──────────►│ B │             │ E │
          └───┘           └───┘             └───┘
                            │
                            ▼
                          ┌───┐       ┌───┐ ◄──┐
                          │ C │       │ F │    │      F and G reference each other
                          └───┘       └───┘ ───► G    but nothing reachable points to them
                                      ┌───┐
                                      │ D │           D: nothing points to it
                                      └───┘
 MARK phase:  start at roots → mark A, B, E → follow A→B (already marked), B→C → mark C
              Marked (live): A, B, C, E
 SWEEP phase: scan the heap; unmarked objects D, F, G → their memory is freed
```

- **Mark:** traverse the object graph from the roots, setting a "marked" bit on every object reached.
- **Sweep:** walk the heap. Unmarked memory is added to a free list for reuse; marks are cleared for the next cycle.
- **Compact (optional):** slide the live objects together to remove fragmentation, then update every reference to the moved objects.
- **Copying collection** (a variant): copy live objects from one region into an empty region; the old region is now entirely free. Its cost is proportional to the *live* objects, not to the garbage.

### Mechanism 2: the generational hypothesis

**The generational hypothesis** is an empirical observation: *most objects die young.* In a web request, the request object, parsed JSON, temporary lists and DTOs all become garbage within milliseconds, while a few objects (caches, connection pools, configuration) live for the process's whole lifetime.

**Generational collectors** exploit this by dividing the heap:

```text
 ┌────────────────── YOUNG GENERATION ──────────────────┐ ┌──── OLD GENERATION ────┐
 │  ┌──────────────┐  ┌────────────┐  ┌────────────┐    │ │                        │
 │  │    EDEN      │  │ SURVIVOR 0 │  │ SURVIVOR 1 │    │ │  long-lived objects    │
 │  │ new objects  │  │            │  │            │    │ │  (caches, pools, ...)  │
 │  └──────────────┘  └────────────┘  └────────────┘    │ │                        │
 └──────────────────────────────────────────────────────┘ └────────────────────────┘
   minor GC: frequent, fast                                major/full GC: rarer, costlier
```

1. New objects are allocated in the young generation (cheaply, by bumping a pointer).
2. When it fills, a **minor (young) collection** runs. Because most young objects are already dead, it only needs to copy the few survivors, which is fast.
3. Objects that survive several minor collections are **promoted** to the old generation.
4. The old generation is collected less often, by a **major (old/full) collection** that is more expensive because it holds many live objects.

How does a minor GC avoid scanning the whole old generation for references *into* the young generation? The runtime inserts a **write barrier**, a few extra instructions on every reference write that record ("card-mark") old-generation regions that were modified to point at young objects. A minor GC then scans only those recorded regions as extra roots.

### Mechanism 3: stop-the-world vs concurrent

While the collector traces the graph, the program (the **mutator**) is changing it. The simplest fix is to pause all application threads: a **stop-the-world (STW) pause**.

Modern collectors do most of their work **concurrently**, while application threads run, and keep only short STW phases (for example, scanning roots). This needs extra machinery (barriers that let the collector notice references changed by the running program) and costs some throughput and extra memory.

The **GC trade-off triangle** is a common way to frame collector design: improving one of these three usually costs one of the others, so no collector is best at all three:
- **Throughput:** the share of CPU time spent running your code rather than collecting.
- **Latency:** short, predictable pauses.
- **Memory footprint:** how much extra heap headroom it needs.

| Collector family (examples) | Optimises | Pauses |
|---|---|---|
| Parallel/throughput (JVM Parallel GC) | Throughput | Long STW pauses, using all cores |
| Region-based, pause-target (JVM G1; HotSpot's default on server-class machines in recent versions, at the time of writing) | Balance; aims for a pause-time goal | Usually short; can degrade under pressure |
| Concurrent low-latency (JVM ZGC, Shenandoah; Go's GC) | Latency | Very short pauses; more CPU/memory overhead |

### Concrete: how a GC pause becomes a p99 spike

```text
 time ──────────────────────────────────────────────────────────────────►
 requests:  r1 r2 r3 r4  r5 r6 r7 r8 r9 ...
 app:       ██████████████░░░░░░░░░░░░░░░░██████████████████████
                          │ STOP-THE-WORLD │
                          │   pause 300 ms │
 r5 arrives ─────────────►│   waits ...    │──► processed, latency ≈ 300 ms + normal 5 ms

 Normal latency 5 ms. A request that arrives during (or is in flight across) the
 pause waits for the rest of it: up to 300 ms for one arriving at the start (r5),
 almost nothing for one arriving just before the end.
```

Put numbers on it. Assume a steady stream of requests and a 300 ms pause every 15 seconds. The process is paused 1.2 s out of every 60 s, so about 2% of requests arrive during a pause, and each waits for the remainder: between 0 and 300 ms, 150 ms on average.

- **Average latency:** 5 ms + 2% × 150 ms ≈ 8 ms. On a dashboard this looks like a small bump.
- **p99 latency** (the latency that 99% of requests stay under): the slowest 1% of requests are the half of the paused requests that waited more than 150 ms, so p99 ≈ 155 ms, about thirty times normal.
- **Maximum:** about 305 ms.

The average hides the problem; the tail exposes it. This is the classic "averages lie" lesson (Chapter 26). Worse, if a load balancer's health check or a client's timeout is shorter than the pause, the instance can be marked unhealthy or requests can be retried elsewhere, amplifying load (Chapter 24).

### Failure modes and diagnosis

| Symptom | Likely cause | Evidence |
|---|---|---|
| Periodic latency spikes on an otherwise idle-looking service | Long STW pauses | GC logs: pause durations aligned with spike timestamps |
| CPU high, throughput low, memory near max | **GC thrashing**: the heap is nearly full of live data and the collector runs back-to-back while reclaiming little | GC logs show frequent collections recovering little; time spent in GC is high |
| Memory grows steadily over days, then OutOfMemory | Leak (unintended reachability) | Heap dump: find the dominating collections and who references them |
| OutOfMemory right after a large request | A single huge allocation (loading a whole file or result set into memory) | Allocation profile; code path loading unbounded data |

**Mitigations:** reduce allocation rate (fewer temporary objects on hot paths); size the heap with headroom; choose a collector matching your latency goals; stream large data instead of loading it whole (Chapters 7 and 18); bound caches (Chapter 21).

### Guarantees and non-guarantees

- **Guaranteed:** unreachable objects are eventually reclaimed; you cannot use-after-free or double-free.
- **Not guaranteed:** *when* memory is reclaimed; pause-free execution; that memory is freed if still reachable (leaks remain possible); timely cleanup of non-memory resources (open files, sockets, DB connections). GC is **not** a resource manager: close those explicitly (e.g. `try-with-resources`, `with`, `defer`), or you exhaust them (Chapter 4).

---

## 2.9 Language Runtimes as Execution Environments

A **language runtime** is the software layer that executes a program and provides services the language promises: memory management, threads or tasks, standard I/O, and sometimes a JIT. For a backend engineer, the runtime determines how concurrency, memory and latency behave, which matters more than syntax.

| | JVM (Java, Kotlin) | CPython (Python) | Node.js (V8, JavaScript) | Go |
|---|---|---|---|---|
| Execution | Bytecode; interpreter + tiered JIT | Bytecode interpreter | Interpreter + tiered JIT | AOT-compiled native binary |
| Memory management | Tracing, generational, several collectors | Reference counting + cycle collector | Tracing, generational | Tracing, concurrent, non-generational, non-moving |
| Parallel CPU execution of user code | Yes: many OS threads | Limited by the **GIL** in the default build | One JS thread per event loop; parallelism via worker threads/processes | Yes: goroutines on all cores |
| Concurrency model | OS threads; virtual threads (Java 21+) | Threads (GIL), `asyncio` event loop, multiple processes | Single-threaded **event loop** with non-blocking I/O | Goroutines (M:N scheduled by the runtime) |
| Typical server scaling | Thread pool or virtual threads in one process | Several worker processes per machine | One process per core (cluster) or more instances | One process using all cores |

### The GIL (CPython)

The **Global Interpreter Lock (GIL)** is a **mutex** (a lock that only one thread can hold at a time; Chapter 5) in CPython's default build that allows only one thread at a time to execute Python bytecode within a process. It exists largely because reference counting (Section 2.7) would otherwise need an atomic update on every reference change, and because it simplifies the interpreter and C extensions.

- **Consequence:** CPU-bound Python code does **not** get faster with more threads in one process. Threads still help with I/O-bound work, because a thread releases the GIL while it waits on network or disk.
- **Backend practice:** Python web servers usually run **multiple worker processes** (e.g. one per core), each with its own interpreter and GIL. Recent CPython versions also offer an optional **free-threaded build** without the GIL; the default build still has it.

### "Node.js is single-threaded"

Node runs your JavaScript on **one thread** with an **event loop**: a loop that waits for I/O events and runs the callback registered for each. I/O is non-blocking, so while one request waits for the database, the thread handles other requests. The runtime does use additional internal threads (e.g. a pool for file system work and DNS lookups).

- **Strength:** very cheap concurrency for I/O-bound work.
- **Weakness:** any CPU-heavy work (a big JSON parse, a synchronous crypto call, a tight loop) **blocks the loop**, and *every* request on that process waits.

The event loop's mechanism is in Chapter 5; its OS foundation (epoll) is in Chapter 4.

### The JVM

The JVM runs a service as one process that uses all cores: each Java thread has classically been an OS thread, so CPU-bound work runs in parallel within one process. The heap is a large, explicitly sized region managed by a generational tracing collector chosen at startup (Section 2.8), and code is interpreted first and JIT-compiled once hot (Section 2.5). Its characteristic production concerns follow from that design: warm-up after every start, GC pauses in the latency tail, and process memory well beyond the heap (Section 2.10). Virtual threads (Java 21+) make *waiting* cheap, but they add no CPU (Chapter 5).

### Go

Go compiles ahead of time to a single native binary, so it starts quickly with no JIT warm-up. Its runtime multiplexes many lightweight **goroutines** onto a small number of OS threads (by default, about as many as there are CPUs available to the process), so one process uses all cores. A goroutine starts with a stack of a few kilobytes that the runtime grows as needed, which is why a Go server can hold hundreds of thousands of them where one OS thread per connection would not fit in memory. The collector is concurrent and non-generational with short pauses, and it is tuned mainly by a target for heap growth between collections and, optionally, a soft memory limit. Goroutine scheduling is in Chapter 5.

### Why this matters

These are not trivia. They determine:
- how many requests one process can serve concurrently (Chapter 11);
- whether a CPU-heavy endpoint hurts other endpoints;
- how to scale on one machine (threads vs processes);
- what latency spikes look like (GC pauses vs event-loop blocking).

---

## 2.10 Backend Connection: Memory and Runtimes in Production

### A process's memory is more than its heap

You configure a heap size, say 2 GB, and then the operating system reports that the process uses 2.8 GB. Nothing is wrong. A managed runtime's memory includes much more than the heap:

```text
 ┌──────────────── process memory as the OS sees it (≈ RSS, Chapter 3) ───────────────┐
 │ ┌──────────────────────── heap (configured max) ───────────────────────┐            │
 │ │ young gen │ old gen │ GC headroom                                    │            │
 │ └───────────────────────────────────────────────────────────────────────┘            │
 │ thread stacks (N threads × stack size)   │ JIT code cache │ class/metadata areas     │
 │ GC bookkeeping structures                │ native/off-heap buffers (network, I/O)    │
 │ the runtime's own code & shared libraries│ allocator overhead / fragmentation        │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

So **total process memory = heap + everything else**. Consequences:
- If you run in a **container** (a group of processes that the kernel isolates and limits as a unit, including a memory limit; Chapter 4) with a 2 GB memory limit and set the heap to 2 GB, the process will exceed the limit and the **kernel will kill it** (OOM kill). This happens without a heap out-of-memory error, because the runtime never ran out of *heap* (mechanism in Chapters 3 and 4).
- Rule of thumb: leave substantial room (often 25% or more) between heap max and the container or machine limit, and measure actual usage under load. Many runtimes can size the heap as a fraction of the container limit.
- Thousands of threads cost memory through their stacks. This is one reason thread-per-request designs have limits (Chapter 11).

> **Java lens.** The JVM detects the container's memory limit and can size the heap as a percentage of it with `-XX:MaxRAMPercentage`, instead of a fixed `-Xmx`.

### Choosing heap size

- **Too small:** GC runs constantly (thrashing), CPU is wasted, and there is a risk of out-of-memory.
- **Too large:** collections are rarer but each may scan more (depending on the collector), the process uses memory other processes could use, and in containers it risks the OOM kill above.
- **Method:** measure the **live set** (heap usage right after a full collection) under realistic load, then give it headroom for allocation bursts and collector needs. Watch GC logs or metrics, not just "memory used", because a GC heap is *supposed* to fill up between collections.

### Warm-up and deploys

JIT runtimes need warm-up (Section 2.5), and every new instance also starts with cold caches and empty connection pools (Chapter 18). Schedule deploys and autoscaling with this in mind (Chapter 26).

### What to observe

- **GC metrics:** pause durations (max, p99), frequency, time spent in GC, heap used after GC.
- **Process memory (RSS)** versus the limit.
- **CPU:** user time vs time spent in GC threads.
- **Event-loop lag** (Node) or thread pool saturation (Chapter 11) as signs of CPU-blocked request handling.

---

## Chapter Summary

A program at the hardware level is a stream of simple instructions that a CPU core fetches, decodes and executes, operating on a few **registers**. Cores are fast; **waiting for data is what makes programs slow**. Storage is organised as a **memory hierarchy** (registers, L1/L2/L3 caches, RAM, SSD, HDD, network), with each level larger, cheaper and slower by roughly an order of magnitude or more. It works because of **locality**. Caches move data in **cache lines**, so contiguous access is fast and pointer-chasing is slow, and two cores writing to the same line cause **false sharing**. Only storage devices are persistent. HDDs pay mechanical seek and rotation costs (random I/O ≫ sequential), while SSDs have no seeks but erase-before-write internals (smaller random/sequential gap, write amplification).

Source code reaches the CPU via **AOT compilation**, **interpretation** or **bytecode + JIT**. JIT runtimes reach high speed after **warm-up** using runtime profiling. A running program's memory has code, globals, a per-thread **stack** of call frames (cheap, automatic, limited in size) and a shared **heap** for dynamic data. Heap memory is managed manually, by **reference counting** (prompt, but leaks cycles) or by **tracing GC** (marks from roots, handles cycles). Generational collectors exploit "most objects die young". Collectors trade throughput, latency and memory, and **stop-the-world pauses** show up as tail-latency spikes. GC prevents use-after-free but not leaks through unintended reachability, and it does not close files or sockets.

Runtimes differ in ways that shape backend design: the JVM and Go use all cores from one process; CPython's **GIL** pushes servers to multiple processes; Node's **event loop** handles I/O concurrency on one thread but is blocked by CPU work. A process uses far more memory than its heap, which matters most inside containers.

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| CPU core execution | Your thread sees its own operations in program order | Other cores see your writes immediately or in order (Chapter 5) |
| CPU caches | Automatic, transparent copies of memory | That your data is cached; freedom from false sharing |
| RAM | Fast random access | Persistence across power loss |
| SSD/HDD | Persistence once data is flushed to stable media | That a completed `write` is already on media; uniform latency |
| JIT runtime | Same program semantics | Constant performance over time (warm-up, deoptimisation) |
| Stack | Automatic, cheap reclamation on return | Unlimited depth |
| Reference counting | Prompt reclamation of unreferenced objects | Reclamation of cycles (without a cycle collector) |
| Tracing GC | Unreachable memory is eventually reclaimed; no use-after-free | Pause-free execution; reclamation of reachable-but-unused data; closing non-memory resources |

## Interview Drill

**Q1. What happens at the hardware level when a program runs?**
A strong answer must mention:
- Code is compiled or interpreted into machine instructions stored in memory.
- A core runs the fetch–decode–execute cycle driven by the program counter.
- Arithmetic happens on registers; loads and stores move data between memory and registers.
- Pipelining, out-of-order execution and branch prediction keep the core busy, but memory stalls dominate.
*Common weak answer:* "The CPU runs the code line by line." Lines become several instructions, and execution is overlapped and reordered.

**Q2. What is the memory hierarchy, and why does it exist?**
A strong answer must mention:
- The levels (registers → L1/L2/L3 → RAM → SSD → HDD → network) with their order-of-magnitude latencies.
- No single technology is fast, large, cheap and persistent at once.
- Locality (temporal and spatial) makes the small fast levels effective.
- Persistence starts only at the storage devices.
*Common weak answer:* listing levels without saying *why* the hierarchy works (locality).

**Q3. Why is RAM slower than CPU cache, roughly how much slower, and why do latency numbers matter for design?**
A strong answer must mention:
- Roughly ~1 ns for L1 vs ~100 ns for RAM: two orders of magnitude.
- The physical reason: caches are SRAM on the processor chip; RAM is DRAM on separate chips, where a read opens and senses a row of capacitors and travels through the memory controller and bus.
- Cache misses stall the core, so data layout changes speed even at equal Big-O.
- Order-of-magnitude numbers let you estimate a design before building it. Network and disk round trips (µs–ms) dwarf in-memory work, so removing round trips matters most (ten sequential queries cost more than processing 100,000 objects).
*Common weak answer:* exact numbers stated as universal constants, or numbers recited with no use. Their value is in estimating which part of a design dominates.

**Q4. What is a cache line, and what is false sharing?**
A strong answer must mention:
- The unit of transfer between RAM and cache, commonly 64 bytes.
- Coherence protocols invalidate other cores' copies when one core writes.
- False sharing: independent variables written by different cores share a line, causing it to ping-pong.
- The symptom (worse scaling with more threads) and the fix (padding/alignment, per-thread accumulation).
*Common weak answer:* confusing false sharing with a data race. No data is logically shared.

**Q5. HDD vs SSD: how does each work, and how does that affect random vs sequential I/O?**
A strong answer must mention:
- HDD: seek + rotational latency, milliseconds per random I/O, sequential vastly faster.
- SSD: no moving parts, page reads/writes, block erases, FTL remapping, write amplification, wear levelling.
- SSD random I/O is far faster than HDD, and the random/sequential gap is much smaller.
- Databases' sequential log writes originate from HDD constraints and still help on SSDs.
*Common weak answer:* "Sequential is 100× faster than random" with no mention of the device.

**Q6. Compiled vs interpreted vs JIT?**
A strong answer must mention:
- These describe implementations, not languages.
- AOT: machine code before running; fast startup.
- Interpreter: executes operations one by one; flexible, slower.
- JIT: compiles hot code at runtime using profiles; high peak speed after warm-up; can deoptimise.
*Common weak answer:* "Java is interpreted, C is compiled." Java bytecode is mostly JIT-compiled to native code.

**Q7. Stack vs heap?**
A strong answer must mention:
- Stack: per-thread, LIFO frames of locals, arguments and return addresses; pointer-bump allocation; freed on return; size-limited.
- Heap: shared dynamic memory for data that outlives calls; freed manually or by GC.
- Locals holding references live on the stack while their objects live on the heap; escape analysis can change placement.
- Per-thread stacks are why locals are thread-safe and heap objects need synchronisation.
*Common weak answer:* "Primitives on the stack, objects on the heap" as an absolute rule.

**Q8. What is a stack overflow? Give a backend example.**
A strong answer must mention:
- The per-thread stack size limit is exceeded by too many nested frames.
- Typical cause: unbounded or deep recursion.
- Backend examples: deeply nested JSON parsed recursively; cyclic object graphs serialised recursively.
- Defences: depth limits, iterative algorithms, breaking cycles.
*Common weak answer:* confusing it with running out of heap.

**Q9. How does a tracing garbage collector work?**
A strong answer must mention:
- Roots (stack variables, globals, registers).
- The mark phase traverses reachable objects; sweep frees the unmarked; compaction or copying removes fragmentation.
- Cycles are collected naturally.
- Concurrent collectors need barriers; STW pauses still exist.
*Common weak answer:* "It deletes objects that aren't used." It deletes objects that are *unreachable*, which is different.

**Q10. What is the generational hypothesis, and how do collectors exploit it?**
A strong answer must mention:
- Most objects die young.
- Young generation collected often and cheaply (copy only survivors); survivors promoted to old.
- Write barriers / card marking track old→young references so minor GCs don't scan the whole heap.
*Common weak answer:* stating the hypothesis with no mechanism.

**Q11. How can GC affect a web service's latency? How would you tell?**
A strong answer must mention:
- STW pauses stall all in-flight requests. Only the few percent of requests caught by a pause are slowed, so p99 jumps while the average moves only slightly.
- Pauses can trip health checks and timeouts, causing retries and amplification.
- Evidence: correlate GC logs/metrics (pause times) with latency spikes.
- Mitigations: lower the allocation rate, right-size the heap, choose a low-latency collector.
*Common weak answer:* "GC makes Java slow" with no link to tail latency or evidence.

**Q12. Reference counting vs tracing GC?**
A strong answer must mention:
- RC frees immediately at count zero, but leaks cycles and pays per-assignment (atomic in multi-threaded code) overhead.
- Tracing handles cycles and has no per-assignment count cost, but reclaims in batches with pauses or concurrent overhead.
- CPython uses RC + a cycle collector; the JVM, Go and V8 use tracing.
*Common weak answer:* "RC has no pauses." Cascading frees can pause too.

**Q13. Can a garbage-collected program leak memory?**
A strong answer must mention:
- Yes: reachable-but-unused objects are never collected.
- Examples: unbounded caches/maps, unremoved listeners, per-thread storage on pooled threads.
- Symptoms: steadily rising memory, longer and more frequent GCs, out-of-memory.
- Diagnosis via heap dumps.
*Common weak answer:* "No, the GC handles it." GC frees only unreachable objects; anything still referenced, however useless, stays.

**Q14. Why does a process use more memory than its heap? Why does that matter in containers?**
A strong answer must mention:
- Thread stacks, JIT code cache, metadata, GC structures, native buffers, runtime libraries.
- A container memory limit applies to total process memory, so heap = limit leads to an OOM kill by the kernel.
- Leave headroom and measure under load.
*Common weak answer:* "Memory leak." A stable gap between heap and process memory is normal runtime overhead; a leak shows as steady growth.

**Q15. What is the GIL, and how do Python web servers deal with it?**
A strong answer must mention:
- A mutex allowing one thread to execute Python bytecode at a time in default CPython.
- Exists largely because of reference counting and interpreter simplicity.
- Threads help I/O-bound work (the GIL is released while waiting), not CPU-bound work.
- Servers use multiple worker processes; an optional free-threaded build exists in recent versions.
*Common weak answer:* "Python can't do concurrency." It can, for I/O, and via processes for CPU work.

**Q16. Why is Node.js called single-threaded, and what's the consequence for a backend?**
A strong answer must mention:
- User JavaScript runs on one thread driven by an event loop with non-blocking I/O.
- Excellent for many concurrent I/O-bound requests.
- CPU-heavy work blocks the loop, delaying all requests in that process.
- The runtime still uses internal threads (e.g. for file I/O, DNS), and you can scale across cores with multiple processes or worker threads.
*Common weak answer:* "Node can only handle one request at a time." It handles many concurrently, because waiting for I/O does not occupy the thread; only CPU work is serialised.

## Scenario Exercises

1. A service's average latency is 8 ms, but p99 is 450 ms. CPU averages 30%. Every few minutes there is a cluster of slow requests across *all* endpoints at once. List two hypotheses grounded in this chapter and the evidence that would distinguish them.
2. A team doubles the threads in a CPU-bound computation that increments a per-thread counter stored in a shared array (`counters[threadId]++`). Throughput drops. Explain why.
3. A Java service runs in a container with a 1 GB limit and `-Xmx1g`. It is killed periodically, and there is no `OutOfMemoryError` in its logs. Explain the mechanism and the fix.
4. A Node.js API serves 5,000 requests/second comfortably. A new endpoint that generates a 20 MB PDF synchronously is added. Shortly after it gets traffic, *all* endpoints show latency spikes. Why?
5. A Python team rewrote a CPU-heavy image-thumbnail step to use 8 threads on an 8-core machine and saw no speed-up. What happened, and what are two alternatives?
6. A newly deployed JVM instance shows p99 latency 5× higher than older instances for the first two minutes, then matches them. Name two causes and one mitigation.

### Solution Sketches

1. (a) Stop-the-world GC pauses: correlate GC log pause timestamps with the slow clusters. Pauses affect all endpoints simultaneously, and CPU can look moderate on average. (b) A host-level stall: CPU throttling from a container quota (Chapter 4) or a noisy neighbour; check throttling metrics. A third possibility, a slow shared dependency, would show in traces as time spent in one downstream call (Chapter 26).
2. False sharing: adjacent array elements share cache lines, so each core's write invalidates the others' copies and the line ping-pongs. Fix: pad each counter to its own cache line, or use local variables and combine at the end.
3. Process memory = heap + stacks + code cache + metadata + native buffers, which exceeds 1 GB. The kernel's cgroup OOM killer terminates the process (Chapter 4), and the runtime never reports a heap error. Fix: set the heap to a fraction of the limit (e.g. ~70–75%), measure RSS, or raise the limit.
4. The synchronous PDF generation runs on the single event-loop thread and blocks it. Every other request waits until it finishes. Fix: offload to worker threads or processes, or to a background job queue (Chapter 25).
5. The GIL lets only one thread execute Python bytecode at a time, so CPU-bound threads don't run in parallel (unless the library releases the GIL in native code). Alternatives: multiple processes (a process pool), or a native library that releases the GIL. Another option is the free-threaded build, after verifying library compatibility.
6. JIT warm-up (code is still interpreted or lightly optimised), plus cold caches and connection pools. Mitigation: warm-up traffic before marking the instance ready, or gradual traffic ramp-up (Chapter 26).

## Diagnosis Lab

**Lab 1: Periodic latency spikes in a JVM service**
- *Symptom:* p99 latency, computed over 10-second windows, jumps from 20 ms to 600 ms every 3–5 minutes. Average latency barely changes. Errors are rare except occasional client timeouts.
- *Hypotheses:* (a) long GC pauses; (b) a scheduled job competing for CPU; (c) a downstream dependency slowing periodically.
- *Evidence:* enable or inspect GC logs (pause start time and duration) and overlay them on the latency graph. Check per-dependency latency in traces (Chapter 26). Check CPU per thread or process.
- *Finding:* full collections of ~550 ms align exactly with the spikes. Heap used *after* each collection is ~85% of max, so the live set nearly fills the heap and the old generation fills quickly.
- *Root cause:* heap too small for the live set (an in-process cache grew over time).
- *Fix:* bound the cache (Chapter 21), increase heap headroom within the container limit, and consider a lower-pause collector.
- *Prevention:* alert on GC pause p99 and on heap-after-GC trends.

**Lab 2: Memory climbs until the process dies**
- *Symptom:* a service's memory rises steadily over ~2 days, GC frequency increases, then the process crashes with an out-of-memory error. A restart "fixes" it.
- *Hypotheses:* a leak through unintended reachability (unbounded map, listeners, per-thread data) vs a legitimately growing workload.
- *Evidence:* take two heap dumps hours apart and compare the largest retained objects. Identify the collection that keeps growing and the path from a GC root to it.
- *Finding:* a global map from session ID to user context, to which an entry is added on every login and never removed.
- *Fix:* store sessions in a proper session store with expiry (Chapter 19), or use a bounded cache with eviction (Chapter 21).
- *Prevention:* memory-after-GC trend alerts; code review rule against unbounded static collections.

**Lab 3: "The container keeps restarting, but the app logs nothing"**
- *Symptom:* a service in a container restarts every few hours with exit code 137, and no error appears in application logs.
- *Hypotheses:* the OS killed the process (exit code 137 = 128 + 9, i.e. killed by SIGKILL), most likely the memory-limit OOM killer, as opposed to an application crash.
- *Evidence:* container/orchestrator events report "OOMKilled". Kernel logs (`dmesg`) show an OOM kill. Process RSS approaches the limit, while the heap stays below its own max.
- *Root cause:* heap max set close to the container limit; non-heap memory pushes the total over.
- *Fix:* lower the heap fraction or raise the limit, and limit thread counts (each costs stack memory).
- *Prevention:* monitor RSS against the limit, not just heap usage. Mechanisms continue in Chapters 3 and 4.

---

**END OF CHAPTER 2**
