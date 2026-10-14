# Chapter 4: Operating Systems II: Files, I/O, IPC, and Isolation

## Where We Are

Chapter 1 described a web application as a set of processes on several machines, talking over networks. Chapter 2 went down to the hardware: a CPU running instructions, a memory hierarchy in which every level is slower and larger than the one above it, and storage devices (hard disks and SSDs) that are thousands to millions of times slower than RAM. Chapter 3 introduced the operating system as the software that shares one machine among many programs. It explained processes and threads, the system call as the doorway into the kernel, the scheduler that decides which thread runs, and virtual memory, which gives every process a private address space.

Chapter 3 deliberately covered only the CPU and memory side of the kernel. A backend server spends most of its life doing something else: **waiting for input and output**. It waits for bytes from a client's TCP connection, for a reply from the database, for a log line to reach the disk. This chapter covers that other half of the operating system: how a process names the things it reads and writes (file descriptors), how files are stored and cached, what it really takes for data to survive a power cut, how one thread can wait on ten thousand network connections at once, how processes on the same machine talk to each other, and how the kernel fences processes off from one another, which is what a "container" really is.

These mechanisms are behind some of the most common production incidents: "Too many open files" errors; a disk that is full although `du` finds nothing large; a database that loses committed data after a power failure; a server whose clients connect but never get a response; a container that is killed with no error in its logs; a deploy that fails a few hundred requests every time. By the end of this chapter you can explain each one and find the evidence for it.

**After this chapter you can:**
- explain what a file descriptor is and trace the three kernel tables between a file descriptor number and the data on disk;
- explain inodes, directories, hard and symbolic links, and what a filesystem journal does and does not protect;
- trace a `read()` and a `write()` through the page cache, and explain dirty pages and writeback;
- state precisely what `write()` returning means and what `fsync()` returning means, and explain why databases care;
- compare blocking, non-blocking, multiplexed, signal-driven and asynchronous I/O, and explain why `epoll` scales where `select` and `poll` do not;
- trace the system calls a server and a client make to establish a TCP connection, including the SYN queue and the accept queue;
- explain zero-copy transfer and why it matters for proxies and file servers;
- compare pipes, Unix domain sockets, shared memory, message queues and signals, and explain why services usually talk over the network instead;
- diagnose file-descriptor exhaustion and other resource-limit failures;
- explain a container as an ordinary Linux process plus namespaces and cgroups, and compare containers with virtual machines;
- run a server process correctly inside a container: PID 1, signal forwarding and graceful shutdown;
- use `lsof`, `strace`, `iostat`, `ss` and `dmesg` to diagnose FD leaks, CPU throttling, full disks and slow `fsync`.

**Interview questions this chapter prepares you for:**
1. What is a file descriptor? What happens to file descriptors across `fork`?
2. What is an inode? What is the difference between a hard link and a symbolic link?
3. What does a journaling filesystem protect against, and what does it not protect against?
4. What is the page cache? Why does Linux show almost no "free" memory?
5. What does it mean when `write()` returns? Why does `fsync` matter, and does `O_DIRECT` make a write durable?
6. What is the difference between blocking, non-blocking and asynchronous I/O?
7. How does `epoll` work, and why does it scale better than `select` and `poll`? What is the difference between level-triggered and edge-triggered notification?
8. What is the C10K problem, and how was it solved?
9. Walk through the system calls a TCP server makes. What is the listen backlog?
10. What is zero-copy? How does `sendfile` work?
11. What inter-process communication mechanisms exist, and when would you use each?
12. What causes "Too many open files", and how do you diagnose it?
13. What is the difference between a virtual machine and a container? What are namespaces and cgroups?
14. Why is a container less isolated than a virtual machine?
15. Why might a containerised application ignore `SIGTERM`, and how do you fix it?

**Scope.** Here: file descriptors, filesystems, the page cache and durability, I/O models, the socket system calls, zero-copy, inter-process communication, resource limits, and isolation with virtual machines, namespaces and cgroups. Processes, signals, the scheduler, virtual memory and the OOM killer were taught in Chapter 3 and are only recalled. Race conditions, locks and the programming model of event loops (callbacks, futures, coroutines) are in Chapter 5. TCP itself (the handshake, retransmission, `TIME_WAIT`, ports) is in Chapter 6. How a server combines these system calls into an accept → parse → dispatch pipeline is in Chapter 10, and how it sizes its thread pools is in Chapter 11. How a database uses `fsync`, the write-ahead log and its own buffer pool is in Chapter 15. Container orchestration is in Chapter 26.

---

## 4.1 Everything Is a File: File Descriptors

### Definition

A **file descriptor (FD)** is a small non-negative integer that a process uses to refer to an open I/O resource: a file, a directory, a pipe, a network socket, a terminal or a device. The kernel hands out the number when the resource is opened, and the process passes it to every later system call (`read`, `write`, `close`) that operates on that resource.

"**Everything is a file**" is the Unix design principle that lets these very different resources be used through the same small set of system calls. More precisely: everything is *accessed through a file descriptor*, and a file descriptor supports a common interface, even when the thing behind it is not a file on disk.

### Why it exists

Without a common abstraction, every kind of I/O would need its own API: one set of calls for disk files, another for network connections, another for terminals, another for pipes between processes. Programs would be harder to write and impossible to compose. A shell pipeline such as `grep error app.log | sort | uniq -c` works because `grep` writes to "file descriptor 1" without knowing or caring whether that is a terminal, a file or a pipe into `sort`.

There is a second reason. Application code runs in user mode and cannot touch devices or kernel memory (Chapter 3, Section 3.2). The kernel must keep its own records about each open resource (where the file is on disk, how far into it you have read, which TCP connection a socket represents). The process needs a *handle* to refer to that record, and the handle must be safe to give to untrusted code. A small integer is ideal: it means nothing outside the process's own table, so the process cannot forge a pointer into kernel memory. It can only name entries the kernel has already created for it.

### Mechanism: three tables between a number and the data

When a process calls `open("/var/log/app.log", O_WRONLY | O_APPEND)`, the kernel builds or reuses three structures. Understanding all three explains almost every surprising FD behaviour.

1. **The per-process file descriptor table.** Each process has an array indexed by FD number. Entry *n* holds a pointer to an open file description (below), plus a few per-descriptor flags, of which the important one is *close-on-exec*. The table lives in the kernel and is referenced from the process's Process Control Block (Chapter 3, Section 3.3). Threads of the same process share one table.
2. **The system-wide open file table.** Each entry is an **open file description** (Linux calls the structure `struct file`). It records the state of *one act of opening*: the current **file offset** (how far into the file the next `read` or `write` will happen), the access mode (read, write or both), status flags such as `O_APPEND` and `O_NONBLOCK`, and a pointer to the inode. Several FDs, in one process or in several, can point to the same open file description.
3. **The inode table.** Each entry is the kernel's in-memory copy of an **inode**: the object that represents the file itself, including its size, owner, permissions, timestamps and the location of its data blocks (Section 4.2). Every open file description for the same file points to the same inode, no matter how many times or by how many processes the file was opened.

```text
   Process A (PID 4100)            System-wide                     Inodes
   FD table                        open file table                 (one per file)
  ┌────┬────────┐               ┌───────────────────────┐
  │ 0  │   ─────┼──────────────▶│ /dev/pts/0  off=—  RW │──────▶ [inode: tty]
  │ 1  │   ─────┼──────────┐    ├───────────────────────┤
  │ 2  │   ─────┼──────────┤    │ app.log  off=8192     │
  │ 3  │   ─────┼───┐      └───▶│ O_WRONLY|O_APPEND     │──┐
  │ 4  │   ─────┼─┐ │           ├───────────────────────┤  │
  └────┴────────┘ │ └──────────▶│ data.db  off=4096  RW │──┼──▶ [inode 917: data.db]
                  │             ├───────────────────────┤  │          ▲
   Process B      │      ┌─────▶│ data.db  off=0     RO │──┼──────────┘
  ┌────┬────────┐ │      │      ├───────────────────────┤  │
  │ 3  │   ─────┼─┼──────┘      │ socket  (TCP 10.0.0.5 │  └──▶ [inode 523: app.log]
  └────┴────────┘ └────────────▶│ :8080 ↔ 10.0.1.9:5143)│──────▶ [socket inode]
                                └───────────────────────┘
```

Read the diagram as answers to three questions:

- **Who has the file open?** Process A's FD 3 and process B's FD 3 both lead to inode 917. Same file.
- **Do they share a position?** No. Each called `open` separately, so each has its own open file description with its own offset. A reads at offset 4096 while B reads at offset 0.
- **Can two FDs share a position?** Yes. In process A, FD 1 and FD 2 both point to the same `app.log` open file description, so standard output and standard error append to the log through one shared offset. This happens when one FD is copied from another with `dup2`, as a shell does for `2>&1`.

The **file offset lives in the open file description, not in the FD and not in the inode.** That single fact explains these behaviours:

- **`fork` shares offsets.** `fork` (Chapter 3, Section 3.5) copies the parent's FD *table*, but the copied entries point to the *same* open file descriptions. A parent and child that both write to an inherited FD advance one shared offset, so their writes do not overwrite each other. A parent and child that both *read* from an inherited FD split the data between them, each seeing only the bytes the other did not consume.
- **`dup` and `dup2` share offsets.** They create a second FD pointing at the same open file description.
- **Two separate `open` calls do not share offsets.** Two processes that each open the same log file without `O_APPEND` and write will overwrite each other's bytes, because each starts from its own offset 0. With `O_APPEND`, the kernel moves the offset to the end of the file atomically before every write, which is why loggers open files in append mode.

**Allocation rule.** When a new FD is needed, the kernel uses the **lowest-numbered free slot** in the process's table. By convention FDs 0, 1 and 2 are **standard input**, **standard output** and **standard error**, inherited from the parent, so the first file a program opens usually gets FD 3. The lowest-free rule has a dangerous consequence: if you close FD 7 and then open a socket, the socket is likely to get FD 7. Code that still holds the old number (a bug called a stale or double-closed FD) now reads from or writes to the wrong resource, and it may write a log line into a client's HTTP response.

**Reference counting and `close`.** `close(fd)` removes the entry from the process's table and decrements the reference count on the open file description. The description is destroyed only when its count reaches zero, that is, when every FD in every process that pointed to it has been closed. For a socket, the TCP connection is only shut down then. A parent that forks a child for each connection must close its own copy of the connection's FD, or the connection never closes when the child finishes.

**Close-on-exec.** `exec` (Chapter 3, Section 3.5) replaces a process's program but keeps its FD table. Every open FD, including database connections and secret files, is inherited by whatever program is executed, unless it was marked **close-on-exec** (opened with `O_CLOEXEC`, or flagged later with `fcntl`). Well-written runtimes set this flag on every FD they open by default, so that a child program started with `exec` cannot accidentally hold the server's listening socket or read its files.

### What can sit behind an FD

| Resource | Created by | `read` / `write` mean | Notes |
|---|---|---|---|
| Regular file | `open` | bytes at the current offset | Supports `lseek` (move the offset) |
| Directory | `open` with `O_DIRECTORY` | not readable with `read`; listed with `getdents` | Section 4.2 |
| Pipe / FIFO | `pipe`, `open` on a FIFO | a one-way byte stream | No offset; Section 4.8 |
| TCP socket | `socket`, `accept` | a two-way byte stream to a peer | No offset; Section 4.6 |
| UDP socket | `socket` | one datagram per call | Chapter 6 |
| Terminal | inherited | keyboard input / screen output | |
| `epoll` instance | `epoll_create1` | not used with read/write | Section 4.5 |
| `eventfd`, `timerfd`, `signalfd` | their own calls | counters, timer expiries, signals delivered as data | Lets an event loop wait for these too |

The last two rows show how far the principle was pushed in Linux. Timers and signals were turned into FDs so that a single waiting call (Section 4.5) can wait for network data, a timer and a signal at the same time.

### Concrete: watching FDs with `strace` and `/proc`

`strace` (Section 4.12) prints each system call a process makes. Here is `cat notes.txt` reading a small file and writing it to the terminal:

```text
$ strace -e trace=openat,read,write,close cat notes.txt
openat(AT_FDCWD, "notes.txt", O_RDONLY) = 3
read(3, "order 1001 shipped\norder 1002 pe"..., 131072) = 38
write(1, "order 1001 shipped\norder 1002 pe"..., 38) = 38
read(3, "", 131072)                     = 0
close(3)                                = 0
```

The kernel returned FD 3 (0–2 were taken). `read` returned 38 bytes, then 0. **A return of 0 from `read` means end of file**, or for a socket or pipe, that the other end has closed its side. `write(1, ...)` sent the bytes to standard output.

On Linux you can list any process's FD table through the `/proc` filesystem (Chapter 3, Section 3.19). Each entry is a symbolic link to what the FD refers to:

```text
$ ls -l /proc/4100/fd
lr-x------ 1 app app 64 Oct 14 10:02 0 -> /dev/null
l-wx------ 1 app app 64 Oct 14 10:02 1 -> /var/log/app/app.log
l-wx------ 1 app app 64 Oct 14 10:02 2 -> /var/log/app/app.log
lrwx------ 1 app app 64 Oct 14 10:02 3 -> /srv/data/data.db
lrwx------ 1 app app 64 Oct 14 10:02 4 -> socket:[48213]
lrwx------ 1 app app 64 Oct 14 10:02 5 -> anon_inode:[eventpoll]
lrwx------ 1 app app 64 Oct 14 10:02 6 -> socket:[48977]
```

`socket:[48213]` names a socket by its inode number; `ss -e` or `lsof` (Section 4.12) will show you which TCP connection that is. `anon_inode:[eventpoll]` is an `epoll` instance (Section 4.5). `ls /proc/4100/fd | wc -l` counts open FDs, the first number to check when you suspect a leak.

### Guarantees and non-guarantees

The kernel guarantees that an FD number is meaningful only inside its own process, that a new FD is the lowest free number, that the resource stays alive while any FD refers to its open file description, and that `O_APPEND` writes go to the current end of file. It does **not** guarantee that `read` returns as many bytes as you asked for (a "short read" is normal, especially on sockets and pipes), that `write` writes everything (a short write is possible on sockets and pipes, so the caller must loop), that writes from different open file descriptions without `O_APPEND` will not overwrite each other, or that an FD number refers to the same resource after it has been closed.

### Failure and diagnosis

- **FD leak:** code opens files or sockets and does not close them on every path (typically an error path). The count in `/proc/<pid>/fd` grows steadily with traffic and never falls. When it reaches the process's limit, every `open`, `socket`, `accept` and `connect` fails with `EMFILE`, "Too many open files" (Section 4.9).
- **Double close / use after close:** a bug closes an FD twice. In between, another thread opened something and received the same number, which the second `close` now closes. Symptoms are bizarre: a database connection that fails with "bad file descriptor", or response bytes appearing in a log file. `strace -f -e trace=close,openat,socket,accept4` shows the same number being closed twice.
- **Inherited FDs:** a child process started with `exec` keeps the server's listening socket open, so after the server restarts, the port is "already in use", or a socket that the server closed stays open in the child. `ls -l /proc/<child>/fd` shows the unexpected entries; the fix is close-on-exec.

### Backend connection

A backend server is, from the kernel's point of view, a process holding a few thousand FDs: one listening socket, one socket per client connection, one socket per pooled database connection, one per outbound HTTP connection, plus log files and `epoll` instances. Each is a kernel object that costs memory and counts against a limit. "How many connections can this server hold?" is partly "how many FDs may it have?", and the first question in many incidents is "how many FDs does it have now, and what are they?"

> **Java lens:** Java hides FDs behind objects (`FileInputStream`, `Socket`, `FileChannel`), but every one of these holds a real OS FD until `close()` is called. The garbage collector does not close FDs promptly; an unreachable but unclosed stream can hold its FD for minutes. Use try-with-resources (`try (var in = Files.newInputStream(p)) { ... }`) so the FD is closed on every path, including exceptions. The JVM opens its own files with close-on-exec set.

---

## 4.2 Filesystems: Inodes, Directories, Links, and Journaling

### Definitions

A **filesystem** is the on-disk data structure, and the kernel code that manages it, that turns a storage device (a flat array of numbered blocks, Chapter 2, Section 2.4) into a tree of named files and directories with permissions and timestamps. Examples on Linux are ext4, XFS and Btrfs.

An **inode** ("index node") is the filesystem's record for one file. It stores the file's **metadata** (type, size, owner, permissions, timestamps, link count) and the **location of its data blocks**. It does **not** store the file's name.

A **directory** is a special file whose contents are a list of entries, each mapping a **name** to an **inode number**.

A **hard link** is a directory entry. Every name of a file is a hard link to its inode. A **symbolic link** (symlink) is a separate small file whose content is a path to another file.

**Journaling** is a technique in which the filesystem first records the changes it is about to make in a separate area (the journal), and only then applies them in place, so that a crash in the middle of an update can be repaired quickly to a consistent state.

### Why they exist

A disk offers only "read block 1,048,576" and "write block 1,048,576". Programs want "open `/srv/uploads/invoice-17.pdf` and append 4 KB". Something must record which blocks belong to which file, which blocks are free, and how names map to files. That something is the filesystem.

Separating the name (in the directory) from the file (the inode) gives three useful properties: a file can have several names; a file can be renamed or moved within the filesystem by changing only directory entries, never copying data; and a file can keep existing after its last name is removed, as long as someone still has it open.

Journaling exists because updating a filesystem takes several separate disk writes. Creating a file means writing a new inode, marking it as used in the inode bitmap, adding an entry to the directory, and, once data is written, marking data blocks as used and pointing the inode at them. If the power fails after some of these writes and before others, the on-disk structure is **inconsistent**: a directory entry pointing at an unused inode, or blocks marked used that belong to no file. Before journaling, the repair tool (`fsck`) had to scan the entire disk after every crash, which could take hours on a large volume.

### Mechanism: from a path to data blocks

A filesystem divides its disk space into regions: a superblock (describing the filesystem), bitmaps of free inodes and free blocks, the **inode table** (a fixed-size array of inodes on many filesystems, such as ext4), the journal, and the data blocks.

```text
 ext4-style layout (simplified, one block group)
┌──────────┬────────────┬────────────┬──────────────┬──────────────────────────┐
│superblock│ block      │ inode      │ inode table  │ data blocks              │
│(fs size, │ bitmap     │ bitmap     │ [1][2]...[N] │ file contents and        │
│ counts)  │ (free?)    │ (free?)    │              │ directory contents       │
└──────────┴────────────┴────────────┴──────────────┴──────────────────────────┘

 Inode 917 ("data.db")                       Data blocks
┌──────────────────────────────┐
│ type: regular file           │
│ mode: rw-r----- owner: app   │
│ size: 3,145,728 bytes        │
│ links: 1                     │
│ mtime, ctime, atime          │
│ extents:                     │
│   blocks 40960–41727 ────────┼──────────▶ [768 contiguous 4 KB blocks]
└──────────────────────────────┘
```

Older filesystems listed every data block individually (with indirect blocks for large files). Modern ones such as ext4 and XFS use **extents**: "768 blocks starting at block 40960". Extents are compact and encourage contiguous placement, which helps sequential reads.

**Path resolution.** To open `/srv/data/data.db`, the kernel:

1. starts at the root directory's inode (a known number);
2. reads the root directory's contents and looks up the entry `srv`, getting its inode number;
3. checks that the process has *execute* (search) permission on that directory, reads its contents and looks up `data`;
4. repeats for `data.db`, getting inode 917;
5. checks the permissions in inode 917 against the process's user and the requested access mode;
6. creates an open file description pointing to the in-memory inode and returns an FD (Section 4.1).

The kernel caches directory lookups in the **dentry cache** (directory entry cache) so that repeated opens of the same path do not reread directories from disk.

### Concrete: inodes and links on a real system

```text
$ stat data.db
  File: data.db
  Size: 3145728     Blocks: 6144       IO Block: 4096   regular file
Device: 259,2       Inode: 917         Links: 1
Access: (0640/-rw-r-----)  Uid: ( 1001/   app)   Gid: ( 1001/   app)

$ ln data.db data-hardlink.db        # second name for inode 917
$ ln -s data.db data-symlink.db      # new file whose content is "data.db"
$ ls -li
917 -rw-r----- 2 app app 3145728 Oct 14 10:00 data.db
917 -rw-r----- 2 app app 3145728 Oct 14 10:00 data-hardlink.db
1204 lrwxrwxrwx 1 app app       7 Oct 14 10:01 data-symlink.db -> data.db
```

Both hard links show inode 917 and a link count of 2: they are two equal names for one file. Neither is "the original". The symlink has its own inode (1204) and its size is 7 bytes, the length of the text `data.db`.

| | Hard link | Symbolic link |
|---|---|---|
| What it is | Another directory entry for the same inode | A separate file containing a path |
| Survives deleting the "original" name | Yes; the data stays until the link count is 0 | No; it becomes a dangling link |
| Can cross filesystems | No (inode numbers are per filesystem) | Yes |
| Can point to a directory | No (prevents loops in the tree) | Yes |
| Typical backend use | Atomic snapshots of log or data files, deduplicated deploy directories | `current -> releases/2026-10-14` for deploys; config indirection |

### Deleting a file that is open

`rm` calls `unlink`, which removes **one directory entry** and decrements the inode's link count. The kernel frees the inode and its data blocks only when **both** the link count is 0 **and** no open file description refers to it. So:

```text
 t0  logger process opens /var/log/app.log         (links=1, open=1)
 t1  operator runs rm /var/log/app.log             (links=0, open=1)
     → the name is gone; `ls` and `du` no longer see the file
     → the inode and its 40 GB of data blocks still exist
 t2  logger keeps writing; the invisible file keeps growing
 t3  disk full. `df` says 100%, `du -sh /` adds up to far less
 t4  logger process restarts (or closes the FD) → space finally freed
```

This is the classic "disk is full but I can't find the files" incident. `lsof +L1` lists open files whose link count is below 1, that is, deleted but still open (Section 4.12, Lab 3). Log rotation tools avoid it either by asking the program to reopen its log file (typically with `SIGHUP`, Chapter 3, Section 3.6) or by truncating the file in place ("copytruncate").

### Rename is atomic

`rename(old, new)` within one filesystem replaces the directory entry `new` with the inode of `old` in one step: any other process opening `new` sees either the complete old file or the complete new file, never a mixture and never "no file". This is the basis of the standard pattern for safely replacing a file's contents (completed with `fsync` in Section 4.4):

```pseudo
write the new contents to "config.json.tmp"
make the new contents durable           // fsync, Section 4.4
rename("config.json.tmp", "config.json")
make the rename durable                 // fsync the directory, Section 4.4
```

Writing directly over `config.json` instead risks a reader (or a crash) seeing a half-written file.

### Mechanism: journaling

A journaling filesystem turns a multi-write update into a **transaction**:

```text
  1. write the intended changes (new inode, bitmap bits, directory block)
     to the journal, followed by a commit record        ──▶ journal on disk
  2. wait until the journal blocks are on stable storage
  3. write the same changes to their real locations      ──▶ "checkpoint"
  4. mark the journal space as reusable

  Crash before the commit record is durable  → discard the incomplete entry;
                                               the old state is intact
  Crash after the commit record is durable   → on mount, replay the journal
                                               (redo the writes); new state
```

Recovery after a crash takes seconds, because only the journal needs to be read, not the whole disk. This "write your intention durably first, then apply it, and replay it after a crash" is the same idea a database's write-ahead log uses (Chapter 15).

Filesystems differ in **what** they journal. ext4 offers three modes:

| ext4 mode | What is journaled | Effect after a crash |
|---|---|---|
| `data=ordered` (the usual default) | Metadata only, but data blocks are written to disk *before* the metadata that points to them is committed | Structure consistent; a file never points at blocks containing someone else's old data; recently written file contents may be missing or old |
| `data=writeback` | Metadata only, no ordering for data | Structure consistent; a file may contain stale garbage from previously freed blocks |
| `data=journal` | Metadata and data | Slower (data is written twice); contents of committed writes are consistent |

XFS journals metadata only. Copy-on-write filesystems such as Btrfs and ZFS take a different approach: they never overwrite live blocks in place, so a crash leaves either the old tree or the new tree.

### Guarantees and non-guarantees

A journal guarantees that the **filesystem's own structure** is consistent after a crash: no directory entry points to a free inode, no block belongs to two files, free-space counts are correct. It does **not** guarantee:

- that data you wrote recently is on disk (it may still be in the page cache, Section 4.3; only `fsync` gives that, Section 4.4);
- that a file's contents are consistent from your application's point of view (if your program wrote half of a record, the journal faithfully preserves half a record);
- that several writes to several files happen atomically together (the filesystem has no notion of your transaction; databases build their own, Chapter 15 and Chapter 17);
- protection from disk hardware failure or bit rot (journaling is not replication or a checksum; some filesystems, such as ZFS and Btrfs, add data checksums).

### Failure and diagnosis

- **Disk full** (`ENOSPC`, "No space left on device"): writes fail; databases may refuse to start or stop accepting writes; logs stop. Check `df -h` for space, then `du` to find where it is; if they disagree, look for deleted-but-open files with `lsof +L1`.
- **Inodes exhausted:** `df -h` shows free space, but creating files fails with "No space left on device". `df -i` shows `IUse%` at 100%. Typical cause: millions of tiny files (session files, cache entries, mail spool). ext4 fixes the number of inodes when the filesystem is created.
- **Read-only remount:** after detecting corruption or an I/O error, ext4 can remount itself read-only (depending on its `errors=` mount option), and every write fails with "Read-only file system". `dmesg` (Section 4.12) shows the I/O errors that triggered it.
- **Torn configuration file** after a crash or a concurrent reader: the program overwrote the file in place. Use the write-temporary-then-rename pattern.

### Backend connection

Application servers touch the filesystem for logs, uploaded files, temporary files and configuration; databases and message brokers store everything in it. Three habits follow from this section: replace files atomically with rename; never assume a journal makes your data safe; and alert on both disk space and inode usage, with headroom, because a full disk tends to stop several things at once (the database, the logger, the message broker).

---

## 4.3 The Read/Write Path and the Page Cache

### Definition

The **page cache** is the part of RAM in which the Linux kernel keeps copies of file data, organised in pages (usually 4 KB, Chapter 3, Section 3.14). Ordinary `read()` and `write()` calls do not touch the disk directly: they copy bytes between the process's buffer and the page cache, and the kernel moves data between the page cache and the disk separately.

A **dirty page** is a page-cache page that has been modified by a write and not yet written back to disk. **Writeback** is the kernel's process of writing dirty pages to the storage device, after which they become clean.

### Why it exists

Chapter 2 (Section 2.2) gave the gap: reading RAM takes on the order of 100 nanoseconds; reading from an SSD takes tens to hundreds of microseconds; a random read on a hard disk takes several milliseconds. If every `read()` went to the device, a server rereading the same configuration file or the same hot database page would wait thousands of times longer than necessary. If every `write()` waited for the device, a program writing a log line per request would be limited by disk latency.

The page cache attacks both problems:
- **Reads:** data read once stays in RAM, so the next read of it is a memory copy. The kernel also **reads ahead**: when it sees a process reading a file sequentially, it fetches the next pages before they are asked for.
- **Writes:** a write completes as soon as the bytes are in RAM. The kernel can later write many small changes to the same page as one disk write, sort writes by location, and write large contiguous ranges, which suits both disks (sequential) and SSDs (large writes reduce write amplification, Chapter 2, Section 2.4).

### Mechanism: the read path

```text
 user space                  │  kernel                                    │  device
                             │                                            │
 buf[8192]  ◀── copy ─────── │◀── page cache: is page (inode 917,         │
     ▲                       │    offset 4096) present?                   │
     │                       │        │yes: copy it (≈ microseconds)       │
 read(fd, buf, 8192)  ──────▶│        │no:  allocate a page, issue a read  │
                             │        │     to the block layer ───────────▶│ SSD / disk
                             │        │     thread sleeps (state D or S)   │
                             │        │◀──── interrupt: data arrived ──────│
                             │        │     page now cached; copy it       │
                             │    readahead: if access looks sequential,  │
                             │    also queue reads for the next pages     │
```

1. The process calls `read(fd, buf, 8192)`. The kernel finds the open file description, its offset and its inode.
2. For each page covering the requested range, it looks the page up in the page cache (an index per inode, keyed by page offset).
3. **Cache hit:** the kernel copies the bytes from the cached page into `buf`. No device I/O happens.
4. **Cache miss:** the kernel allocates a page, asks the block layer to read it from the device, and puts the thread to sleep. When the device signals completion with an interrupt (Chapter 3, Section 3.2), the thread is woken and the copy proceeds. A thread waiting here is the classic **uninterruptible sleep** (`D` state) that shows up in load average and iowait (Chapter 3, Section 3.12).
5. The offset advances by the number of bytes returned.

A memory-mapped file (`mmap`, Chapter 3, Section 3.15) uses the **same** page cache pages, mapped directly into the process's address space instead of copied. That is why a file read with `read()` by one process and mapped by another shows the same data: there is only one cached copy.

### Mechanism: the write path

```text
 write(fd, buf, 4096)
   │
   ▼
 1. copy buf into the page cache page for (inode, offset)   ← may first read the
    (allocate the page if needed)                             page if only part of
 2. mark the page DIRTY; update size and mtime in the inode   it is overwritten
 3. return 4096 to the process        ◀── write() IS DONE HERE. The disk has
                                          not been touched.
   ⋮  (milliseconds to tens of seconds later)
 4. writeback (kernel flusher threads) picks dirty pages when:
      • a page has been dirty longer than the expiry interval, or
      • dirty memory exceeds the background threshold, or
      • memory is needed and dirty pages must be cleaned before reuse, or
      • someone calls fsync/sync on the file (Section 4.4)
 5. block layer → device driver → device; on completion the page is CLEAN
 6. clean pages can be dropped instantly when RAM is needed (they are on disk)
```

Linux controls writeback with a few tunable settings, visible under `/proc/sys/vm/`: `dirty_background_ratio` (or `_bytes`), the amount of dirty memory at which background writeback starts; `dirty_ratio` (or `_bytes`), the amount at which **a process that writes is itself made to wait** while writeback catches up; and `dirty_expire_centisecs`, how old a dirty page may get before it is written regardless. Their defaults differ between distributions and kernel versions; check them on your system rather than assuming.

The second threshold matters for latency. A process that writes faster than the disk can absorb (a bulk export, a chatty debug logger) fills the page cache with dirty pages until it hits the limit, and from then on its `write()` calls stall, sometimes for hundreds of milliseconds. The "fast" path of buffered writes is only fast while the disk keeps up on average.

### Concrete: where the memory went

Because the page cache uses otherwise idle RAM, a healthy Linux server shows very little "free" memory:

```text
$ free -m
               total        used        free      shared  buff/cache   available
Mem:           64035       21310        1204          88       41521       41870
Swap:              0           0           0
```

- `used` (21 GB) is mostly anonymous memory: process heaps and stacks.
- `buff/cache` (41 GB) is mostly the page cache.
- `free` (1.2 GB) is RAM holding nothing at all. Low is normal.
- **`available` (41.8 GB) is the number that matters**: the kernel's estimate of how much memory could be given to processes right now, mostly by dropping clean cache pages.

Dirty data in flight is visible in `/proc/meminfo`:

```text
$ grep -E '^(Dirty|Writeback):' /proc/meminfo
Dirty:            812344 kB
Writeback:         20480 kB
```

`Dirty` is data written by processes that is not yet on disk; `Writeback` is data being written right now. A `Dirty` value that keeps growing during a bulk write shows the disk falling behind.

**Benchmark trap.** Timing `dd if=/dev/zero of=test bs=1M count=1024` reports a speed of several GB/s on a slow disk, because `dd` only measured copies into the page cache. Add `conv=fsync` (wait for the data to reach the device before reporting) or `oflag=direct` (bypass the cache, Section 4.4) to measure the device.

### Guarantees and non-guarantees

The page cache guarantees **coherence on one machine**: once `write()` returns, any later `read()` or `mmap` of that file by any process on the same machine sees the new data, even though the disk does not have it yet. It does **not** guarantee durability (a crash or power failure loses dirty pages), it does not guarantee *when* writeback happens, it does not guarantee the order in which dirty pages reach the disk (page 9 may be written before page 2), and it does not make a cold read fast: the first read after a restart, or after the kernel evicted the page, goes to the device.

### Failure and diagnosis

- **"The server is out of memory!"** while `free` is low and `available` is high. Not a problem: it is the page cache. Look at `available` and at major page fault rates (Chapter 3, Section 3.15) before acting.
- **Write stalls:** request latency spikes during a bulk write or log burst. `Dirty` in `/proc/meminfo` is near its limit; `iostat` (Section 4.12) shows the device at high utilisation; threads are in `D` state in the write path. Fixes: write less, spread writes out, move bulk writes off the request path, or put logs on a different device.
- **Cold-cache slowness after restart:** a database or file server is slow for minutes after a reboot, then recovers. The page cache (and the database's own cache, Chapter 15) is refilling. Evidence: high read IOPS and many major faults, falling over time.
- **Container memory "leak" that is really cache:** under cgroups, page cache pages created by a container's processes are **charged to that container's memory usage** (Section 4.10). A container that reads large files can show memory near its limit. Clean cache is reclaimed before an OOM kill, but dirty pages must be written first, and a container that writes heavily under a tight limit can stall or be OOM-killed. Look at the cgroup's `memory.stat` file, which separates `anon` (process memory) from `file` (cache).

### Backend connection

The page cache is why a database server is usually given far more RAM than its processes appear to use; why the second run of a report is faster than the first; and why the same service can be fast on one host and slow on another with identical CPU. Some databases rely on the page cache as their main cache (PostgreSQL uses both its own shared buffer pool and the OS page cache); others bypass it with `O_DIRECT` and manage their own (Section 4.4 and Chapter 15). For your own code: buffered writes are fast but not safe until synced, and a log-heavy request path is a disk-latency path in disguise.

---

## 4.4 Durability: write, fsync, and O_DIRECT

### Definitions

**Durability** is the property that data, once reported as saved, survives a crash, a process kill, a kernel panic or a power failure.

**`fsync(fd)`** is the system call that asks the kernel to write all of a file's dirty data **and** the metadata needed to find it to the storage device, and to ask the device to make it stable, and that **does not return until the device reports completion**. **`fdatasync(fd)`** is the same, except that it skips metadata not needed to read the data back (such as the last-modified time), which saves a write.

**`O_DIRECT`** is an `open` flag that tells the kernel to transfer data directly between the process's buffer and the device, **bypassing the page cache**.

`O_SYNC` and `O_DSYNC` are `open` flags that make every `write()` on that FD behave as if it were followed by `fsync` or `fdatasync` respectively.

### Why it exists

Section 4.3 showed that `write()` returns when the bytes are in RAM. For a log line that is fine: losing the last second of logs in a power failure is acceptable. For a database it is not. If a bank's database tells a client "your transfer is committed" and then loses the record in a power failure, the client's money and the bank's books disagree forever. A database must be able to say "this commit is on stable storage" before telling anyone it happened, so the kernel must offer a call that waits for the device. That call is `fsync`.

### Mechanism: what each return means

```text
 time ──────────────────────────────────────────────────────────────────────▶

 app:   write(fd, rec, 200) ─┐            fsync(fd) ──────────────────────┐
                             │ returns                │ BLOCKS            │ returns
                             ▼ ≈ µs                   ▼                   ▼ ≈ 0.05–10 ms
 page cache:   [page DIRTY]────────────────── writeback of this file's dirty pages
                                                  │
 filesystem:                                      ├─ write the inode / journal commit
                                                  │  if metadata changed (size, extents)
 block layer:                                     ├─ submit I/O to the device
                                                  │
 device:      volatile write cache ◀── data ──────┤
                                                  ├─ FLUSH command (or FUA writes):
                                                  │  "move your cache to non-volatile
                                                  │   media before you answer"
              NAND / platter (non-volatile) ◀─────┘  device: "done" ──▶ fsync returns
```

| When this returns... | ...this is true | ...and this is NOT true |
|---|---|---|
| `write()` | The bytes are in the page cache. Every process on this machine will read the new data. | The data is on any disk. A power failure or kernel crash can lose it. |
| `fsync()` with return value 0 | The kernel wrote the file's data and the metadata needed to find it, and the device acknowledged a cache flush. The data should survive a power failure, if the device is honest. | The directory entry for a **newly created** file is durable (that is the directory's metadata; `fsync` the directory too). Any *other* file is durable. |
| `fsync()` returning an error (for example `EIO`) | Some writeback failed. | That retrying `fsync` will fix it. See "fsyncgate" below. |
| `write()` on an `O_DIRECT` FD | The data was transferred to the device (or its cache), bypassing the page cache. | The data is durable. The device's volatile cache may still hold it; you still need `fsync`/`fdatasync` or `O_DSYNC`. |

`O_DIRECT` is therefore a **cache-control** option, not a durability option. It requires the buffer, offset and length to be aligned to the device's block size, and every read goes to the device. Databases use it because they keep their own cache of pages (Chapter 15) and do not want the same data cached twice (once by the database, once by the kernel) or the kernel's writeback deciding when their pages are written.

### The new-file trap

Creating a file and syncing it is not enough:

```pseudo
fd = open("orders-00042.log", O_CREAT | O_WRONLY)
write(fd, records)
fsync(fd)          // data and the file's inode are durable
close(fd)
// after a crash, the file may still be MISSING:
// its name lives in the directory, and the directory was never synced
dirfd = open(".", O_RDONLY | O_DIRECTORY)
fsync(dirfd)       // now the directory entry is durable too
```

The same applies to `rename` in the atomic-replace pattern of Section 4.2: after the rename, `fsync` the directory. Some filesystems happen to make this unnecessary in some modes, but portable software, including databases, does it explicitly.

### Why `fsync` is expensive, and what databases do about it

`fsync` waits for the slowest part of the stack: the device. Rough orders of magnitude, which vary widely by device and configuration:

| Device | Approximate `fsync` latency | Why |
|---|---|---|
| Hard disk | several milliseconds or more | Must wait for the platter to rotate under the head |
| Consumer SSD without power-loss protection | ~0.5 to several milliseconds | Must really program NAND before acknowledging the flush |
| Datacenter SSD with power-loss protection (capacitors) | tens of microseconds | Can acknowledge from its cache, which it can always save on power loss |
| Network block storage (cloud volumes) | ~0.5 to several milliseconds | Each flush is a round trip over the network to replicated storage |

If each committed transaction needs one `fsync` and each `fsync` takes 2 ms, a database committing transactions one at a time can commit at most 1,000 / 2 = **500 transactions per second**, no matter how fast its CPU is. Databases escape this ceiling with **group commit**: they gather the log records of many concurrent transactions and make them all durable with one `fsync`, so 50 transactions waiting together cost one 2 ms flush. They also append to a sequential **write-ahead log** and `fsync` only that, instead of syncing every changed data page. Both mechanisms are the subject of Chapter 15.

PostgreSQL ships a tool, `pg_test_fsync`, that measures how many syncs per second a volume sustains with each sync method; it is a quick way to find out which row of the table above your storage belongs to.

### The "fsync lie" and other risks

`fsync` returning 0 is a promise made by every layer to the one above it. Any layer can break it:

1. **Devices that ignore flush commands.** Some drives, controllers and virtual disk layers acknowledge a cache flush without actually making the data non-volatile, usually for benchmark speed. The OS does everything right and data is still lost on power failure. Hardware RAID controllers with a write cache are safe only if the cache is battery- or flash-backed.
2. **Platforms where `fsync` does less.** On macOS, `fsync` passes data to the drive but does not force the drive to flush its cache; full durability requires a separate `fcntl` operation, `F_FULLFSYNC`. Code tested on a laptop can behave differently from code on a Linux server.
3. **Lost errors ("fsyncgate").** In 2018, PostgreSQL developers found that on Linux, when writeback of a dirty page failed (for example, because of a transient storage error), the kernel reported the error once and could then mark the page clean. A later `fsync` retry returned success even though the data never reached the disk. PostgreSQL had been retrying failed `fsync` calls on the assumption that the data was still pending. It now treats an `fsync` failure as fatal (the server panics and recovers from its write-ahead log) rather than retrying. The general lesson: **after an `fsync` error, you cannot know which data is on disk**; the only safe response is to recover from a log you know is durable.
4. **Configurations that turn it off.** Many databases have a setting that disables or relaxes `fsync` for speed (PostgreSQL's `fsync` and `synchronous_commit` settings; MySQL InnoDB's `innodb_flush_log_at_trx_commit`). They are useful for throwaway test data and dangerous anywhere else.
5. **Virtualisation and containers.** A VM's disk is a file or network volume on the host; whether a guest's flush reaches stable storage depends on the hypervisor's cache settings.

### Guarantees and non-guarantees

On an honest storage stack, a successful `fsync` guarantees the file's data and the metadata needed to read it back survive a power failure. It does **not** guarantee the file's name is durable (sync the directory), that other files are durable, that the device is honest, that a retry after an error means anything, or that several files were made durable atomically. `write()` guarantees only visibility to other processes on the same machine.

### Failure and diagnosis

- **Data lost after power failure, although the application "saved" it.** The code called `write` and never `fsync`, or synced the file but not the directory. Evidence: `strace -f -e trace=fsync,fdatasync,write,rename -p <pid>` shows writes without syncs.
- **Slow commits / high write latency.** Database commit latency follows `fsync` latency. Evidence: `iostat -x` shows high `w_await` on the log device; the database's own statistics show long log-flush or WAL-sync times; `strace -T` shows `fsync` or `fdatasync` calls taking milliseconds. Common causes: a slow or throttled cloud volume (provisioned IOPS exhausted), a degraded disk, another process saturating the same device (Lab 4).
- **"PANIC: could not fsync file"** in a PostgreSQL log, followed by a restart: an I/O error during sync. Look at `dmesg` for the underlying device errors; the storage is failing.

### Backend connection

Every "the data is saved" message a backend gives a user rests on an `fsync` somewhere, usually inside the database. Knowing this lets you reason about three things. **Capacity:** commit throughput is bounded by sync latency divided across group commit. **Configuration:** settings that relax `fsync` trade durability for speed, and must be chosen deliberately. **Your own files:** if your service writes files that matter (uploads, exports, a local queue), it must use the write → `fsync` → rename → `fsync` directory sequence, or accept that a crash can lose or truncate them. Chapter 15 builds the database's write-ahead log on exactly these calls.

> **Java lens:** `FileChannel.force(true)` maps to `fsync` (data and metadata) and `force(false)` to roughly `fdatasync`; `FileOutputStream.getFD().sync()` is another route to `fsync`. `Files.move(tmp, target, StandardCopyOption.ATOMIC_MOVE)` performs a `rename`. Java has no direct API to sync a directory portably; on Linux, opening the directory as a `FileChannel` and calling `force` commonly works, but it is platform-dependent. Closing a stream does **not** sync it: `close()` only guarantees the bytes have been handed to the kernel.

---

## 4.5 I/O Models: Blocking, Non-Blocking, Multiplexing, and Asynchronous I/O

This is the most important section of the chapter for understanding server design. It answers one question: **when a thread asks for data that is not there yet, what happens to the thread?**

### Definitions

An **I/O model** is the contract between a program and the kernel about what a thread does while an I/O operation cannot yet complete, and how the program learns that it can proceed.

- **Blocking I/O:** the call does not return until it can make progress. If no data is available, the kernel puts the calling thread to sleep (blocked state, Chapter 3, Section 3.4) and wakes it when data arrives.
- **Non-blocking I/O:** the call never sleeps. If it cannot make progress immediately, it returns at once with the error `EAGAIN` (also called `EWOULDBLOCK`), meaning "nothing now, try later". An FD is put in this mode with the `O_NONBLOCK` flag.
- **I/O multiplexing** (also called **readiness notification**): one call waits on **many** FDs at once and returns when **at least one** of them is ready, telling the program which. The calls are `select`, `poll`, Linux's `epoll`, and BSD/macOS's `kqueue`.
- **Signal-driven I/O:** the kernel sends the process a signal (`SIGIO`) when an FD becomes ready.
- **Asynchronous I/O:** the program **submits** an operation ("read 8 KB from FD 9 into this buffer") and the call returns immediately; the kernel performs the whole operation, including the data copy, and later reports **completion**. On Linux the modern interface is `io_uring`.

Two distinctions organise the whole list:

- **Blocking vs non-blocking** is about whether **one call** can put the thread to sleep.
- **Synchronous vs asynchronous** is about **who performs the transfer and when**. In every model except true asynchronous I/O, the data copy from kernel to user buffer happens during the program's own `read()` call, synchronously. Multiplexing only tells you *when* a synchronous `read` will not block. This is the **readiness model** (`epoll`: "you can read now"). Asynchronous I/O is the **completion model** (`io_uring`: "your read is done; the bytes are in your buffer").

### Why it matters: the C10K problem

In 1999 Dan Kegel posed the **C10K problem**: how can one server handle ten thousand concurrent client connections? The obvious design, one thread per connection with blocking I/O, struggled for reasons Chapter 3 already gave: each thread needs a stack (Section 3.7), the kernel must schedule all of them, and with thousands of threads most CPU time goes to context switches and cold caches (Section 3.11). Most of those ten thousand connections are idle at any instant (a browser holding a keep-alive connection open, a mobile client on a slow network), so the server pays for ten thousand sleeping threads to do the work of perhaps fifty. The solution that emerged was a small number of threads, each multiplexing many non-blocking connections through a scalable readiness mechanism, `epoll` on Linux and `kqueue` on BSD. That design underlies Nginx, Redis, Node.js, Netty and many others.

### Model 1: blocking I/O

```pseudo
// one thread per connection
thread handle(conn_fd):
    loop:
        n = read(conn_fd, buf)      // sleeps until bytes arrive
        if n == 0: break             // client closed the connection
        request = parse(buf[0:n])    // (assume one read = one request, for now)
        write(conn_fd, respond(request))
    close(conn_fd)
```

When `read` finds the socket's receive buffer empty, the kernel adds the thread to the socket's **wait queue** (a list of threads waiting for that socket), marks it sleeping, and runs someone else. When a packet arrives, the network stack places the data in the receive buffer and wakes everything on the wait queue. The thread becomes runnable, is scheduled, copies the bytes and returns.

Blocking I/O is **simple**: the code reads top to bottom, and a stack trace shows exactly where each request is. It costs nothing while waiting except the memory of a sleeping thread. Its cost is **one thread per concurrent wait**.

### Model 2: non-blocking I/O with polling

```pseudo
// one thread, many connections, busy-polling
set O_NONBLOCK on every fd
loop forever:
    for fd in connections:
        n = read(fd, buf)
        if n == EAGAIN: continue     // nothing yet; try the next one
        if n == 0: close(fd); remove fd
        else: handle(fd, buf[0:n])
```

One thread can now serve many connections, but it never sleeps. With 10,000 idle connections, it makes 10,000 system calls per loop to learn that nothing has happened, using a full core to do no work. Adding a short `sleep` saves CPU but adds latency. Non-blocking mode on its own is not a solution; it is one half of one.

### Model 3: I/O multiplexing

The missing half is a call that **sleeps until something is ready** and then says **what**:

```pseudo
// one thread, many connections, readiness-driven
set O_NONBLOCK on every fd
loop forever:
    ready = wait_for_readiness(all fds)   // select / poll / epoll_wait: SLEEPS
    for fd in ready:
        n = read(fd, buf)                 // will not block: data is there
        ...
```

This is the core of an **event loop**: wait for readiness, handle each ready FD a little, loop. The programming model built on top of it (callbacks, futures, `async`/`await`) is the subject of Chapter 5; here we care about the system calls underneath.

### Concrete: three connections, three models

Three clients are connected. Client A sends a request at t=1 ms, C at t=3 ms, B at t=6 ms. Each takes 0.5 ms of CPU to handle.

```text
 time     │ BLOCKING (3 threads)          │ NON-BLOCKING POLLING (1 thread)  │ EPOLL EVENT LOOP (1 thread)
 ─────────┼───────────────────────────────┼──────────────────────────────────┼──────────────────────────────
 0 ms     │ thread-A, -B, -C each asleep  │ read(A)=EAGAIN read(B)=EAGAIN    │ epoll_wait({A,B,C}): asleep
          │ inside its own read()         │ read(C)=EAGAIN read(A)=EAGAIN …  │
 1 ms     │ A's bytes arrive → thread-A   │ next read(A) returns bytes →     │ epoll_wait returns {A} →
          │ wakes, handles A (0.5 ms),    │ handle A, then resume spinning   │ read(A), handle A (0.5 ms),
          │ calls read(A), sleeps again   │ through read(B), read(C), …      │ epoll_wait again: asleep
 3 ms     │ thread-C wakes, handles C,    │ read(C) returns bytes →          │ returns {C} → handle C,
          │ sleeps                        │ handle C, resume spinning        │ asleep
 6 ms     │ thread-B wakes, handles B,    │ read(B) returns bytes →          │ returns {B} → handle B,
          │ sleeps                        │ handle B, resume spinning        │ asleep
 ─────────┼───────────────────────────────┼──────────────────────────────────┼──────────────────────────────
 CPU      │ 3 × 0.5 ms of handling        │ 100% of a core for all 7 ms,     │ 3 × 0.5 ms of handling
          │                               │ almost all of it wasted          │ + 3 cheap wake-ups
 threads  │ 3 (3 stacks, 3 sleepers)      │ 1                                │ 1 (+ kernel interest list)
 syscalls │ 6 reads                       │ thousands of failed reads        │ 4 epoll_waits + 3 reads
```

The epoll loop has the efficiency of the blocking model (no wasted CPU) and the footprint of the polling model (one thread). Its price is that **no handler may block**. If the handler for A spends 200 ms on a blocking database call, B and C wait 200 ms too, because the only thread is busy. Event-loop servers therefore either use non-blocking clients for everything (database drivers, HTTP clients) or hand blocking and CPU-heavy work to a separate thread pool. Chapter 11 develops this into the execution models of real servers.

### select and poll: why they stop scaling

**`select`** takes three bitmaps (`fd_set`s): FDs to watch for reading, writing and exceptional conditions. On each call:

1. the program builds the bitmaps (they are overwritten on return, so they must be rebuilt every time);
2. the kernel copies them in, then **checks every FD up to the highest number**, asking each one "are you ready?" and adding the thread to each FD's wait queue;
3. if none is ready, the thread sleeps; when any FD becomes ready, the thread wakes and the kernel **removes it from every wait queue** and checks every FD again;
4. the kernel copies the bitmaps back, and the program **scans every bit** to find which are set.

Each call costs O(n) in the number of **watched** FDs, even if only one is ready. `select` also has a hard limit: the bitmap size, `FD_SETSIZE`, is 1,024 in common C libraries, so it cannot watch FD 1,024 or above at all.

**`poll`** replaces bitmaps with an array of `{fd, events, revents}` structures. That removes the 1,024 limit, but nothing else changes: the whole array is copied in and out and scanned in full on every call.

Put numbers on it. A server holds 10,000 connections; on a typical wake-up, 20 are ready. With `poll`, each loop iteration copies 10,000 entries in, checks 10,000 FDs, registers on and removes from 10,000 wait queues, copies 10,000 entries out and scans them, to find 20. If the loop runs 5,000 times a second under load, that is 50 million FD checks per second of pure overhead, growing with every idle connection.

### epoll: how it works

`epoll` splits the work into two phases: **registration**, done once per FD, and **waiting**, done in a loop.

```text
 epoll_create1()        → returns an FD for a new epoll instance (kernel object)
 epoll_ctl(epfd, ADD, fd, events)   → register interest in fd   (once per fd)
 epoll_ctl(epfd, MOD/DEL, fd, ...)  → change or remove interest
 epoll_wait(epfd, out[], max, timeout) → sleep until something is ready;
                                          return ONLY the ready fds
```

Inside the kernel, an epoll instance holds two structures:

```text
                     epoll instance (anon_inode:[eventpoll])
 ┌──────────────────────────────────────────────────────────────────────┐
 │ INTEREST LIST (red-black tree, keyed by file + fd)                   │
 │   fd 7  {EPOLLIN}      fd 8  {EPOLLIN}     ...     fd 10006 {EPOLLIN}│
 │   each entry has registered a CALLBACK on its socket's wait queue    │
 │                                                                      │
 │ READY LIST (linked list)                                             │
 │   ──▶ [fd 812] ──▶ [fd 4410]                                         │
 │                                                                      │
 │ WAITERS: threads sleeping in epoll_wait                              │
 └──────────────────────────────────────────────────────────────────────┘
        ▲                                                │
        │ 2. callback appends fd 812 to READY LIST       │ 3. wake a waiter;
        │    (if not already there)                      │    epoll_wait copies
 ┌──────┴────────┐                                       ▼    the ready entries out
 │ socket fd 812 │◀── 1. packet arrives; network    application gets [812, 4410]
 │ wait queue    │       stack wakes the socket's
 └───────────────┘       wait queue
```

1. `epoll_ctl(ADD)` inserts the FD into the **interest list**, a red-black tree, so adding, finding and removing an entry costs O(log n). It also hooks a small callback onto the socket's wait queue. This is done **once**, not on every wait.
2. When data arrives on socket 812, the network stack wakes the socket's wait queue as usual. One of the "waiters" is epoll's callback, which appends 812 to the **ready list**: O(1) work done at the moment of the event, by the code that caused it.
3. `epoll_wait` checks the ready list. If it is empty, the thread sleeps. If not, the kernel copies up to `max` ready entries to the program and returns their count.

So `epoll_wait` costs O(number of **ready** FDs), not O(number of **watched** FDs). In the example above, it returns 20 entries without looking at the 9,980 idle connections. Idle connections cost memory (an entry in the tree, a hook on a wait queue) but no CPU per loop. This is why `epoll` scales.

| | `select` | `poll` | `epoll` |
|---|---|---|---|
| Interest set stored | In user space; copied in on every call | In user space; copied in on every call | **In the kernel**, registered once |
| Cost per wait | O(watched) | O(watched) | **O(ready)** |
| FD limit | `FD_SETSIZE` (1,024 commonly) | None (beyond the process FD limit) | None (beyond the process FD limit) |
| Portability | Everywhere | POSIX systems | Linux only (`kqueue` on BSD and macOS, I/O completion ports on Windows) |
| Best for | A handful of FDs, portable code | A few hundred FDs | Thousands of mostly idle connections |

`kqueue`, on FreeBSD and macOS, follows the same design (kernel-held interest set, return only events) and also handles timers, signals and process events in the same call.

### Level-triggered versus edge-triggered

`epoll` can report readiness in two modes:

- **Level-triggered (LT, the default):** `epoll_wait` reports an FD **as long as** it is ready. If the socket has 10 KB buffered and you read only 4 KB, the next `epoll_wait` reports it again. Implementation: after reporting an LT entry, the kernel puts it back on the ready list if it is still ready.
- **Edge-triggered (ET, `EPOLLET`):** `epoll_wait` reports an FD when its state **changes** from not ready to ready (new data arrived). If you read only part of the data and wait again, you are **not** told again until *more* data arrives.

```text
 bytes in receive buffer:   0 ──▶ 10 KB ──(read 4 KB)──▶ 6 KB ──(no new data)──▶ 6 KB
 level-triggered reports:         ready                  ready                  ready ...
 edge-triggered reports:          ready                  (nothing)              (nothing)
                                                         └── 6 KB stranded until the
                                                             client sends more, which
                                                             it won't: it is waiting
                                                             for your response
```

The rule for edge-triggered mode is therefore: **on every notification, read (or write, or accept) in a loop until the call returns `EAGAIN`**. Forgetting this produces connections that hang forever with data sitting in the kernel, and the bug appears only under load, when more data arrives at once than one read takes. Edge-triggered mode exists because it wakes the program fewer times, and it is used by high-performance frameworks that drain FDs carefully. Level-triggered mode is more forgiving and is the usual choice for hand-written loops.

Two more flags solve problems you will meet when several threads share one epoll instance. `EPOLLONESHOT` disables an FD after one notification until it is re-armed, so two threads never handle the same connection at once. `EPOLLEXCLUSIVE` reduces the **thundering herd** problem, in which one new connection on a listening socket wakes *every* thread waiting on it, though only one can accept it; the others wake for nothing and go back to sleep.

### Readiness is not a request

Two subtleties catch everyone once:

1. **Ready means "a call will not block", not "a whole message is here".** An HTTP request of 3 KB can arrive as two TCP segments. The first `read` returns 1,460 bytes; the request is incomplete. The program must keep a **per-connection buffer** and parse incrementally, resuming when more bytes arrive. TCP is a byte stream with no message boundaries (Chapter 6); message framing is the application protocol's job (Chapter 7).
2. **Regular files are always "ready".** Readiness is meaningful for sockets, pipes and terminals, where data arrives from outside. A disk file's data is always "available" in this sense, even if reading it will block for 10 ms on the device. `epoll` refuses to register regular files at all (`epoll_ctl` returns `EPERM`), and `select`/`poll` always report them ready. Event-loop runtimes therefore perform file I/O on a separate thread pool (Node.js does this through its libuv library), or use true asynchronous I/O.

An event loop also has to watch **writes**: if a client reads its response slowly, the socket's send buffer fills up and `write` returns `EAGAIN` after a partial write. The loop must keep the unsent remainder and register interest in `EPOLLOUT` (writable) for that FD until it drains.

### Model 4: signal-driven I/O

With signal-driven I/O, a program sets an FD's owner and the `O_ASYNC` flag, and the kernel sends `SIGIO` when the FD becomes ready. The handler runs asynchronously, in the middle of whatever the program was doing (Chapter 3, Section 3.6), which makes it awkward to use safely, and a signal says little about *which* FD is ready when there are many. It survives mainly in older code and is rarely used in servers. Its existence is mostly interesting as the answer to "what other I/O models are there?"

### Model 5: asynchronous I/O and io_uring

In every readiness model, the program still makes one system call per operation, and the copy happens during that call. For a server doing hundreds of thousands of small operations per second, the per-call cost (Chapter 3, Section 3.2) becomes significant, and readiness does not help with disk files at all.

**Asynchronous I/O** inverts the contract: describe the operation, let the kernel do it, collect the result. Linux had two earlier attempts. POSIX AIO in the GNU C library is implemented with user-space threads that make blocking calls. The older native Linux AIO interface (`io_submit`) is genuinely asynchronous only for `O_DIRECT` file I/O and can still block in some cases. Neither became a general solution.

**`io_uring`** (added in Linux 5.1, 2019) is built around two ring buffers in memory **shared between the program and the kernel**:

```text
            user space                               kernel
 ┌──────────────────────────────┐
 │ SUBMISSION QUEUE (SQ)        │  program writes entries:
 │ [read fd=9 buf=A len=8K]     │  "read", "write", "accept",
 │ [write fd=12 buf=B len=2K]   │  "send", "fsync", "openat", ...
 │ [fsync fd=5]                 │──── io_uring_enter() ───▶ kernel consumes SQ,
 └──────────────────────────────┘    (one call for many       performs each operation
                                      ops, or none at all      (async), and writes a
 ┌──────────────────────────────┐     in SQPOLL mode)          result per op to CQ
 │ COMPLETION QUEUE (CQ)        │◀────────────────────────────────────┘
 │ [op#1 result=8192]           │  program reads results:
 │ [op#3 result=0]              │  bytes transferred, or -errno
 └──────────────────────────────┘
```

- The program puts **submission queue entries** describing operations into the SQ and tells the kernel with one `io_uring_enter` call, which can submit hundreds of operations at once. In **SQPOLL** mode a kernel thread watches the SQ continuously, so submitting needs no system call at all, at the cost of that thread using CPU.
- The kernel performs each operation and writes a **completion queue entry** with its result. The program reads completions from shared memory, again without a system call, or waits for them with `io_uring_enter`.
- It works for files (buffered and `O_DIRECT`) and for sockets, and supports many other operations, so one interface covers everything.

Its benefits are fewer system calls (batching), true asynchrony for disk I/O, and less copying in some modes. Its costs are complexity, a large new kernel interface, and a history of security vulnerabilities; some container runtimes' default security profiles block its system calls, and some organisations disable it on production systems.

### The five models side by side

| Model | Thread sleeps? | Program learns... | Who copies data, and when | Typical users |
|---|---|---|---|---|
| Blocking | Yes, in each `read` | Implicitly, when `read` returns | Kernel, during `read` | Thread-per-connection servers, most classic database drivers |
| Non-blocking polling | Never | By asking repeatedly | Kernel, during `read` | Almost nobody alone; a building block |
| Multiplexing (`select`, `poll`, `epoll`, `kqueue`) | Yes, in the *wait* call, for many FDs | Which FDs are ready | Kernel, during the later `read` | Nginx, Redis, Node.js, Netty |
| Signal-driven | No | Via `SIGIO` | Kernel, during the later `read` | Rare |
| Asynchronous (`io_uring`; Windows I/O completion ports) | Optional, waiting for completions | That the operation completed | Kernel, before notifying | High-performance storage and network engines |

### Guarantees and non-guarantees

`epoll` guarantees that a reported FD could make progress at the moment it was reported. It does **not** guarantee it still can (another thread may have read the data), that a complete message is available, or anything at all about regular files. Edge-triggered mode does not repeat notifications for data you left unread. Non-blocking mode guarantees a call never sleeps; it does not guarantee the call does anything. `io_uring` guarantees a completion entry for each submitted operation; it does not guarantee completion order matches submission order unless you ask for linked operations.

### Failure and diagnosis

- **Event loop blocked:** latency of *all* requests on a node rises together, often in steps equal to the duration of some slow operation, while CPU is low. A handler is making a blocking call (a synchronous DNS lookup, a blocking database driver, a large file read) or doing heavy computation on the loop thread. Evidence: a stack dump of the loop thread shows it inside a blocking call; event-loop lag metrics (the delay between when a timer should fire and when it does) spike. Fix: move the work to a worker pool.
- **Busy loop at 100% CPU with no traffic:** often a level-triggered FD that is permanently ready and never handled, such as a closed connection still registered (always readable, `read` returns 0), or a listening socket that cannot be accepted because the FD limit is reached (Section 4.9). `strace -c -p <pid>` shows millions of `epoll_wait` calls returning immediately.
- **Hung connections under load only:** edge-triggered mode without draining to `EAGAIN`.
- **`select` crashes or corrupts memory above 1,024 FDs:** an old library using `select` in a process that now has more FDs. The fix is to use `poll` or `epoll`.

### Backend connection

The I/O model decides the shape of a server: thread-per-connection (simple, limited by threads) or event loop (scales to many idle connections, but every handler must avoid blocking). It also decides what a slow dependency does to you: in a blocking model a slow database ties up threads; in an event loop it ties up nothing until you accidentally call a blocking driver from the loop. Chapter 11 turns this into a decision procedure.

> **Java lens:** `java.nio.channels.Selector` is the JDK's multiplexing API; on Linux it is implemented with `epoll`, on macOS with `kqueue`. Netty can use either the JDK selector or its own native `epoll` transport. Classic `java.io` streams and JDBC drivers are blocking. Java's virtual threads (Java 21 and later) let you write blocking-style code while the runtime parks the virtual thread on blocking socket operations and uses an internal `epoll`-based poller to resume it; file I/O is handled differently, typically by temporarily adding a carrier thread. Virtual threads reduce the cost of *waiting*; they add no CPU, database connections or downstream capacity.
>
> **Other runtimes:** Node.js runs JavaScript on a single event-loop thread (libuv, using `epoll`/`kqueue`) and performs file system operations and DNS lookups on a small thread pool. Go uses a "netpoller" built on `epoll`/`kqueue` underneath goroutines, so goroutine code looks blocking while the runtime multiplexes. Python's `asyncio` uses `selectors`, which picks `epoll` on Linux.

---

## 4.6 Sockets as File Descriptors

### Definitions

A **socket** is a kernel object that is one endpoint of a communication channel, accessed through a file descriptor. A TCP socket is an endpoint of a reliable, ordered byte stream between two processes, usually on different machines.

A **listening socket** is a socket that does not carry data; it accepts new connections on an address and port. A **connected socket** (or connection socket) carries the data of one connection and is identified by the **4-tuple** (source IP, source port, destination IP, destination port).

The **backlog** is the queue of connections that the kernel has fully established on a listening socket but the application has not yet taken with `accept()`.

(A **port** is a 16-bit number that identifies an endpoint within one IP address, Chapter 1. IP addresses, ports, the TCP handshake and `TIME_WAIT` are taught in Chapter 6. Here we need only the system calls and the kernel's queues.)

### Why it matters

Every HTTP request your server ever receives arrives through these calls. The queues between them explain a family of failures where clients "connect" but nothing happens, and the 4-tuple explains why one port can serve a million clients.

### Mechanism: the server and client system call sequences

```text
        SERVER                                   CLIENT
 fd=socket(AF_INET, SOCK_STREAM)          c=socket(AF_INET, SOCK_STREAM)
   └ new socket, no address
 bind(fd, 0.0.0.0:8080)
   └ claim the port (EADDRINUSE if taken)
 listen(fd, backlog=4096)
   └ socket becomes LISTENING; kernel
     now answers handshakes by itself
                                          connect(c, 10.0.0.5:8080)
                                            └ kernel picks an ephemeral source
                                              port (e.g. 51432), sends SYN
                ◀──────────── SYN ─────────────
 kernel: create a half-open entry
   in the SYN QUEUE, reply SYN-ACK
                ───────────── SYN-ACK ────────▶
                                          kernel: connection ESTABLISHED;
                                          connect() returns 0
                ◀──────────── ACK ─────────────
 kernel: move entry from SYN queue to
   ACCEPT QUEUE (fully established);
   listening fd becomes "readable"
                                          write(c, "GET / HTTP/1.1...")
                ◀──────────── data ────────────   (bytes wait in the server's
                                                   receive buffer)
 conn=accept(fd)
   └ remove one entry from the accept
     queue, create a NEW socket and a
     NEW fd (e.g. 9) for it
 read(conn, buf)  → "GET / HTTP/1.1..."
 write(conn, "HTTP/1.1 200 OK...")  ─────────▶ read(c, buf)
 close(conn)                                    close(c)
```

The key facts in this sequence:

- **The kernel completes the TCP handshake without the application.** Once `listen` has been called, connections are established and queued even if the application never calls `accept`, for example because all its threads are busy.
- **There are two queues.** The **SYN queue** holds connections that have received a SYN and are waiting for the client's final ACK (half-open). The **accept queue** holds fully established connections waiting for `accept()`. On Linux, the `backlog` argument of `listen` sets the size of the accept queue, capped by the system setting `net.core.somaxconn`, whose default differs between kernel versions. How many half-open connections the SYN queue may hold depends on the same backlog and on the setting `net.ipv4.tcp_max_syn_backlog`; the exact rules have changed between kernel versions.
- **`accept` returns a new FD.** The listening socket keeps listening; each connection gets its own socket and FD, and its own receive and send buffers. That is why one listening port serves many clients: all connections share the destination port 8080 but each has a distinct 4-tuple.
- **The client needs no `bind`.** `connect` makes the kernel choose a free local **ephemeral port**. Each outbound connection uses one, which is why a client making many short connections to the same destination can exhaust them (Chapter 6).

```text
 listening socket 0.0.0.0:8080 (fd 3)
     │
     ├── SYN queue (half-open)      [10.0.4.2:40112] [10.0.7.1:39021]
     │                                    ↓ final ACK arrives
     ├── accept queue (established) [10.0.1.9:51432] [10.0.3.3:60110] ... ← backlog limit
     │                                    ↓ accept()
     ▼
 connected sockets:  fd 9  (10.0.0.5:8080 ↔ 10.0.1.9:51432)
                     fd 10 (10.0.0.5:8080 ↔ 10.0.2.8:45013)
                     ...  each with its own receive buffer and send buffer
```

### Concrete: a server's startup in `strace` and `ss`

```text
$ strace -f -e trace=socket,setsockopt,bind,listen,accept4 ./server
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_TCP) = 3
setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(3, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(3, 4096)                         = 0
accept4(3, {sa_family=AF_INET, sin_port=htons(51432), sin_addr=inet_addr("10.0.1.9")}, [16], SOCK_CLOEXEC|SOCK_NONBLOCK) = 9
```

`accept4` is the Linux variant of `accept` that can set close-on-exec and non-blocking mode on the new FD in the same call, which event-loop servers want.

`ss` (Section 4.12) shows listening sockets with their queue state. **For a socket in `LISTEN` state, `Recv-Q` is the current number of connections in the accept queue and `Send-Q` is the queue's maximum:**

```text
$ ss -ltn
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port
LISTEN   0        4096             0.0.0.0:8080          0.0.0.0:*
LISTEN   129      128            127.0.0.1:9000          0.0.0.0:*
```

Port 8080 is healthy: nothing waiting. Port 9000 has an accept queue that is **full**: 129 connections waiting with a limit of 128 (Linux allows the count to reach the limit plus one). The application on port 9000 is not calling `accept` fast enough.

### Binding addresses and the `SO_REUSE*` options

- **Bind address:** `0.0.0.0` means "all IPv4 interfaces of this machine (or of this network namespace, Section 4.10)". `127.0.0.1` means "loopback only": reachable only from the same machine (or the same network namespace). A server inside a container that binds to `127.0.0.1` cannot be reached from outside the container, a very common first-deploy bug.
- **`SO_REUSEADDR`** lets a restarted server bind its port even though connections from the previous run are still in `TIME_WAIT` (Chapter 6). Servers typically set it; without it, a quick restart can fail with "Address already in use".
- **`SO_REUSEPORT`** lets several sockets, typically one per worker process or thread, bind the **same** port; the kernel then distributes incoming connections among them. This spreads accept work across cores and avoids the thundering herd on a single shared listening socket.

### Socket buffers and flow control

Each connected socket has a **receive buffer** (bytes that arrived and have not been read) and a **send buffer** (bytes written by the application that TCP has not yet had acknowledged by the peer). `write` on a socket returns once the bytes are copied into the send buffer, not when the peer receives them. When the send buffer is full because the peer is slow or not reading, a blocking `write` sleeps and a non-blocking `write` returns `EAGAIN` or writes only part of the data. This is how TCP flow control (Chapter 6) reaches the application: a slow reader makes the writer wait.

The same pattern as `write()` and the disk (Section 4.4) appears here: **`write()` on a socket returning success means "the kernel has the bytes", not "the other side has them"**, and not "the other side processed them".

### Guarantees and non-guarantees

The socket API guarantees that `accept` returns established connections in a usable state and that each has its own FD and buffers. It does **not** guarantee that a connection in the accept queue is still alive when you accept it (the client may have given up), that `connect` succeeding means the server application has seen the connection (it may sit in the accept queue for seconds), or that `write` succeeding means delivery.

### Failure and diagnosis

- **Accept queue overflow.** Under a traffic burst, or when the application stops calling `accept` (all workers busy, event loop blocked), the accept queue fills. On Linux, by default, the kernel then ignores the client's final handshake ACK, so the connection stays half-open and the server retransmits its SYN-ACK; new SYNs may be dropped. **The client's `connect` usually succeeds** (from its side the handshake looked complete), it sends its request, and then waits for a response that comes late or never. The symptom is client timeouts with **no corresponding log line on the server**, because the application never saw the connection. Evidence: `ss -ltn` shows `Recv-Q` at the `Send-Q` limit; `nstat -az TcpExtListenOverflows TcpExtListenDrops` (or `netstat -s`, "times the listen queue of a socket overflowed") shows counters rising. (A related message, "Possible SYN flooding on port ...", appears in `dmesg` when the *SYN queue* of half-open connections fills, typically under a flood of SYNs, and the kernel falls back to SYN cookies or drops SYNs; Chapter 6.) Fix the cause (why is the application not accepting?), and size the backlog for bursts.
- **"Address already in use" (`EADDRINUSE`) on startup:** another process holds the port, or an old instance is still running, or a child process inherited the listening socket. `ss -ltnp 'sport = :8080'` shows the owner.
- **"Connection refused" (`ECONNREFUSED`) on the client:** nothing is listening on that address and port, so the server's kernel answered the SYN with a reset. Often the server binds the wrong interface (`127.0.0.1` inside a container) or the wrong port.
- **`EPIPE` / "Broken pipe", or "Connection reset by peer" (`ECONNRESET`):** writing to a connection the peer has closed or reset. In a server, usually a client that gave up before the response was ready. Normally harmless individually; a spike means clients are timing out on you.

### Backend connection

Every load balancer, reverse proxy, application server and database (Chapter 10) runs this sequence. The accept queue is the first queue a request waits in, before any application-level queue, and the only one invisible to application metrics. When latency rises and the application's own request timers look fine, check whether requests are waiting in the kernel before the application ever sees them.

---

## 4.7 Zero-Copy and Buffering

### Definitions

**Buffering** is collecting many small reads or writes in a user-space memory area and transferring them in fewer, larger system calls.

**Zero-copy** is a family of techniques that move data between kernel objects (for example, from a file to a socket) without copying it through a user-space buffer. On Linux the main calls are `sendfile` (file to socket), `splice` (between a pipe and another FD), and `mmap` (Chapter 3, Section 3.15) combined with `write`.

### Why it matters

Each system call has a fixed cost (Chapter 3, Section 3.2), and each copy of a buffer costs CPU time and memory bandwidth and evicts useful data from CPU caches (Chapter 2, Section 2.3). A server that writes a log line with one `write` per line, or a file server that copies every byte of every file through its own memory, spends CPU on moving bytes rather than on serving requests.

### Mechanism: buffering

Suppose a program writes one million log lines of 100 bytes each. Unbuffered, that is **one million `write` calls**. With a 64 KB user-space buffer that is flushed when full, it is about 100 MB / 64 KB ≈ **1,600 calls**. If each call costs about 1 µs of overhead, buffering saves roughly a second of CPU time.

The price is a new place where data can be lost. Data moves through four places before it is durable, and each is lost on a different kind of failure:

| Where the bytes are | Lost if... | Moved onward by |
|---|---|---|
| User-space buffer (in the process) | **the process crashes or is killed**, including `SIGKILL` and OOM kills | `flush` → `write()` |
| Kernel page cache | the kernel crashes or power fails | writeback or `fsync` (Section 4.4) |
| Device's volatile write cache | power fails (unless the device has power-loss protection) | the device's flush, triggered by `fsync` |
| Non-volatile media | the device fails | (replication, backups: Chapters 15 and 22) |

This is why the last log lines before a crash are often missing: they were in the process's buffer, not the kernel's. A process killed by `SIGKILL` (Chapter 3, Section 3.6) gets no chance to flush.

### Mechanism: the traditional copy path versus `sendfile`

A static file server sending a 1 MB file the traditional way:

```pseudo
fd = open("video.mp4")
loop:
    n = read(fd, buf, 64K)     // copy: page cache → user buffer
    write(sock, buf, n)        // copy: user buffer → socket send buffer
```

```text
 TRADITIONAL read() + write()                    SENDFILE
                                                 
 disk ──DMA──▶ page cache                         disk ──DMA──▶ page cache
                 │ CPU copy (read)                                │ CPU copy, or none with
                 ▼                                                │ NIC scatter-gather:
             user buffer                                          │ socket buffer just
                 │ CPU copy (write)                               │ references the pages
                 ▼                                                ▼
          socket send buffer                              socket send buffer
                 │ DMA                                            │ DMA
                 ▼                                                ▼
                NIC                                              NIC

 2 DMA copies + 2 CPU copies                    2 DMA copies + 1 or 0 CPU copies
 2 system calls per chunk                       1 system call for the whole file
 = 4 user/kernel mode switches per chunk        = 2 mode switches
 data passes through CPU caches twice           data need not enter user space at all
```

(**DMA**, direct memory access, is the hardware mechanism by which a device copies data to or from RAM without the CPU doing the copying.)

`sendfile(sock, file_fd, offset, count)` tells the kernel to send `count` bytes of the file directly to the socket. The bytes go from the page cache to the socket without visiting user space. If the network card supports gathering data from scattered memory pages, the kernel can even avoid the remaining CPU copy by handing the card references to the page cache pages.

`splice` moves data between an FD and a pipe without copying through user space; a proxy can use it to move bytes from one socket to another through a pipe. `mmap` plus `write` removes one copy (the file is mapped, so `write` copies straight from the page cache to the socket), but page-table setup and page faults have their own cost.

### Where zero-copy does not apply

- **TLS.** Bytes sent over HTTPS (Chapter 8) must be encrypted, so they must pass through code that encrypts them, normally in user space. Zero-copy file serving is lost unless the encryption is done in the kernel (Linux's **kernel TLS**, kTLS) or by the network card.
- **Transformation.** If the server compresses, templates or otherwise changes the bytes, they must be in user space.
- **Small payloads.** For a 2 KB JSON response, the copy is negligible next to everything else.

### Guarantees and non-guarantees

`sendfile` guarantees the same result as `read` plus `write` with less work. It does **not** guarantee that all requested bytes are sent in one call (on a non-blocking socket it may send part and return the count, like `write`), and zero-copy does not make the network faster; it frees CPU. Buffering guarantees fewer system calls, not that buffered data survives a crash.

### Failure and diagnosis

- **Missing last log lines after a crash or kill:** they were in a user-space buffer when the process died (`SIGKILL` and OOM kills give it no chance to flush). Evidence: the orchestrator reports exit code 137 and the log ends mid-burst. Fix: flush on a short timer and on shutdown, and write critical events unbuffered.
- **High kernel CPU from tiny writes:** a process writes one small piece at a time without buffering. `strace -c -p <pid>` shows millions of `write` calls averaging a few dozen bytes; `top` shows a high `sy` (kernel) share. Fix: buffer.
- **A file server or proxy that is CPU-bound on large transfers:** profiles show time in memory copying and in `read`/`write`, not in application logic. Check whether the zero-copy path is enabled and applicable (TLS or compression disables it).
- **Truncated downloads:** code calls `sendfile` (or `write`) once and assumes everything was sent. On a non-blocking socket it can return a partial count; the rest must be sent when the socket is writable again. Evidence: byte counts in access logs smaller than the files' sizes.

### Backend connection

- **Static file serving and reverse proxies.** Nginx serves static files with `sendfile` when its `sendfile` option is enabled. A proxy forwarding a large download between two sockets can use `splice`.
- **Message brokers.** Kafka (Chapter 25) stores messages in files in a format it can send to consumers unchanged, which lets it use `sendfile` from the page cache to consumer sockets when TLS is not in use.
- **Your own code.** Streaming a file from disk to an HTTP response in chunks is better than loading it into one byte array (which costs memory proportional to the file size, Chapter 2, Section 2.6) and usually lets the framework use a zero-copy path. Buffer log output, but understand what a crash loses.

> **Java lens:** `FileChannel.transferTo(position, count, socketChannel)` uses `sendfile` on Linux where possible. `BufferedOutputStream` and `BufferedWriter` are user-space buffers: data in them is lost if the JVM is killed before `flush()`.

---

## 4.8 Inter-Process Communication

### Definition

**Inter-process communication (IPC)** is any mechanism that lets separate processes exchange data or coordinate. Because each process has its own address space (Chapter 3, Section 3.13), one process cannot simply read another's variables; the kernel must provide a channel. The main mechanisms are **pipes**, **named pipes (FIFOs)**, **Unix domain sockets**, **shared memory**, **message queues**, **signals**, and network sockets.

### Why it exists

Splitting work into separate processes brings isolation: a crash in one does not corrupt the memory of another, each can run as a different user with different permissions, and each can be written in a different language. Processes still need to cooperate: a shell pipeline passes data between programs; PostgreSQL's per-connection backend processes (Chapter 3, Section 3.9) must share one cache of database pages; a web server talks to a local database; a supervisor tells its workers to reload configuration. IPC is the set of doors the kernel puts in the walls between processes.

### Pipes

A **pipe** is a one-way byte stream inside the kernel, with a write end and a read end, each an FD. `pipe(fds)` creates one; the usual pattern is to create it and then `fork`, so the parent and child each keep one end.

```text
  $ grep ERROR app.log | sort

  grep (fd 1) ──write──▶ ┌─────────────────────────┐ ──read──▶ (fd 0) sort
                         │ kernel pipe buffer      │
                         │ (on Linux, 64 KB by     │
                         │  default)               │
                         └─────────────────────────┘
  buffer full  → writer blocks (backpressure: grep waits for sort)
  buffer empty → reader blocks
  all write ends closed → reader's read() returns 0 (EOF)
  all read ends closed  → writer gets SIGPIPE, or EPIPE if SIGPIPE is ignored
```

The shell creates the pipe, forks twice, and in each child uses `dup2` to put the pipe ends at FD 1 and FD 0 before `exec` (Section 4.1). Writes of up to `PIPE_BUF` bytes (4,096 on Linux) are atomic: concurrent writers' small messages are never interleaved mid-message. Pipes are anonymous: only related processes (those that inherited the FDs) can use them.

**`SIGPIPE` in servers.** Writing to a pipe *or socket* whose reader has gone delivers `SIGPIPE`, whose default action is to terminate the process (Chapter 3, Section 3.6). A server that does not ignore `SIGPIPE` can be killed by a client disconnecting mid-response. Server runtimes ignore it or use `send` with a flag that suppresses it, and handle the resulting `EPIPE` error instead.

### Named pipes (FIFOs)

A **named pipe** or **FIFO** is a pipe with a name in the filesystem, created with `mkfifo`. Unrelated processes open the path, one for reading and one for writing, and then use it like a pipe. Data never touches the disk; the filesystem entry is just a meeting point. FIFOs are simple and one-way, and they are mostly seen in scripts and older tools.

### Unix domain sockets

A **Unix domain socket** (UDS) uses the socket API (Section 4.6) with the address family `AF_UNIX` and a **filesystem path** instead of an IP address and port, for example `/var/run/postgresql/.s.PGSQL.5432`. It supports stream (like TCP) and datagram modes, and it is bidirectional.

Compared with TCP over the loopback interface on the same machine, a Unix domain socket:
- skips the network stack (no TCP segments, checksums, congestion control or routing), so it typically has lower latency and higher throughput;
- is **access-controlled with file permissions** on the socket path, so only processes running as permitted users can connect;
- lets the server learn the connecting process's PID, user and group from the kernel (the `SO_PEERCRED` option on Linux), which databases use for password-less local authentication;
- can **pass open file descriptors** between processes (a message with `SCM_RIGHTS`): the kernel installs a reference to the same open file description in the receiver's FD table. Zero-downtime restart schemes use this to hand a listening socket from an old server process to a new one.

Common backend uses: local connections to PostgreSQL, MySQL or Redis; the Docker daemon's control socket (`/var/run/docker.sock`); communication between a web server and an application server, or between an application and a sidecar proxy, on the same host.

### Shared memory

**Shared memory** is a region of physical memory mapped into the address spaces of several processes, so that a write by one is immediately visible to the others. It is created with `shm_open` plus `mmap`, an anonymous shared `mmap` inherited across `fork`, or the older System V calls (`shmget`). Once set up, communication involves **no system calls and no copies**: it is the fastest IPC.

Its cost is that the kernel provides **no coordination**. If two processes update the same structure at once, they corrupt it. Shared memory must be combined with synchronisation (locks, semaphores or atomic operations, Chapter 5), and those must work across processes. And because every process can write the shared region, one process crashing in the middle of an update can leave it inconsistent for all the others.

PostgreSQL is the standard example. Its shared buffer pool, lock tables and other shared state live in shared memory mapped into every backend process. Because a crashing backend might have left that memory corrupt, when one backend process crashes, the PostgreSQL supervisor process terminates all other backends, reinitialises shared memory and runs crash recovery, so every client connection is dropped, not just the one that crashed.

### Message queues

A **message queue** (POSIX `mq_open`, or System V `msgget`) is a kernel-held queue of **discrete messages**, each with a size and, for POSIX queues, a priority. Unlike a pipe, which is a byte stream, a reader receives whole messages, one at a time. These kernel message queues are local to one machine and are rarely used in web backends. They should not be confused with **message brokers** such as RabbitMQ or Kafka, which are network services with persistence, routing and delivery guarantees (Chapter 25).

### Signals as IPC

A signal (Chapter 3, Section 3.6) carries essentially one number. It is suitable for **control** ("reload your configuration", "shut down", "dump your thread stacks"), not for data. Standard signals of the same type can merge if several arrive before the first is handled.

### Comparison

| Mechanism | Direction | Unit | Scope | Speed | Coordination provided | Typical use |
|---|---|---|---|---|---|---|
| Pipe | One-way | Byte stream | Related processes | Fast (one kernel copy each way) | Blocking read/write, EOF | Shell pipelines, parent ↔ child |
| FIFO | One-way | Byte stream | Same machine, by path | Like a pipe | Like a pipe | Scripts, simple tools |
| Unix domain socket | Two-way | Stream or datagram | Same machine, by path | Faster than loopback TCP | Socket semantics, permissions, peer credentials, FD passing | Local DB connections, sidecars, daemons' control sockets |
| Shared memory | Any | Raw memory | Same machine | Fastest (no copies, no syscalls) | **None**: you add locks | Databases' shared caches, high-rate data exchange |
| Message queue | One-way per queue | Messages with priority | Same machine | Fast | Message boundaries, blocking | Rare in web backends |
| Signal | One-way | A signal number | Same machine | Fast | Almost none | Shutdown, reload, debug |
| TCP/UDP socket | Two-way | Stream / datagram | **Any machines** | Slowest (network stack, and the network) | Socket semantics; TCP reliability (Chapter 6) | Everything between services |

### Guarantees and non-guarantees

A pipe guarantees in-order bytes, blocking when full or empty, EOF when every write end is closed, and atomicity for writes of up to `PIPE_BUF` bytes; it does **not** preserve message boundaries, and unrelated processes cannot find it. A Unix domain socket guarantees socket semantics and permission checks on its path; it does not work across machines. Shared memory guarantees that writes become visible to the other processes; it guarantees **no** coordination, and no consistency after a process crashes mid-update, which you must build yourself (Chapter 5). Kernel message queues guarantee whole messages; they do not survive a reboot. Signals guarantee delivery of a signal number; they do not guarantee that two identical standard signals are both delivered.

### Why services communicate over the network instead

Within one host, pipes, Unix sockets and shared memory are faster than TCP. Yet services in a backend system almost always talk over TCP, using HTTP or gRPC (Chapters 7 and 13). The reasons:

1. **Location independence.** A service that talks TCP works the same whether the other service is on the same host, the next rack, or another region. Deployments move services between machines constantly (Chapter 26); local IPC would tie them together.
2. **Independent scaling and deployment.** Each service can run any number of instances behind a load balancer (Chapter 10); shared memory requires processes on one machine with compatible memory layouts.
3. **Language and version independence.** A network protocol is a stable contract of bytes; shared memory exposes internal data structures.
4. **Failure isolation.** A crash on the other side of a socket shows up as a connection error; a crash in a process sharing your memory can corrupt your data.

The cost is real: a call over the network is slower than a function call by several orders of magnitude, requires serialisation, and can fail in ways a function call cannot, including an **unknown outcome** when it times out (Chapter 1, Section 1.2). Where co-location is guaranteed, as with a sidecar proxy in the same pod or a local database, Unix domain sockets or loopback TCP are common compromises.

### Failure and diagnosis

- **Pipeline or child process stuck:** a parent writes to a child's stdin pipe while the child is blocked writing to a stdout pipe the parent is not reading; both buffers fill and both wait forever. This is a deadlock (Chapter 5) between two processes. `strace -p` on each shows one in `write` and the other in `write`. Fix: read and write concurrently, or use files.
- **Stale Unix socket file:** a server crashed and left its socket path behind; on restart, `bind` fails with "Address already in use". Servers usually remove the old path at startup.
- **Permission denied connecting to a Unix socket:** the socket file's permissions or owner exclude the client's user; `ls -l` the path.
- **All database connections dropped at once** on PostgreSQL after a single backend crash: the shared-memory reset described above. The server log shows a backend terminated by a signal, followed by "terminating any other active server processes".

### Backend connection

IPC choices show up in performance tuning (a local database over a Unix socket instead of TCP), in security (who can reach `/var/run/docker.sock` effectively controls the host), and in failure analysis (why one crashed PostgreSQL backend disconnected everyone). The broader lesson is that a service boundary is an IPC boundary, and the network is the IPC mechanism you choose when the processes may be anywhere.

---

## 4.9 Resource Limits and Exhaustion

### Definitions

A **resource limit** (**rlimit**) is a per-process ceiling, enforced by the kernel, on how much of a resource the process may use: open file descriptors, processes and threads, stack size, core dump size, CPU time and others. Each limit has a **soft limit**, the value actually enforced, and a **hard limit**, the ceiling up to which an unprivileged process may raise its own soft limit. `ulimit` is the shell command that shows and sets the limits of the shell and of the programs it starts; the underlying system calls are `getrlimit` and `setrlimit` (or `prlimit`).

**Resource exhaustion** is a failure in which an operation fails not because of a bug in the operation, but because some finite resource it needs (FDs, threads, ports, memory, disk) has run out.

### Why limits exist

A machine's resources are finite and shared. Without limits, one buggy process leaking file descriptors, or a "fork bomb" creating processes in a loop, could consume a kernel table that every other process needs, and take down the whole host. Rlimits make the damage stay with the process that caused it. They are an old, per-process mechanism; cgroups (Section 4.10) are the newer, per-group mechanism and are what containers rely on.

### Mechanism: the limits that matter to servers

| Limit (rlimit name) | `ulimit` flag | What it caps | What happens when it is reached |
|---|---|---|---|
| `RLIMIT_NOFILE` ("nofile") | `-n` | The highest FD number + 1, effectively the number of open FDs per process | `open`, `socket`, `accept`, `pipe` fail with `EMFILE`, "Too many open files" |
| `RLIMIT_NPROC` ("nproc") | `-u` | Processes **and threads** owned by the same user ID, counted across the whole system | `fork` / thread creation fails with `EAGAIN`, "Resource temporarily unavailable" |
| `RLIMIT_STACK` | `-s` | The main thread's stack size (Chapter 2, Section 2.6) | Stack overflow → `SIGSEGV` |
| `RLIMIT_CORE` | `-c` | Size of the core dump written when a process crashes | Core dump truncated or not written (0 disables them) |
| `RLIMIT_AS` | `-v` | Total virtual address space | `mmap`/`malloc` fail; runtimes with large reservations may not even start |

There are also **system-wide** limits: `fs.file-max` caps the total number of open file descriptions in the whole kernel (reaching it gives `ENFILE`, "Too many open files in system", for every process), and `fs.nr_open` caps how high any process's `nofile` hard limit can be set.

Rlimits are **inherited** across `fork` and `exec`. So the limits that apply to your server are those of whatever started it, which is the source of a classic confusion:

- Limits set in `/etc/security/limits.conf` are applied by the login system to **user login sessions**. They do **not** apply to services started by systemd.
- A systemd service gets its limits from its unit file (`LimitNOFILE=`, `LimitNPROC=`).
- A container gets its limits from the container runtime's configuration (for example `docker run --ulimit nofile=65536:65536`) or its defaults.

The only reliable way to know the limits of a running process is to ask the kernel:

```text
$ cat /proc/4100/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max processes             63432                63432                processes
Max open files            1024                 524288               files
...
```

This process has a soft limit of 1,024 open files, very low for a server, although its hard limit would allow it to raise the limit itself to 524,288. Many runtimes raise their own soft limit to the hard limit at startup; others do not. The low soft default survives on many systems for compatibility: a program using `select` (Section 4.5) breaks if it ever receives an FD above 1,023.

### Concrete: sizing `nofile`

A server is expected to hold 20,000 client connections. Each needs one FD. It also has a database pool of 50 connections, an outbound HTTP pool of up to 200 connections, about 100 open files (libraries, logs, configuration), and a few `epoll` instances and pipes. That is about 20,400 FDs at the expected peak. A limit of 65,536 gives headroom for bursts and slow leaks; a limit of 1,024 fails at about 2% of the target load.

### Guarantees and non-guarantees

The kernel guarantees that a process cannot exceed its soft limits, that only a privileged process can raise a hard limit, and that limits are inherited by child processes. It does **not** guarantee that the limits are the ones you configured (they come from whatever started the process: a login session, a systemd unit, or a container runtime), it gives **no warning** as a process approaches a limit (the first sign is a failed call), and a limit reserves nothing: a limit of 65,536 FDs does not mean the system-wide table or memory can supply them.

### Failure and diagnosis: "Too many open files"

When a process hits `nofile`, everything that needs a new FD fails at once:

```text
accept4(3, ...) = -1 EMFILE (Too many open files)
openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = -1 EMFILE (Too many open files)
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_TCP) = -1 EMFILE (Too many open files)
```

The server cannot accept clients, cannot connect to its database, cannot even read `/etc/hosts` to resolve a hostname, so the errors in the logs are a confusing mix of "connection failed", "DNS resolution failed" and "Too many open files".

A particularly bad interaction with level-triggered `epoll`: the listening socket stays readable (connections keep arriving in the accept queue), `accept` keeps failing with `EMFILE`, and the event loop spins at 100% CPU calling `epoll_wait` and `accept` in a loop. Robust servers back off for a moment when `accept` fails with `EMFILE`, or keep one spare FD open that they can close, accept-and-immediately-close one pending connection, and reopen, so clients get a prompt failure instead of waiting in the queue.

**Diagnosis procedure:**

1. Confirm: `ls /proc/<pid>/fd | wc -l` against "Max open files" in `/proc/<pid>/limits`.
2. Classify: `lsof -p <pid>` (Section 4.12) and count by type. Thousands of `REG` entries for the same file means a file leak; thousands of TCP sockets to the same remote address suggests a connection leak; many sockets in `CLOSE_WAIT` (the peer has closed, but this process never called `close`; TCP states are in Chapter 6) is the classic signature of code that forgets to close connections.
3. Trend: sample the count over time. A count that rises with traffic and falls back when traffic falls means the limit is too low for the load; a count that only ever rises means a leak.

### Other exhaustion failures that look similar

| Symptom | Resource | Where to look | Home |
|---|---|---|---|
| "Too many open files" | Per-process FDs (`nofile`) | `/proc/<pid>/fd`, `/proc/<pid>/limits`, `lsof` | This section |
| "Too many open files in system" | System-wide `fs.file-max` | `/proc/sys/fs/file-nr` | This section |
| "Resource temporarily unavailable" on thread or process creation | `nproc`, or the cgroup's `pids.max` | `/proc/<pid>/status` (Threads), `pids.current` in the cgroup | This section, Section 4.10 |
| "Cannot assign requested address" on `connect` | Ephemeral ports to one destination | `ss -tan state time-wait` counts | Chapter 6 |
| "Timeout waiting for a connection from the pool" | The application's database connection pool | Pool metrics | Chapter 18 |
| "No space left on device" | Disk blocks or inodes | `df -h`, `df -i` | Section 4.2 |

The last three rows are **different** resources. A connection pool can be exhausted while the process has thousands of FDs to spare, and FDs can be exhausted while the pool is fine. Name the exact resource before choosing a fix.

**Port exhaustion preview.** A client process making many short-lived outbound connections to the same destination address and port needs a distinct local ephemeral port for each, and a closed connection keeps its port reserved for a while in the `TIME_WAIT` state. On many Linux systems the default ephemeral range covers about 28,000 ports (see `net.ipv4.ip_local_port_range`), so a service opening and closing connections at a few hundred per second to one database or API can run out. The cure is to reuse connections (keep-alive and pooling, Chapters 7 and 18). The mechanism is in Chapter 6.

### Backend connection

Set `nofile` explicitly for every server (in the unit file, the container runtime, or both), at a value well above the expected peak, and verify it in `/proc/<pid>/limits` after deployment. Export the open-FD count as a metric and alert on its ratio to the limit. A steady upward trend is a leak to find before it becomes an outage.

> **Java lens:** "java.io.IOException: Too many open files" and "java.net.SocketException: Too many open files" are `EMFILE`. "java.lang.OutOfMemoryError: unable to create native thread" usually means the OS refused to create a thread (the `nproc` limit, the cgroup's `pids.max`, or a lack of memory for the stack); the Java heap may be nearly empty. The HotSpot JVM on Linux raises its soft `nofile` limit to the hard limit at startup by default, so the hard limit is the one to check.

---

## 4.10 Isolation: Virtual Machines and Containers

### Definitions

**Isolation** is preventing one workload from seeing, modifying, or exhausting the resources of another workload on the same hardware.

A **virtual machine (VM)** is an emulated computer, with its own virtual CPUs, memory, disks and network cards, on which a complete **guest operating system**, including its own kernel, runs. A **hypervisor** (or virtual machine monitor) is the software that creates and runs VMs and divides the real hardware among them.

A **container** is **an ordinary process (or group of processes) on the host's Linux kernel**, started with **namespaces** that restrict what it can see, **cgroups** that limit what it can use, a **root filesystem** taken from an image, and reduced privileges. There is no guest kernel.

A **namespace** is a kernel feature that gives a process its own private view of one kind of global resource (process IDs, network interfaces, mount points, hostname, and so on).

A **control group (cgroup)** is a kernel feature that groups processes and **limits and accounts** for the CPU, memory, number of processes and I/O the group uses.

A **container image** is a packaged root filesystem, made of stacked read-only **layers**, plus configuration (the program to run, environment variables, the user to run as).

### Why isolation exists

A server rarely runs only one thing. Without isolation:

- **Dependency conflicts:** service A needs version 1 of a system library and service B needs version 2.
- **Noisy neighbours:** one runaway batch job uses all the CPU or memory and slows or kills everything else (Chapter 3, Sections 3.12 and 3.17).
- **Security:** a compromised process can read other processes' files and memory, or see their network traffic.
- **Reproducibility:** "it works on my machine" because the production host's files differ from the developer's.

VMs solve all four by giving each workload a whole simulated machine. Containers solve the same problems more cheaply, but less completely, by giving each workload a fenced-off **view** of one shared kernel.

### Mechanism: virtual machines

```text
  TYPE 1 ("bare-metal")                     TYPE 2 ("hosted")
 ┌────────┬────────┬────────┐             ┌────────┬────────┐
 │ App    │ App    │ App    │             │ App    │ App    │
 │ libs   │ libs   │ libs   │             │ libs   │ libs   │ ┌───────────┐
 │ GUEST  │ GUEST  │ GUEST  │             │ GUEST  │ GUEST  │ │ host apps │
 │ KERNEL │ KERNEL │ KERNEL │             │ KERNEL │ KERNEL │ │ (browser) │
 ├────────┴────────┴────────┤             ├────────┴────────┴─┴───────────┤
 │ HYPERVISOR               │             │ HYPERVISOR (an application)   │
 │ (runs on the hardware)   │             ├───────────────────────────────┤
 ├──────────────────────────┤             │ HOST OPERATING SYSTEM         │
 │ HARDWARE (VT-x / AMD-V)  │             ├───────────────────────────────┤
 └──────────────────────────┘             │ HARDWARE                      │
                                          └───────────────────────────────┘
  e.g. Xen, VMware ESXi, Hyper-V;          e.g. VirtualBox, VMware Workstation
  cloud servers                            developer laptops
```

A **type 1** hypervisor runs directly on the hardware and is used in data centres and clouds. A **type 2** hypervisor runs as an application on a normal operating system and is used on desktops. Linux's **KVM** blurs the line: it is a kernel module that turns Linux itself into a hypervisor, so a Linux host runs VMs and ordinary processes side by side. It is usually classed as type 1.

The problem a hypervisor must solve is that a guest kernel expects to run in kernel mode (Chapter 3, Section 3.2) and control the hardware: page tables, interrupts, devices. It cannot be allowed to. Modern CPUs provide **hardware virtualisation extensions** (Intel VT-x, AMD-V) that add a separate mode for guests:

1. **CPU.** The guest kernel runs in a "guest" mode in which most instructions, including most kernel-mode ones, run directly on the CPU at full speed. Sensitive operations (accessing certain registers, some I/O, halting the CPU) cause a **VM exit**: the CPU traps to the hypervisor, which emulates the operation and resumes the guest. VM exits are expensive, so performance depends on how rarely they happen.
2. **Memory.** The guest manages its own page tables, translating guest-virtual to guest-"physical" addresses, but those guest-physical addresses are themselves virtual from the host's point of view. **Second-level address translation** (Intel EPT, AMD NPT) lets the MMU (Chapter 3, Section 3.13) walk both levels of page tables in hardware. A TLB miss (Chapter 3, Section 3.14) is more expensive than on bare metal, because a two-dimensional walk touches more page-table entries; huge pages help more inside VMs.
3. **Devices.** A hypervisor can emulate a real device (slow: every register access traps), provide a **paravirtual** device designed for virtualisation, such as virtio network and disk devices, where the guest's driver knows it is virtualised and exchanges batches of requests with the hypervisor through shared memory, or pass a real device (or a slice of one) through to the guest.

The result: each VM has its own kernel, so a bug or compromise in one guest kernel does not directly affect another; the attacker must also break the hypervisor, a much smaller and more carefully guarded piece of code. The costs: each VM carries a whole OS (hundreds of MB of memory, a boot that takes seconds or more), and every I/O crosses an extra layer. **Steal time** (Chapter 3, Section 3.12) is the guest's view of the hypervisor running someone else on "its" CPU.

### Mechanism: a container is a process

A container runtime (such as runc, which Docker and Kubernetes typically use underneath) starts a container roughly like this:

```pseudo
create a cgroup for the container; write its limits (cpu.max, memory.max, pids.max)
child = clone(flags = NEW_PID | NEW_NET | NEW_MNT | NEW_UTS | NEW_IPC [| NEW_USER])
in child:
    join the cgroup
    mount the image's layers as one root filesystem (overlay, below)
    switch the root directory to it            // pivot_root
    mount a fresh /proc (so ps sees only this PID namespace)
    set the hostname                            // UTS namespace
    drop capabilities; install a seccomp filter // restrict privileged operations
    switch to the configured user
    exec(entrypoint)                            // e.g. /app/server, becomes PID 1
```

There is no emulated hardware and no second kernel. The "container" is the result: a process whose system calls go to the **same host kernel** as everyone else's, but which sees a restricted world and is limited in what it can consume. From the host, it is just another process:

```text
 on the HOST:                               inside the CONTAINER:
 $ ps -eo pid,user,comm | grep server       $ ps -e
   31877 10001 server                           PID TTY          TIME CMD
                                                  1 ?        00:00:04 server
                                                 27 pts/0    00:00:00 ps
```

Same process, two PIDs: 31,877 in the host's PID namespace and 1 in the container's.

### Namespaces

Linux provides eight namespace types. A process belongs to exactly one namespace of each type; `ls -l /proc/<pid>/ns` shows which.

| Namespace | Isolates | Effect inside the container | Backend consequence |
|---|---|---|---|
| **pid** | Process IDs | Sees only its own processes; the first is PID 1 | PID 1 duties: signals and reaping (Chapter 3, Section 3.5; Section 4.11) |
| **net** | Network interfaces, IP addresses, routing tables, ports, firewall rules | Own loopback `lo`, own `eth0`, own port space | Two containers can both listen on port 8080; `127.0.0.1` inside is not the host's loopback |
| **mnt** | Mount points (the filesystem tree) | Own root filesystem from the image | Host files are invisible unless explicitly mounted in (volumes) |
| **uts** | Hostname and domain name | Own hostname | Logs and metrics carry the container's hostname |
| **ipc** | System V IPC objects, POSIX message queues | Cannot see other containers' shared memory segments or queues | Shared-memory IPC (Section 4.8) works only within the container unless the namespace is shared |
| **user** | User and group IDs | UID 0 ("root") inside can map to an unprivileged UID outside | "Rootless" containers; reduces the damage of a container escape |
| **cgroup** | The view of the cgroup hierarchy | Sees its own cgroup as the root | Tools inside read their own limits at `/sys/fs/cgroup` |
| **time** | Some system clocks (monotonic and boot time offsets) | Can have shifted clocks | Rarely used by servers |

**Networking, briefly.** A new network namespace contains only a loopback interface. The runtime creates a **virtual ethernet pair** (veth), a virtual cable with one end inside the container (seen as `eth0`) and the other attached to a software bridge on the host. Publishing a container port ("map host port 8080 to container port 8080") is done with network address translation rules on the host (NAT, Chapter 6). Containers in the same Kubernetes **pod** (the orchestrator's smallest deployable unit: one or more containers always scheduled together on one host; Chapter 26) **share** one network namespace, which is why they can reach each other on `localhost`.

**Try it yourself** (requires root on a Linux machine): `unshare` runs a program in new namespaces without any container runtime.

```text
$ sudo unshare --pid --fork --mount-proc --uts bash
# hostname sandbox
# hostname
sandbox
# ps -e
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
      8 pts/0    00:00:00 ps
```

This shell is PID 1 in its own world and has its own hostname, yet it runs on the host kernel and, without a separate mount namespace and root filesystem, sees the host's files. A container is this idea, plus a filesystem, cgroups and privilege restrictions.

### cgroups

Namespaces control what a process can **see**. cgroups control what it can **use**. In cgroup version 2 (the current interface), cgroups form one tree of directories under `/sys/fs/cgroup`. Each directory is a group; a process belongs to one group; limits are set and usage is read by reading and writing files in the directory. **Controllers** implement each resource:

| Controller | Key files | Meaning |
|---|---|---|
| cpu | `cpu.max` = "*quota* *period*" | The group may use at most *quota* microseconds of CPU time per *period*, summed over all its threads and all cores. `50000 100000` = half a CPU. Exceeding it means **throttling**: every thread in the group waits until the next period (Chapter 3, Section 3.12) |
| | `cpu.weight` | Relative share **when CPUs are contended**; no effect on an idle machine |
| | `cpu.stat` | `nr_periods`, `nr_throttled`, `throttled_usec`: the evidence of throttling |
| memory | `memory.max` | Hard limit. Above it, the kernel reclaims the group's memory (dropping its clean page cache, swapping if allowed); if that fails, the **OOM killer runs inside the group** and kills one of its processes |
| | `memory.high` | Soft limit: above it, the group's allocations are slowed and reclaim is forced, but nothing is killed |
| | `memory.current`, `memory.stat`, `memory.events` | Usage now; breakdown (`anon` vs `file` cache); counters of `max` hits and `oom_kill` events |
| pids | `pids.max` | Maximum number of processes **and threads** in the group; above it, `fork` and thread creation fail with `EAGAIN` |
| io | `io.max`, `io.weight` | Per-device bandwidth and IOPS limits, or relative shares |

```text
$ cat /sys/fs/cgroup/cpu.max
50000 100000
$ cat /sys/fs/cgroup/cpu.stat
usage_usec 81234567
user_usec 70123456
system_usec 11111111
nr_periods 36000
nr_throttled 4100
throttled_usec 302345678
$ cat /sys/fs/cgroup/memory.events
low 0
high 0
max 312
oom 3
oom_kill 3
```

(Paths shown are as seen from inside a container with its own cgroup namespace; on the host, the same files are deeper in the tree.) This group is limited to half a CPU; it was throttled in 4,100 of 36,000 periods (11%), accumulating about 302 seconds of throttled time; it hit its memory limit 312 times and had a process OOM-killed 3 times.

Two cgroup behaviours surprise people:

1. **The memory limit counts more than your heap.** It counts all the group's anonymous memory (heap, stacks, native buffers, Chapter 3, Section 3.18), kernel memory used on its behalf, and the **page cache** of files it reads and writes (Section 4.3). Clean cache is reclaimed before anything is killed, so a high `memory.current` is not alarming by itself; look at `anon` in `memory.stat`.
2. **The OOM kill is local.** The host may have 100 GB free; a container at its 2 GB `memory.max` still has a process killed. The process dies with `SIGKILL`, exit code 137 (Chapter 3, Section 3.6), and `dmesg` shows "Memory cgroup out of memory".

**What the process sees.** Namespaces do not virtualise everything. Inside a container, `/proc/meminfo` and `free` typically show the **host's** memory, and the number of CPUs reported by many tools is the **host's** core count, not the cgroup's quota. A runtime that sizes its heap from "total memory" or its thread pools from "number of CPUs" without reading the cgroup files will size them for a machine it does not have.

**Orchestrators use the same files.** In Kubernetes, a container's CPU **request** becomes its `cpu.weight` (its share under contention), its CPU **limit** becomes `cpu.max`, and its memory **limit** becomes `memory.max`. Orchestration itself is in Chapter 26.

> **Java lens:** Modern JVMs are container-aware: they read the cgroup limits to decide the default heap size and the value of `Runtime.availableProcessors()`, which in turn sizes the default garbage collector threads and the common fork-join pool. `-XX:MaxRAMPercentage` sets the heap as a percentage of the container's memory limit; leaving headroom for non-heap memory (Chapter 3, Section 3.18) avoids OOM kills.

### Container images and layered filesystems

An image is a stack of read-only layers. Each layer is an archive of the files added, changed or deleted by one build step. When a container starts, the runtime combines the layers with a thin **writable layer** on top, using a **union filesystem**, typically Linux's **overlayfs**:

```text
  container's view (merged): /app/server  /app/config.yml  /etc/os-release  /tmp/x
          ▲
 ┌────────┴──────────────────────────────────────────────────────────────┐
 │ upper (writable, per container): /tmp/x, modified /app/config.yml,    │
 │                                   "whiteout" marking /etc/motd deleted │
 ├───────────────────────────────────────────────────────────────────────┤
 │ layer 3 (read-only): /app/server, /app/config.yml   ← "COPY app/"     │
 │ layer 2 (read-only): runtime libraries              ← "install deps"  │
 │ layer 1 (read-only): base OS files /etc, /usr, /bin ← base image      │
 └───────────────────────────────────────────────────────────────────────┘
```

- **Reading** a file finds the topmost layer that contains it.
- **Modifying** a file from a lower layer first **copies it up** into the writable layer (the first write to a large file is therefore slow), then changes the copy.
- **Deleting** a lower-layer file creates a **whiteout** entry in the upper layer that hides it.
- Layers are identified by a cryptographic hash of their contents, so ten containers from the same image, and different images with the same base, **share** the read-only layers on disk and in the page cache.
- The writable layer **disappears when the container is removed**. Anything that must survive (database files, uploads) belongs on a **volume**, a host directory or network disk mounted into the container. Write-heavy data should be on a volume anyway, because overlay copy-up and lookups add overhead.

### Security implications of a shared kernel

All containers on a host make system calls into **one kernel**. That kernel's attack surface, several hundred system calls and many drivers and filesystems, is the wall between containers. A kernel vulnerability exploitable from inside one container can give an attacker control of the host and therefore of every container on it. A VM's wall is the hypervisor's much narrower interface.

Container runtimes reduce the risk with several layers of restriction:
- **Capabilities.** Linux splits root's powers into about forty capabilities (for example, binding ports below 1024, changing file ownership, loading kernel modules, administering the network). Containers run with a reduced set by default.
- **seccomp filters** block system calls the container should never need.
- **Linux security modules** (AppArmor, SELinux) restrict file and resource access.
- **User namespaces** make "root" inside the container an unprivileged user outside it.

Common configurations defeat these protections:
- Running as root **without** a user namespace: UID 0 in the container is UID 0 on the host kernel, limited only by capabilities and filters.
- **"Privileged" containers**, which get all capabilities and device access: effectively root on the host.
- **Mounting the container runtime's control socket** (such as `/var/run/docker.sock`) into a container: whoever can talk to it can start a privileged container, so this is equivalent to host root.
- **Sharing host namespaces** (the host's network or PID namespace) or mounting sensitive host directories.
- **No limits:** without `pids.max` or `memory.max`, one container's fork bomb or leak exhausts the host for everyone.

When untrusted code must run (multi-tenant platforms, user-submitted code), stronger isolation is used: **sandboxed runtimes** such as gVisor, which intercepts the container's system calls with a user-space kernel, or **micro-VMs** such as Firecracker and Kata Containers, which run each container inside a minimal, fast-booting VM.

### Containers versus VMs

```text
           VIRTUAL MACHINES                              CONTAINERS
 ┌──────────┐ ┌──────────┐ ┌──────────┐     ┌──────────┐ ┌──────────┐ ┌──────────┐
 │ App A    │ │ App B    │ │ App C    │     │ App A    │ │ App B    │ │ App C    │
 │ libs     │ │ libs     │ │ libs     │     │ libs     │ │ libs     │ │ libs     │
 │ guest OS │ │ guest OS │ │ guest OS │     │ (image   │ │ (image   │ │ (image   │
 │ + KERNEL │ │ + KERNEL │ │ + KERNEL │     │  rootfs) │ │  rootfs) │ │  rootfs) │
 ├──────────┴─┴──────────┴─┴──────────┤     ├──────────┴─┴──────────┴─┴──────────┤
 │ virtual hardware (vCPU, vRAM, virtio)│   │ namespaces: pid net mnt uts ipc user│
 ├──────────────────────────────────────┤   │ cgroups:    cpu memory pids io      │
 │ HYPERVISOR                           │   │ seccomp, capabilities, LSM          │
 ├──────────────────────────────────────┤   ├──────────────────────────────────────┤
 │ HARDWARE                             │   │ ONE SHARED HOST KERNEL               │
 └──────────────────────────────────────┘   ├──────────────────────────────────────┤
                                            │ HARDWARE (or a VM)                   │
                                            └──────────────────────────────────────┘
```

| Aspect | Virtual machine | Container |
|---|---|---|
| What it is | An emulated computer running its own kernel | A host process with namespaces, cgroups and an image filesystem |
| Kernel | One per VM (any OS: Linux, Windows) | Shared with the host (Linux containers need a Linux kernel) |
| Isolation boundary | Hypervisor and hardware virtualisation: narrow, strong | The kernel's system call interface: wide, weaker |
| Start-up time | Seconds to minutes (boot an OS); micro-VMs much less | Typically well under a second (start a process) |
| Memory overhead | A whole guest OS per VM | Only the application's own processes |
| Density per host | Tens | Hundreds or more |
| CPU and memory performance | Near native, with extra cost on VM exits and TLB misses | Native (it is a native process), plus cgroup accounting |
| I/O performance | Through virtual devices (virtio): some overhead | Native, plus overlay filesystem and virtual network costs |
| Resource limits | Fixed virtual CPUs and RAM | cgroups: quotas, weights, memory limits |
| Image size | Gigabytes (a full disk) | Megabytes to hundreds of MB, shared layers |
| Typical use | Cloud instances, multi-tenant isolation, different OSes | Packaging and running services, often *inside* cloud VMs |

In practice the two are layered: cloud providers isolate customers from each other with VMs, and each customer packs many containers into each VM.

### Guarantees and non-guarantees

Namespaces guarantee a process cannot *name* resources outside its namespaces. cgroups guarantee a group cannot use more than its CPU quota per period, its memory maximum (without being OOM-killed), or its PID maximum. Neither guarantees **security against kernel vulnerabilities**, isolation of resources that are not namespaced or limited (the kernel's own memory pools, CPU caches and memory bandwidth, shared disk throughput without `io` limits), or that the application inside sees its limits through the usual interfaces (`/proc/meminfo`, CPU counts). A VM guarantees a separate kernel; it does not guarantee freedom from noisy neighbours (steal time, shared caches) or from hardware-level side channels.

### Failure and diagnosis

- **Container OOM-killed, exit 137, with no error in its logs:** `memory.events` shows `oom_kill` incremented; `dmesg` shows "Memory cgroup out of memory: Killed process ..."; the orchestrator reports `OOMKilled`. Compare `memory.stat`'s `anon` with the limit; non-heap memory is the usual culprit.
- **High latency with low CPU usage, under a CPU limit:** throttling. `cpu.stat`'s `nr_throttled` and `throttled_usec` rise with the latency spikes (Lab 2).
- **"Resource temporarily unavailable" creating threads in a container:** `pids.current` near `pids.max`.
- **Works locally, unreachable in a container:** the server binds `127.0.0.1` in its own network namespace (Section 4.6).
- **Data gone after a container restart:** it was written to the writable layer, not a volume.
- **Slow first write to a large file** in a container: overlay copy-up of the file from a lower layer.

### Backend connection

For a backend engineer, "the container" is the unit that is deployed, scaled, limited and killed. Understanding that it is a process with namespaces and cgroups explains why limits must be read from cgroup files, why the OOM killer can fire on a nearly empty host, why throttling looks like latency, why `localhost` means different things in different places, why running as root in a container is dangerous, and why a multi-tenant platform may wrap containers in VMs.

---

## 4.11 Running a Server Process Well in a Container

### Definition

Running a server process well means that the process **starts as the platform expects** (as the process that receives signals), **shuts down without failing requests it has accepted**, **leaves nothing behind** (no zombies, no half-written files), and **reports through the channels the platform reads** (exit codes, standard output and standard error).

### Why it matters

Chapter 3 introduced the two pieces this section depends on: PID 1's special duties (it reaps orphaned processes, and the kernel does not apply default signal actions to it, Section 3.5) and graceful shutdown on `SIGTERM` (Section 3.6). Containers make both unavoidable: every container has a PID 1, and every deploy, scale-down and node drain stops containers with `SIGTERM`, followed by `SIGKILL` after a grace period. If the process is wired up wrongly, every deploy fails some requests.

### Mechanism: who is PID 1?

The process the image's entrypoint starts becomes PID 1 in the container's PID namespace. There are three common shapes:

```text
 (a) EXEC FORM, server is PID 1        (b) SHELL FORM / wrapper script        (c) WITH A MINIMAL INIT
     ["/app/server"]                       /bin/sh -c "/app/server"               ["tini","--","/app/server"]

     PID 1  server                         PID 1  sh                              PID 1  tini
            ▲                                 │   ▲                                  │   ▲
   SIGTERM ─┘ delivered directly              └─▶ PID 7 server                       └─▶ PID 7 server
   (needs a handler: no default    SIGTERM ─────┘ goes to sh, which              SIGTERM ─┘ received by tini
    action for PID 1)                          may not forward it; server         and FORWARDED to server;
                                               never hears it; SIGKILL            tini also reaps zombies
                                               after the grace period
```

- **(a)** The server is PID 1 and receives signals directly. It must install a `SIGTERM` handler (most server runtimes do), and it must reap any child processes it starts.
- **(b)** A shell is PID 1. Whether a shell forwards `SIGTERM` to its child depends on the shell and on how the command was written; often it does not, and the server is killed with `SIGKILL` at the end of the grace period, with in-flight requests cut off. Container build files distinguish the "exec form" of the start command (a JSON list, run directly) from the "shell form" (a string, run through `/bin/sh -c`). Wrapper scripts that must do setup should end with `exec /app/server "$@"`: `exec` replaces the shell with the server (Chapter 3, Section 3.5), so the server becomes PID 1.
- **(c)** A tiny init program (such as `tini`, which `docker run --init` adds) is PID 1. It forwards signals to the server and reaps zombies. This is the safe choice when the server starts child processes it does not reap.

**Logs go to FD 1 and FD 2.** Container platforms capture each container's standard output and standard error (Section 4.1) and ship them to the logging system. Writing logs to files inside the container fills the writable layer (and possibly the host disk), and the files vanish with the container.

### Mechanism: the graceful shutdown sequence

When the platform decides to stop an instance, two things happen **at about the same time**: it sends `SIGTERM` to PID 1, and it removes the instance from the load balancer's list of targets. The removal propagates through the load balancers over a few seconds. So **requests can keep arriving after `SIGTERM`**. Load balancers decide where to send traffic partly through a **readiness check**: a periodic request (often to an endpoint such as `/ready`) whose failure means "do not send me new traffic"; health checks are covered in Chapter 24. A server that closes its listening socket the instant it receives `SIGTERM` makes those requests fail with "connection refused".

```text
 grace period (Kubernetes default: 30 s at the time of writing) ───────────────────────────▶ SIGKILL
 t=0s     SIGTERM ─▶ handler sets "shutting down"; readiness check starts failing
          platform starts removing the instance from load-balancer targets
 t=0–5s   KEEP ACCEPTING and serving: load balancers may still route new requests here
 t=5s     stop accepting: close the listening socket (Section 4.6), stop consuming
          from job queues (Chapter 25)
 t=5–25s  DRAIN in-flight requests, up to a deadline
          • mark responses "Connection: close" so keep-alive clients reconnect elsewhere
            (HTTP/2 has a GOAWAY frame for the same purpose; Chapter 7)
          • let background jobs finish or return them to their queue
 t=25s    deadline reached: abandon what is left (and log it)
 t=25s    close connection pools (database, HTTP clients), flush buffered logs and metrics
          (user-space buffers die with the process, Section 4.7)
 t=26s    exit(0)
 t=30s    ── if still alive: SIGKILL, exit code 137 ──
```

```pseudo
on SIGTERM:
    shutting_down = true                   // readiness endpoint now returns "not ready"
    sleep(PROPAGATION_DELAY)               // e.g. 5 s; still serving normally
    close(listen_fd)                       // no new connections
    wait until in_flight == 0 or DRAIN_DEADLINE passed
    close(db_pool); close(http_clients)
    flush(logs); flush(metrics)
    exit(0)
```

The numbers must fit together: propagation delay + longest acceptable drain + cleanup must be **less** than the grace period, or the platform's `SIGKILL` arrives first. Requests that may legitimately take longer than the drain window (large uploads, long-running reports) need a different design, such as asynchronous jobs (Chapter 13 and Chapter 25). The full protocol, including how load balancers and health checks take part, is in Chapter 24.

### Guarantees and non-guarantees

The platform guarantees `SIGTERM` first and `SIGKILL` after the grace period. It does **not** guarantee that load balancers have stopped sending traffic when `SIGTERM` arrives, that your process receives the signal (if PID 1 is a shell that does not forward it), or that anything in user-space buffers is flushed if `SIGKILL` arrives first.

### Failure and diagnosis

- **Every deploy produces a burst of 502 or 503 errors** at the load balancer: the server stops accepting or exits immediately on `SIGTERM`. Compare error timestamps with pod termination events; check whether the server logs a shutdown sequence at all.
- **Containers take exactly the grace period to stop, then exit with 137:** `SIGTERM` is not reaching the server, or the server ignores it. Check PID 1 inside the container (`ps -o pid,comm` from a shell in the container, or `cat /proc/1/cmdline`): if it is `sh`, use the exec form or `exec` in the script.
- **Zombie processes accumulating in a container** (`<defunct>` in `ps`): PID 1 does not reap; add an init.
- **Missing last log lines after a crash or kill:** they were in a user-space buffer.

### Backend connection

A correct shutdown sequence is what makes rolling deploys and autoscaling invisible to users. It depends on everything in this chapter: signals reaching the right process (PID 1), the listening socket and accept queue (Section 4.6), buffers that must be flushed (Section 4.7), and the platform's cgroup and namespace machinery (Section 4.10).

> **Framework lens:** Spring Boot's `server.shutdown=graceful` makes the embedded server stop accepting new requests on shutdown and wait for in-flight requests up to `spring.lifecycle.timeout-per-shutdown-phase`. It does not add the initial propagation delay; on Kubernetes that is commonly added with a `preStop` hook that sleeps for a few seconds before `SIGTERM` is sent.

---

## 4.12 Diagnosis Toolkit: lsof, strace, iostat, ss, dmesg

This section is a field guide to the five tools named in the chapter's labs, with what each output means. Output formats vary somewhat between versions and distributions; the columns shown are the common ones.

### `lsof`: what does this process have open?

`lsof` ("list open files") reads `/proc` and prints one line per FD.

```text
$ lsof -p 4100
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
server  4100  app  cwd    DIR  259,2     4096      2 /
server  4100  app    1w   REG  259,2  8812031    523 /var/log/app/app.log
server  4100  app    3u  IPv4  48100      0t0    TCP *:8080 (LISTEN)
server  4100  app    9u  IPv4  48213      0t0    TCP 10.0.0.5:8080->10.0.1.9:51432 (ESTABLISHED)
server  4100  app   47u  IPv4  51990      0t0    TCP 10.0.0.5:44120->10.0.9.20:5432 (CLOSE_WAIT)
server  4100  app   48r   REG  259,2    20480 917223 /srv/tmp/upload-8812.part (deleted)
```

- `FD`: the number plus mode (`r` read, `w` write, `u` both); `cwd` is the working directory.
- `TYPE`: `REG` regular file, `DIR`, `IPv4`/`IPv6` sockets, `unix`, `FIFO`.
- `NAME`: the path or the connection with its TCP state. `CLOSE_WAIT` means the remote side closed and this process has not.
- `(deleted)`: the file was unlinked but is still open (Section 4.2).

Useful forms: `lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn` (count FDs by type), `lsof -i :8080` (who uses port 8080), `lsof +L1` (deleted but open files, system-wide). On a host with many processes, `lsof` without filters is slow; scope it.

### `strace`: what system calls is it making?

`strace` attaches to a process with the kernel's debugging interface (`ptrace`) and prints every system call, its arguments and its result.

```text
$ strace -f -tt -T -e trace=read,write,fsync,epoll_wait -p 4100
[pid  4117] 10:41:07.120331 epoll_wait(5, [{events=EPOLLIN, data={u32=9, u64=9}}], 1024, 1000) = 1 <0.004133>
[pid  4117] 10:41:07.124530 read(9, "POST /api/orders HTTP/1.1\r\nHost"..., 65536) = 412 <0.000011>
[pid  4121] 10:41:07.125002 write(14, "INSERT INTO order_log ...", 182) = 182 <0.000009>
[pid  4121] 10:41:07.125040 fsync(14)  = 0 <0.047815>
```

`-f` follows all threads (`[pid N]` is the thread ID), `-tt` adds timestamps, `-T` shows the time spent in each call in angle brackets. Here an `fsync` took 48 ms, which explains a slow request. `strace -c -p <pid>` (stop with Ctrl-C) prints a summary table instead: for each system call, its count, error count and total time, which is the quickest way to see "this process makes 2 million `epoll_wait` calls a second" or "it spends 80% of its system call time in `fsync`".

**Caution:** `strace` stops the traced process at every system call, which can slow a busy process by a large factor. Use it briefly, with `-e` filters, and prefer lower-overhead tracers (`perf trace`, eBPF-based tools such as `bpftrace`) on production hosts under load.

### `iostat`: is the disk the bottleneck?

`iostat -x 1` (from the sysstat package) prints per-device statistics every second:

```text
Device   r/s     w/s    rkB/s    wkB/s  r_await  w_await  aqu-sz  %util
nvme0n1  12.0  2140.0    96.0  68480.0     0.21    38.40   82.10  99.80
```

- `r/s`, `w/s`: read and write operations per second (IOPS); `rkB/s`, `wkB/s`: throughput.
- `r_await`, `w_await`: average time in milliseconds per request, **including time queued**. This is the latency your threads feel.
- `aqu-sz`: average number of requests in flight or queued.
- `%util`: the fraction of time the device had at least one request in flight. For a hard disk, near 100% means saturated. For SSDs and network volumes, which serve many requests in parallel, 100% does not necessarily mean saturated; trust `await` and queue size more.

Here writes take 38 ms each with 82 requests queued: the device (or its provisioned limit) is overwhelmed by writes.

### `ss`: sockets and their queues

`ss` ("socket statistics") replaces the older `netstat`.

```text
$ ss -s                          # summary: counts by state
$ ss -ltnp                       # listening TCP sockets, numeric, with owning process
$ ss -tan state close-wait       # all TCP sockets in CLOSE_WAIT
$ ss -tn 'dport = :5432'         # connections to PostgreSQL's port
$ ss -tin                        # per-connection TCP internals (RTT, congestion window; Chapter 6)
```

For listening sockets, remember: `Recv-Q` = current accept queue length, `Send-Q` = its limit (Section 4.6). For established sockets, `Recv-Q` is bytes received but not yet read by the application (a large value means the application is not reading) and `Send-Q` is bytes sent but not yet acknowledged by the peer (a large value means the peer or network is slow). Pair it with `nstat -az TcpExtListenOverflows TcpExtListenDrops` for accept-queue overflow counters.

### `dmesg`: what did the kernel say?

`dmesg -T` prints the kernel's log buffer with human-readable timestamps (on systemd hosts, `journalctl -k` shows the same). The kernel reports things no application log will contain:

| You see a line like... | It means |
|---|---|
| "Out of memory: Killed process 4100 (server)" | The system-wide OOM killer ran (Chapter 3, Section 3.17) |
| "Memory cgroup out of memory: Killed process 4100 (server)" | A container (cgroup) hit `memory.max` (Section 4.10) |
| "I/O error, dev nvme0n1, sector ..." | The storage device failed a request |
| "EXT4-fs ... Remounting filesystem read-only" | The filesystem detected errors and stopped accepting writes |
| "Possible SYN flooding on port ..." | A listening socket's SYN queue of half-open connections filled; the kernel used SYN cookies or dropped SYNs (Section 4.6; accept-queue overflow is counted by `nstat` instead) |
| "VFS: file-max limit ... reached" | The system-wide open-file limit was hit (Section 4.9) |

### Other commands used in this chapter

`df -h` and `df -i` (space and inodes per filesystem), `du -sh <dir>` (space used by files that still have names), `free -m` and `/proc/meminfo` (page cache and dirty data), `/proc/<pid>/fd` and `/proc/<pid>/limits` (FDs and limits), and the cgroup files `cpu.stat`, `memory.events`, `memory.stat` and `pids.current` (Section 4.10).

---

## Chapter Summary

A process reaches the outside world through **file descriptors**: small integers indexing a per-process table, whose entries point to **open file descriptions** (which hold the offset and flags, and are shared across `fork` and `dup`), which point to **inodes** (the file itself). Files, pipes, sockets, `epoll` instances and timers all sit behind FDs, so one small set of system calls serves them all, and every connection a server holds is an FD counting against a limit.

On disk, a **filesystem** maps names (directory entries) to inodes and inodes to data blocks. Several names can point to one inode (hard links); a file lives until its last name is removed **and** its last FD is closed, which is how deleted log files keep filling disks. `rename` is atomic. A **journal** keeps the filesystem's own structure consistent after a crash; it does not make your data durable or your writes atomic.

Reads and writes go through the **page cache**. `write()` returns when bytes are in RAM, marked dirty, and visible to every process on the machine; writeback reaches the disk later. Only **`fsync`** (plus syncing the directory for new or renamed files) waits until the device reports the data stable, and that wait, from tens of microseconds to many milliseconds depending on the device, bounds how fast a database can commit unless it batches commits. Devices can lie about flushes, errors from `fsync` cannot be fixed by retrying, and `O_DIRECT` bypasses the cache without making anything durable.

A thread waiting for I/O can **block**, **poll** non-blocking FDs, or **multiplex** many FDs through one waiting call. `select` and `poll` rescan every watched FD on every call; **`epoll`** keeps the interest set in the kernel, has each socket's wake-up push it onto a ready list, and returns only ready FDs, so its cost follows activity rather than connection count. That solved the **C10K problem** and underlies every event-loop server, at the price that nothing on the loop may block. Edge-triggered mode demands draining to `EAGAIN`; readiness never means a whole message has arrived. **`io_uring`** goes further, to completion-based asynchronous I/O through shared rings.

A TCP server calls `socket`, `bind`, `listen` and `accept`; the kernel completes handshakes itself and queues connections in the **SYN queue** and **accept queue**, so an overloaded or stuck server leaves clients connected but unanswered, with no trace in its own logs. **Buffering** reduces system calls but adds a place where a crash loses data; **zero-copy** (`sendfile`, `splice`) moves file data to sockets without visiting user space, unless TLS or transformation requires it. Processes on one machine can talk through pipes, Unix domain sockets, shared memory, message queues and signals, but services talk over the network because it works wherever the other side runs.

**Rlimits** cap FDs and threads per process; "Too many open files" is `EMFILE`, and it is a different failure from pool or port exhaustion. **Containers** are ordinary processes, given private views of PIDs, networks, mounts and hostnames by **namespaces**, held to CPU quotas, memory limits and PID counts by **cgroups**, and run from layered **images**. They share the host kernel, which makes them light and fast to start and also less isolated than **VMs**, which run their own kernels on a hypervisor using hardware virtualisation. Finally, a well-run server in a container is the process that receives `SIGTERM`, keeps serving while load balancers catch up, drains, closes its pools, flushes and exits before the grace period ends.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| File descriptor | Meaningful only in its process; lowest free number; resource lives while any FD refers to it | Full reads or writes in one call; same resource after `close`; non-overlapping writes without `O_APPEND` |
| Journaling filesystem | Filesystem structure consistent after a crash | Recent data on disk; application-level consistency; atomicity across files; protection from disk failure |
| `rename` | Atomic replacement of a name within one filesystem | Durability of the rename without syncing the directory |
| Page cache / `write()` | Immediate visibility to all processes on the machine | Durability; writeback timing or order |
| `fsync()` | Data and needed metadata on stable storage (if the device is honest) | Durability of the directory entry; other files; meaning of a retry after an error |
| `O_DIRECT` | Bypasses the page cache | Durability (device cache) |
| Non-blocking I/O | A call never sleeps | That the call makes progress |
| `epoll` | Reported FDs could make progress when reported; cost proportional to ready FDs | That the FD is still ready; a complete message; usefulness for regular files; repeated edge-triggered notifications |
| `io_uring` | One completion per submitted operation | Completion in submission order (unless linked) |
| `listen` / `accept` | Kernel completes handshakes; each accepted connection gets its own FD | That the application ever accepts a queued connection; that `connect` success means the application saw it |
| Socket `write()` | Bytes copied to the kernel's send buffer | Delivery to or processing by the peer |
| Buffering | Fewer system calls | Survival of buffered data if the process dies |
| `sendfile` | Same result as read + write with fewer copies | Sending all bytes in one call on non-blocking sockets; use with TLS without kernel TLS |
| Shared memory | Fastest IPC; immediate visibility | Any coordination; consistency after one process crashes mid-update |
| Rlimits | A process cannot exceed its soft limit | That the limit is set where you think (systemd vs login vs container) |
| Namespaces | A process cannot name resources outside them | Protection from kernel vulnerabilities; virtualised `/proc/meminfo` or CPU counts |
| cgroups | CPU per period, memory and PID ceilings for a group | No throttling-induced latency; no OOM kill below the host's free memory; isolation of caches and bandwidth |
| VM | A separate kernel behind a hypervisor | Freedom from noisy neighbours or hardware side channels |
| `SIGTERM` + grace period | `SIGKILL` only after the grace period | That load balancers stopped routing; that PID 1 forwards the signal; that buffers are flushed |

---

## Interview Drill

**Q1. What is a file descriptor? What happens to file descriptors across `fork`?**
A strong answer must mention:
- An integer index into the process's FD table, returned by `open`, `socket`, `accept`, `pipe`; used in every later I/O call.
- Three levels: FD table entry → open file description (offset, mode, flags such as `O_APPEND`, `O_NONBLOCK`) → inode.
- `fork` copies the FD table, but both copies point to the same open file descriptions, so parent and child share offsets; `dup` does the same within one process.
- The resource is released when the last FD referring to the open file description is closed; close-on-exec prevents leaking FDs into `exec`ed programs.
- Lowest free number is reused, which is why use-after-close bugs are dangerous.
*Common weak answer:* "It is a pointer to a file." It is not a pointer, it is not only for files, and this answer misses the shared open file description.

**Q2. What is an inode? What is the difference between a hard link and a symbolic link?**
A strong answer must mention:
- An inode holds a file's metadata (type, size, owner, permissions, timestamps, link count) and data block locations, but not its name.
- A directory maps names to inode numbers; a hard link is just another such entry, equal to the first.
- A symlink is a separate file containing a path; it can cross filesystems and point to directories, and dangles if the target is removed.
- Data is freed only when the link count is 0 and no process has the file open.
*Common weak answer:* "A hard link is a copy of the file." No data is copied; both names refer to one inode.

**Q3. What does a journaling filesystem protect against, and what does it not protect against?**
A strong answer must mention:
- It writes intended metadata (and in some modes data) changes to a journal with a commit record before applying them; after a crash it replays committed entries and discards incomplete ones.
- It guarantees structural consistency and fast recovery without a full `fsck`.
- It does not make recently written data durable (page cache), does not provide application-level atomicity across writes or files, and does not protect against device failure.
- ext4's ordered mode writes data blocks before committing the metadata that points to them.
*Common weak answer:* "Journaling means no data is lost in a crash." Unsynced data in the page cache is lost regardless.

**Q4. What is the page cache? Why does Linux show almost no free memory?**
A strong answer must mention:
- Kernel RAM holding file pages; `read` hits are memory copies, misses go to the device; readahead prefetches sequential data.
- `write` copies into the cache and marks pages dirty; flusher threads write them back later; heavy writers are throttled at the dirty limit.
- Idle RAM is used as cache; clean pages are dropped instantly when processes need memory, so "available" is the meaningful number, not "free".
- `mmap` and `read` share the same cached pages; in containers the cache is charged to the cgroup.
*Common weak answer:* "Low free memory means the server needs more RAM." The cache is reclaimable; look at `available` and paging activity.

**Q5. What does it mean when `write()` returns? Why does `fsync` matter?**
A strong answer must mention:
- `write` returning means the bytes are in the page cache and visible to other processes, not on disk.
- `fsync` blocks until the file's data and needed metadata are written and the device acknowledges a cache flush.
- New and renamed files also need the directory synced.
- Databases must `fsync` (their log) before acknowledging a commit; its latency bounds commit rate, so they use group commit and a sequential write-ahead log (Chapter 15).
- Caveats: devices that ignore flushes, platforms where `fsync` does less (macOS needs `F_FULLFSYNC`), and errors that cannot be retried (fsyncgate).
*Common weak answer:* "`fsync` makes writes faster" or "`write` saves to disk." Both confuse visibility with durability.

**Q6. What is `O_DIRECT`, and does it make a write durable?**
A strong answer must mention:
- It bypasses the page cache, transferring between the user buffer and the device, with alignment requirements.
- It does not flush the device's volatile write cache, so durability still needs `fsync`/`fdatasync` or `O_DSYNC`.
- Databases use it to avoid double caching and to control their own buffer pool and write timing.
- Every read becomes a device read; no kernel readahead or caching.
*Common weak answer:* "Yes, it writes straight to disk." It writes to the device, which may still hold the data in volatile cache.

**Q7. What is the difference between blocking, non-blocking and asynchronous I/O?**
A strong answer must mention:
- Blocking: the call sleeps until it can make progress.
- Non-blocking: the call returns `EAGAIN` immediately if it cannot; alone it leads to busy polling.
- Multiplexing (`select`/`poll`/`epoll`) waits on many FDs and reports readiness; the program then does a synchronous non-blocking `read`.
- Asynchronous (`io_uring`, Windows completion ports): submit an operation, the kernel completes it including the copy, and reports completion.
- Readiness model vs completion model; blocking vs non-blocking concerns one call, synchronous vs asynchronous concerns who performs the transfer.
*Common weak answer:* "Non-blocking and asynchronous are the same thing." Non-blocking `read` with `epoll` is still synchronous I/O.

**Q8. How does `epoll` work, and why does it scale better than `select` and `poll`?**
A strong answer must mention:
- `epoll_create1` makes a kernel object; `epoll_ctl` registers each FD once in an interest list (a red-black tree) and hooks a callback on the FD's wait queue.
- When an FD becomes ready, the callback adds it to a ready list; `epoll_wait` returns only ready entries.
- `select`/`poll` copy and scan every watched FD on every call (O(watched)), and `select` is limited to `FD_SETSIZE` (1,024 commonly).
- `epoll_wait` is O(ready); idle connections cost memory but no CPU per wait.
- It does not support regular files, which are always "ready".
*Common weak answer:* "`epoll` is faster because it is newer / uses a better algorithm." The point is the kernel-held interest set and the ready list.

**Q9. Level-triggered versus edge-triggered: what is the difference, and what goes wrong with edge-triggered if you are careless?**
A strong answer must mention:
- Level-triggered reports an FD as long as it is ready; edge-triggered reports when it becomes ready.
- With edge-triggered, you must read/write/accept in a loop until `EAGAIN`.
- Leaving data unread means no further notification until new data arrives, so the connection can hang forever, often only under load.
- Edge-triggered reduces wake-ups; level-triggered is more forgiving.
*Common weak answer:* "Edge-triggered is just faster." It changes the correctness contract.

**Q10. What is the C10K problem, and how was it solved?**
A strong answer must mention:
- Handling 10,000 concurrent connections on one server (posed in 1999).
- Thread-per-connection fails on memory per stack, scheduler load and context-switch overhead, while most connections are idle.
- Solution: non-blocking sockets plus scalable readiness notification (`epoll`, `kqueue`) in a few event-loop threads.
- Consequence: handlers must not block; blocking work goes to separate pools (Chapter 11).
*Common weak answer:* "Use more threads" or "use a faster CPU." The bottleneck was the model, not raw speed.

**Q11. Walk through the system calls a TCP server makes. What is the listen backlog?**
A strong answer must mention:
- `socket` → `bind` (address and port; `EADDRINUSE`) → `listen` (backlog) → loop on `accept`, each returning a new connected FD → `read`/`write` → `close`.
- The client: `socket` → `connect` (kernel picks an ephemeral port, performs the handshake).
- The kernel completes handshakes without the application; half-open connections are in the SYN queue, established ones in the accept queue.
- The backlog sizes the accept queue (capped by `somaxconn` on Linux); overflow leaves clients connected but unanswered; `ss -ltn` shows `Recv-Q`/`Send-Q`.
- One port serves many clients because each connection is a distinct 4-tuple.
*Common weak answer:* "The backlog is the maximum number of connections." It bounds only connections waiting to be accepted.

**Q12. What is zero-copy? How does `sendfile` work?**
A strong answer must mention:
- Traditional `read` + `write`: two DMA copies, two CPU copies (through user space), two system calls per chunk.
- `sendfile` sends file data from the page cache to a socket inside the kernel, removing the user-space copy; with NIC scatter-gather, no CPU copy remains.
- Benefits: less CPU, less memory bandwidth, fewer system calls; used by static file servers and brokers such as Kafka.
- Does not apply when data must be encrypted in user space (TLS without kernel TLS) or transformed.
*Common weak answer:* "Zero-copy means data is never copied." DMA copies still happen; the CPU copies through user space are removed.

**Q13. What IPC mechanisms exist, and when would you use each?**
A strong answer must mention:
- Pipes (one-way byte stream between related processes, blocking for backpressure, EOF and `SIGPIPE` semantics); FIFOs (named pipes).
- Unix domain sockets: bidirectional, permission-controlled by path, peer credentials, FD passing; local databases and sidecars.
- Shared memory: fastest, no copies, but no coordination; requires cross-process locks; PostgreSQL's shared buffers.
- Message queues (discrete messages, local) and signals (control only).
- Services use TCP/HTTP/gRPC for location independence, independent deploy and failure isolation, accepting latency and partial failure.
*Common weak answer:* Listing names without trade-offs, or confusing kernel message queues with brokers like Kafka.

**Q14. What causes "Too many open files", and how do you diagnose it?**
A strong answer must mention:
- `EMFILE`: the process reached its `nofile` soft limit; every `open`, `socket`, `accept` and `connect` fails (`ENFILE` is the system-wide version).
- Causes: an FD leak (unclosed files or sockets, often on error paths; `CLOSE_WAIT` sockets) or a limit too low for legitimate load.
- Evidence: `/proc/<pid>/fd` count vs `/proc/<pid>/limits`; `lsof -p` grouped by type and destination; trend over time.
- Where limits come from: systemd unit, container runtime, not `limits.conf` for services.
- Different from connection pool or ephemeral port exhaustion.
*Common weak answer:* "Raise the ulimit." That only delays a leak.

**Q15. What is the difference between a virtual machine and a container? What are namespaces and cgroups?**
A strong answer must mention:
- A VM runs its own kernel on virtual hardware provided by a hypervisor using hardware virtualisation (VT-x/AMD-V, EPT/NPT).
- A container is an ordinary host process with namespaces (pid, net, mnt, uts, ipc, user, cgroup, time) restricting what it sees and cgroups (cpu, memory, pids, io) limiting what it uses, plus a layered image filesystem.
- Containers share the host kernel: faster start, less overhead, higher density, weaker isolation.
- cgroups enforce CPU quotas (throttling) and memory limits (OOM kill within the group).
*Common weak answer:* "A container is a lightweight VM." There is no guest kernel and no virtual hardware.

**Q16. Why is a container less isolated than a virtual machine?**
A strong answer must mention:
- All containers share one kernel; its large system-call surface is the boundary, so a kernel exploit can escape to the host.
- Root in a container without a user namespace is root on the host kernel, limited only by capabilities, seccomp and LSMs.
- Privileged containers, mounted runtime sockets and shared host namespaces remove protections.
- Some resources are not namespaced or limited (kernel memory, caches, bandwidth without io limits).
- Mitigations: user namespaces, non-root users, seccomp, gVisor, micro-VMs (Firecracker, Kata).
*Common weak answer:* "Containers are just as secure if configured well." Configuration narrows the surface but the shared kernel remains.

**Q17. Why might a containerised application ignore `SIGTERM`, and how do you fix it?**
A strong answer must mention:
- The process is PID 1 in its PID namespace; the kernel does not apply default signal actions to PID 1, so without a handler `SIGTERM` does nothing.
- Or PID 1 is a shell (shell-form command or wrapper script) that does not forward the signal to the server.
- Symptom: the container stops only after the grace period, exit code 137.
- Fixes: exec form, `exec` in wrapper scripts, a handler in the server, or a minimal init (`tini`, `--init`) that forwards signals and reaps zombies.
- Then implement the shutdown sequence: keep serving briefly, stop accepting, drain, close pools, flush, exit before the grace period.
*Common weak answer:* "Docker sends SIGKILL so there is nothing to do." `SIGTERM` comes first; ignoring it is the bug.

**Q18. What happens when a container exceeds its memory limit? Its CPU limit?**
A strong answer must mention:
- Memory: the kernel reclaims the cgroup's page cache, then the OOM killer kills a process in the cgroup (`SIGKILL`, exit 137, "Memory cgroup out of memory" in `dmesg`, `oom_kill` in `memory.events`), even if the host has free memory.
- The memory limit counts all of the group's memory, including non-heap memory and page cache.
- CPU: no kill; once the quota for the period is used, all threads are throttled until the next period, causing latency spikes with low average CPU (`nr_throttled` in `cpu.stat`).
- Runtimes must size heaps and pools from cgroup limits, not host totals.
*Common weak answer:* "The container slows down when it uses too much memory." Exceeding `memory.max` kills; slowing is the CPU behaviour (and `memory.high`).

---

## Scenario Exercises

1. Clients of an internal API report that connections succeed but requests time out after 30 seconds. The API's own request logs show nothing for the failed requests, and its CPU is 10%. List three hypotheses and the evidence that separates them.
2. A Node.js service has an endpoint that generates a report by reading a 200 MB file and parsing it. Whenever someone calls it, latency rises for every other endpoint on that instance for several seconds. Explain the mechanism and two fixes.
3. An upload service writes each uploaded file to disk, closes it, and returns `201 Created`. After the host lost power, several files that clients were told were saved exist but have zero length, and two are missing entirely. Explain both outcomes and give the correct write sequence.
4. A latency-sensitive API runs in containers with a CPU request but no CPU limit. On nodes it shares with a nightly batch container, its p99 latency triples, although `cpu.stat` shows no throttling for the API. What else is shared between containers on one node, and how would you find which shared resource is the problem?
5. A security review finds a container that runs as root, has `/var/run/docker.sock` mounted so it can start helper containers, and runs with the default capability set. The team says "it's in a container, so it's isolated". What could an attacker who achieves code execution in it do, and what would you change?
6. A file download service moves from plain HTTP behind a TLS-terminating load balancer to end-to-end TLS, with TLS now terminated in the service itself. Its CPU usage per gigabyte served roughly triples. Explain why.
7. Writes to a cache directory fail with "No space left on device", yet `df -h` shows the filesystem 40% full. What is happening, and how do you confirm it?

## Diagnosis Lab

**Lab 1: The slow FD leak**
- *Symptom:* every few days, an order service starts failing all requests. Logs show a mix of "Too many open files", "could not connect to database" and "unknown host". Restarting fixes it for another few days.
- *Hypotheses:* (a) the FD limit is too low for peak load; (b) an FD leak; (c) a DNS or database outage (contradicted: other instances are fine at the same moment).
- *Evidence:* `cat /proc/<pid>/limits` shows "Max open files 65536". `ls /proc/<pid>/fd | wc -l` sampled every hour rises steadily from 900 after a restart, by about 1,200 per hour regardless of traffic level, so it is a leak, not load. `lsof -p <pid>` grouped by `NAME` shows about 61,000 sockets to `10.0.9.31:443` in `CLOSE_WAIT`. That is the address of a payment provider's API. `ss -tan state close-wait 'dport = :443' | wc -l` agrees.
- *Root cause:* the payment client created a new HTTP client per request and, on the error path for non-2xx responses, returned without closing the response. The provider closed its side of each idle connection (hence `CLOSE_WAIT`), but the service never closed its FD.
- *Fix:* close responses on every path (a cleanup block that runs on success and on error); share one pooled HTTP client.
- *Prevention:* export the open-FD count and its ratio to the limit as metrics; alert on a steady upward trend; alert on `CLOSE_WAIT` counts; code review rule for resource closing.

**Lab 2: Throttled by a hidden neighbour inside the container**
- *Symptom:* a containerised API with a CPU limit of 2 CPUs shows p99 latency jumps from 40 ms to 300 ms for about two minutes at the start of every hour. Average CPU usage over the hour is 0.8 CPUs. The application's own request handlers show no change in work done.
- *Hypotheses:* (a) garbage-collection pauses (Chapter 2); (b) CPU throttling; (c) a slow dependency at the top of the hour.
- *Evidence:* the cgroup's `cpu.stat` sampled each minute shows `nr_throttled` rising only during those two minutes, with `throttled_usec` growing by about 40 seconds per minute. GC logs show normal pauses. Traces show the extra time inside the service, between operations, not in calls to dependencies. `top` inside the container during a spike shows a parallel compression process (`pigz`) using several cores: the image's entrypoint also starts a cron daemon, which compresses the previous hour's local log files.
- *Root cause:* the log compression job ran inside the API's cgroup and consumed most of its 200 ms-per-100 ms CPU quota; the API's threads were frozen for the rest of each period.
- *Fix:* remove the cron daemon from the image; send logs to standard output (Section 4.11) so the platform handles them; if compression is needed, do it in a separate container with its own limits.
- *Prevention:* one main process per container; dashboard `nr_throttled` and `throttled_usec` next to latency.

**Lab 3: Disk full, but nothing is using it**
- *Symptom:* a host's root filesystem reaches 100%. The database on it stops accepting writes. `du -sh /*` adds up to 22 GB on a 100 GB volume.
- *Hypotheses:* (a) deleted files still held open; (b) inode exhaustion (contradicted: the error is space, and `df -i` shows 8% inode use); (c) files hidden under a mount point (files written to a directory before a volume was mounted over it).
- *Evidence:* `df -h /` shows 100 GB used. `lsof +L1` lists the application process with FD 1 and FD 2 open on `/var/log/app/app.log (deleted)`, `SIZE/OFF` 76 GB. The log-rotation configuration moves and deletes `app.log` nightly but does not signal the application to reopen it.
- *Root cause:* the application kept writing to the unlinked file through its open FD (Section 4.2). The name was gone, so `du` could not see it; the blocks could not be freed while the FD was open.
- *Fix:* immediate: truncate the file through the FD (`: > /proc/<pid>/fd/1`), or restart the process, to free space; then restore the database. Durable: make rotation signal the process to reopen its log (or use copy-and-truncate), or better, log to standard output and let the platform rotate.
- *Prevention:* alert on disk usage with headroom; put logs on a separate filesystem from the database; alert when `df` and `du` diverge.

**Lab 4: Slow commits after a quiet change**
- *Symptom:* a PostgreSQL-backed service's write requests slow from a p99 of 25 ms to 400 ms during busy periods. Read requests are unaffected. Nothing was deployed.
- *Hypotheses:* (a) lock contention in the database (Chapter 17); (b) slow `fsync` on the write-ahead log volume; (c) CPU saturation on the database host.
- *Evidence:* database CPU is 30%. The database's statistics show WAL sync time per commit rising from about 1 ms to 60 ms during the slow periods, while lock waits are flat. `iostat -x 1` on the WAL volume shows `w_await` at 50–120 ms with `aqu-sz` high, and write IOPS flat at a round number. The cloud volume's metrics show its burst allowance exhausted: some cloud block-storage types allow bursts above a baseline IOPS rate for a limited time, and a new nightly batch import had used it up. Running `pg_test_fsync` against the volume confirms only a few hundred syncs per second. `strace -f -T -e trace=fdatasync,fsync -p <wal-writer-pid>` for a few seconds shows `fdatasync` calls taking 40–110 ms each (in angle brackets), confirming that the time goes to the sync itself. `dmesg -T` shows no I/O errors or device resets, which rules out failing hardware and points to a rate limit.
- *Root cause:* commits wait for `fsync` of the WAL (Section 4.4); the storage, reduced to its baseline rate, made each flush slow, and group commit could not absorb the difference.
- *Fix:* move the WAL to a volume with provisioned IOPS sufficient for peak, or with lower sync latency; reschedule or throttle the batch import.
- *Prevention:* alert on WAL sync time and on the volume's burst balance; benchmark `fsync` rate on new storage before putting a database on it.

## Solution Sketches

1. (a) **Accept queue overflow**: the server is not calling `accept` fast enough (all workers busy, or the event loop blocked); evidence: `ss -ltn` shows `Recv-Q` at the `Send-Q` limit; `nstat` shows `TcpExtListenOverflows` rising. (b) **Blocked event loop or exhausted worker pool**: connections are accepted but never read; evidence: thread or loop stack dumps show threads stuck in a blocking call; established sockets show a growing `Recv-Q` (unread requests). (c) **A middlebox in front of the server** (load balancer, firewall) holding or dropping connections; evidence: no SYNs reach the server for failed requests (a packet capture, Chapter 6). The absence of log lines points to the kernel or something in front of the application, not to slow handler code.
2. Node.js runs JavaScript on one event-loop thread. If the endpoint reads the file synchronously, the loop thread is blocked in the read; even if it reads asynchronously (on libuv's thread pool), parsing 200 MB on the loop thread is CPU work that blocks every other callback. Fixes: stream and parse incrementally in small chunks so the loop regains control between them; move the parsing to a worker thread or a separate job service (Chapter 25); cache or precompute the report.
3. Zero-length files: the data was written into the page cache and the file's metadata (its creation, perhaps its size) reached the journal, but the data blocks were never written back before power was lost; `close` does not sync. Missing files: the directory entries were never made durable. Correct sequence: write to a temporary file, `fsync` it, `rename` it to the final name, `fsync` the directory, then return `201`. If response latency matters, batch many uploads per sync, but never return success before the sync.
4. Containers on a node share the kernel and the hardware: CPU cores (without a limit, the API competes for cores through `cpu.weight` only, so under contention it gets its share, and its threads wait in run queues longer: check scheduling delay, for example with `/proc/<pid>/schedstat` or per-cgroup pressure files such as `cpu.pressure`), CPU caches and memory bandwidth (not limited by cgroups), the disk (check `iostat` and `io.pressure`), the page cache (the batch job's reads can evict the API's cached files: major faults rise), and network bandwidth. Compare per-cgroup pressure (PSI, Chapter 3, Section 3.16) for CPU, memory and I/O during the batch window to see which resource the API is waiting on.
5. With the runtime's control socket, the attacker can ask the runtime to start a new **privileged** container with the host's root filesystem mounted, which gives full control of the host and every container on it. Running as root without a user namespace also makes any kernel exploit more damaging. Changes: remove the socket mount (use a dedicated, restricted job API or an orchestrator feature to launch helpers); run as a non-root user; drop all capabilities not needed; keep the default seccomp profile; consider user namespaces.
6. Previously the service could send file data from the page cache to the socket with `sendfile` (zero-copy) because the bytes left the service unencrypted. With TLS in the service, every byte must be read into user space, encrypted and written back, which adds CPU copies and the encryption itself (Chapter 8). Options: kernel TLS to restore a zero-copy-like path, terminating TLS in a proxy that supports it, or offloading large downloads to a CDN or object storage (Chapter 10).
7. The filesystem has run out of **inodes**: millions of small files in the cache directory have used every inode, while blocks remain free. Confirm with `df -i` (`IUse%` at 100%) and count files with `find <dir> -xdev | wc -l`. Fix: delete or consolidate the small files (store entries in fewer, larger files or in a database or cache server); if the workload truly needs many files, recreate the filesystem with more inodes or use one that allocates them dynamically (such as XFS).

---

**END OF CHAPTER 4**
