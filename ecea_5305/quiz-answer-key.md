# ECEA 5305 — Quiz Answer Key
### Linux System Programming and Introduction to Buildroot

Answers to the six graded quizzes: Intro to System Programming, File I/O, Processes, Threading, Signals, Time. Coursera's download includes only the questions, so these answers were worked out from the lecture transcripts and slides. Each one includes a short **why** so you can study from it. Answers marked ⚠️ are ones where the wording is ambiguous; the reasoning for those is spelled out.

> **Note on use:** The course's AI policy ("Full AI") allows AI tools on coursework *if you document the usage*. If you use this key on a quiz you haven't submitted yet, cite it as the AI-usage policy describes. AI tools are **not** allowed during the proctored final.

---

## Quiz 1 — Intro to System Programming

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | System programming is similar to application programming in that both *may* interact with the kernel and C library | **True** | Both can. System software just does so without higher-level libraries in between. |
| 2 | System programs can use syscalls to invoke kernel functions from user space | **True** | A syscall is, by definition, a kernel function invoked from user space. |
| 3 | User-space apps can alternatively be linked directly with kernel-space functions | **False** | Not allowed, for security and reliability. You must trap into the kernel (software interrupt). |
| 4 | Examples of wrapper functions provided by glibc (select all) | **System calls; Threading support** | glibc = "wrappers for system calls, threading support…". Compilation is the toolchain (GCC), not libc. |
| 5 | "APIs ensure source compatibility" means… | **Application source code can be compiled for another platform and expected to work properly** | Copying an *executable* to another platform is binary compatibility, which is the ABI's job. |
| 6 | True statements about ABIs (select all) | **Can define how an app interacts with itself; …with libraries; …with the kernel; Enforced by the toolchain for a platform** | "Defines interfaces at the source level" describes an API, not an ABI. |
| 7 | Most everything is a file in Linux, except device access, which is handled completely differently | **False** | Devices are accessed as files too (e.g., `/dev/ttyUSB0`). |
| 8 | A file's inode contains (select all) | **Modification timestamp; Owner of the file; Location of the file's data on disk** | **The filename is NOT in the inode**; directories map names to inodes. |
| 9 | Linux directories map names with which to access files | **True** | A directory is a file of name → inode entries. |
| 10 | Types of "special files" (select all) | **Block devices; Character devices; Named pipes** | Special files = character devices, block devices, named pipes, sockets. Directories and links are not. |
| 11 | Block and character devices can both address memory on byte boundaries | **False** | Only character devices are byte (linear) streams. Block devices are addressed in sectors/blocks. |
| 12 | Block and character devices are both "special files" | **True** | Both are kernel objects represented as files. |
| 13 | Processes use memory virtualization; each appears to have its own single linear address space | **True** | Virtualized memory: each process sees its own dedicated RAM. |
| 14 | Two threads in the same process each have their own memory space and can't access each other's memory | **False** | Threads **share** the process address space (each has its own stack, but the stacks live in shared memory). |
| 15 | uid 1 is associated with root | **False** | **UID 0** is root. |
| 16 | Octal 755 | **Owner may read, write, execute. Group and Everyone else may read and execute only.** | 7 = 4+2+1 (rwx), 5 = 4+1 (r-x). |
| 17 | Octal 640 | **Owner may read and write only. Group may read only. Everyone else has no access.** | 6 = rw-, 4 = r--, 0 = ---. |
| 18 | Signals are a one-way asynchronous notification mechanism for a process | **True** | Straight from the slides. |
| 19 | The Ctrl-C handler is an example of what type of communication mechanism? | **Signal** | Ctrl-C sends SIGINT, which a signal handler catches. |
| 20 | Which is an absolute pathname? | **/etc/httpd/httpd.conf** | An absolute path starts with `/`. |
| 21 | Which is a relative pathname? | **httpd/httpd.conf** | No leading `/` means relative to the working directory. |

---

## Quiz 2 — File I/O

**Q1. Which commands correctly open a new file for writing at `/existingpath/newfile` that will be readable by anyone, assuming a umask of 022? (select all)**
✅ **`open("/existingpath/newfile", O_WRONLY | O_CREAT, S_IRWXU | S_IRWXG | S_IROTH);`** (only this one)
- `O_RDONLY` and plain `O_WRONLY` (without `O_CREAT`) can't create a new file.
- `O_WRONLY | O_CREAT` with **no mode argument** leaves the permissions undefined (garbage from the stack). When `O_CREAT` is used, the mode argument is required.
- `S_IRWXU | S_IRWXG` gives others **no** read permission, so the file isn't readable by everyone.
- With umask 022: `0774 & ~022 = 0754`, so others keep read permission. ✔

**Q2. Problems with the code** (`fd = open(".../rocketcommand", O_RDONLY); nr = read(fd, &firectl, sizeof(unsigned long)); if (nr != -1) { if (firectl != 0) fireRocket(); }`)
✅ **"In line 6, nr should be checked for sizeof(unsigned long) to indicate success instead of comparison with -1"**
- ❌ "Check nr for 0": a return of 0 means **EOF / nothing read**, not success.
- ❌ "open() should specify an access mode in a 3rd argument": the access mode (`O_RDONLY`) goes in the *flags*. The third (mode) argument only matters with `O_CREAT`.
- The issue is a **short read**: `read()` can return fewer bytes than requested (0 < nr < 8), leaving `firectl` partially filled while the code still fires.

**Q3. You should always use O_NONBLOCK, even if your thread has no other work to do while waiting.**
✅ **False**. O_NONBLOCK exists so a thread can do *other work* while waiting. If there's nothing else to do, blocking is simpler and more efficient.

**Q4. Problems with the code** (`fd = open(..., O_RDONLY); firectl = 1; nr = write(fd, firectl, sizeof(unsigned long)); if (nr == 4) ...`) (select all)
✅ **All three**:
- **Line 4**: `O_RDONLY` won't allow writing (it needs `O_WRONLY` or `O_RDWR`).
- **Line 6**: `write()` needs a **pointer** to the buffer (`&firectl`), not the value.
- **Line 7**: compare with `sizeof(unsigned long)`, not the hard-coded `4`. `unsigned long` is 8 bytes on 64-bit architectures.

**Q5. write() commands typically do not complete writes to disk before returning success.**
✅ **True**. The data goes into a kernel buffer, because disks are slow.

**Q6. A write() command writes directly to disk, so success means the data is stored on disk.**
✅ **False**. Same reason. Use `fsync()` if you need that guarantee.

**Q7. Best way to synchronize a file write to disk for a single file with descriptor fd.**
✅ **`fsync(fd)`**. `sync()` flushes *everything* system-wide (a performance hit). O_NONBLOCK has nothing to do with syncing.

**Q8. Seeking 1 GB past EOF and writing one byte increases disk usage by about 1 GB.**
✅ **False**. Linux creates a **sparse file** with a "hole" that uses no disk space (`du` shows ~4 KB, while `ls` shows ~1 GB).

**Q9. Seeking past EOF with lseek() is allowed and can create sparse files.**
✅ **True**

**Q10. select() or poll() is generally preferable to O_NONBLOCK for non-blocking file I/O.**
✅ **True**. Multiplexed I/O is "a better solution than polling": the thread sleeps until an fd is ready instead of spinning. (`poll()` is the preferred one.)

**Q11. Reasons multiplexed I/O may beat O_NONBLOCK for an app monitoring several fds with little to do while waiting (select all)**
✅ **"Multiplexed file I/O schemes support notifications for multiple file descriptors in a single call"** and ✅ **"Multiplexed file I/O supports sleep during blocking states"**

**Q12. Behavior of `lseek(fd, 0, SEEK_END)`**
✅ **Sets the file position to the end of the file**. (`SEEK_CUR` with 0 doesn't move; `SEEK_SET` with 0 goes to the beginning.)

---

## Quiz 3 — Processes

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | A pid may be re-used after a process terminates | **True** | The kernel allocates and **reuses** pids. A pid is unique only at a single point in time. |
| 2 | Other processes are started at system startup based on the work of this process | **The init process** | pid 1 initializes the system, starts services, and launches login. (Idle = pid 0, power control.) |
| 3 | `ret = execl("/bin/vi","vi",NULL); printf(...)`: which is true? | **If the call to execl() succeeds, the printf at line 3 will never be executed** | exec **replaces** the process image and does not return on success. |
| 4 | The "p" variants (execlp/execvp) use PATH and are safer than exec() because they aren't susceptible to path injection | **False** | It's backwards: relying on PATH is what *creates* the PATH-injection risk. Use absolute paths. |
| 5 | fork() and exec() both create a new process and leave the existing process intact | **False** | fork creates a child and keeps the parent. exec *replaces* the current process. |
| 6 | Best practice: use the maximum level of rights possible | **False** | **Doctrine of least-privileged rights**: run with the minimum rights possible (less exploitable, fewer self-inflicted errors). |
| 7 | It's possible to change the effective user ID after starting the process | **True** | `setuid()`/`setgid()` change the effective IDs. |
| 8 | Sessions interact with a controlling terminal and may be associated with multiple processes | **True** | A session is a collection of process groups with a controlling tty. |
| 9 | Ctrl-C sends SIGINT to… | **All processes in a foreground process group of a session** | There's exactly one foreground process group per session, and that's the one Ctrl-C interrupts. |
| 10 | A daemon is typically associated with a single terminal | **False** | A daemon has **no controlling terminal** (that's why it calls `setsid()`). |
| 11 | A daemon should (select all) | **Use setsid()…; Change the working directory to the root directory with chdir(); Use the child process after fork(); Redirect stdin/stdout/stderr to /dev/null** | The parent exits so init can continue. chdir("/") avoids holding a directory "in use". Use /dev/null rather than a log file (a file can block unmounting); report through syslog instead. |
| 12 | Which function replaces a process? | **exec()** | |
| 13 | Which launches a new child and continues execution of the parent? | **fork()** | |
| 14 | system() is used to… | **Launch a new process and wait for termination** | system() = fork + exec + wait. |

---

## Quiz 4 — Threading

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | Threads are like processes in that each thread has its own dedicated, unique view of memory | **False** | Threads **share** the process's virtual memory. Only processes have separate views. |
| 2 | State machines are in many cases a design pattern usable as an alternative to threads | **True** | "Multithreading is an alternative to state machines" (and vice versa). |
| 3 | On a multiprocessor, state machines offer the same parallelism as threads | **False** | A state machine runs in one thread, so it can't use multiple CPUs at once. |
| 4 | UI thread freezing during long operations: which alternatives? (select all) | **Move blocking operations to a dedicated worker thread; Use non-blocking I/O and a state machine implementation** | Long-running work and blocking I/O don't belong in the UI thread. A single thread with blocking I/O makes it worse. |
| 5 | Is the concern that multithreading is more complex to understand and debug justified? | **Yes** | Complexity in writing, understanding, and debugging is the main reason *not* to multithread. |
| 6 | Event-driven threading typically won't improve by using more threads than processors | **True** | There's no waiting in event-driven threads, so threads beyond the CPU count add no parallelism. |
| 7 | Race conditions can only occur in environments that support parallelism | **False** | **Concurrency** (time-slicing on one CPU) causes races too. |
| 8 | Parallelism isn't possible on a single processor or execution unit | **True** | Parallelism means *simultaneous* execution, which needs multiple processors. |
| 9 | Race conditions are avoided by making critical regions atomic | **True** | Synchronize access to critical regions (e.g., with a mutex) so they behave atomically. |
| 10 | Associate locks with code instead of data | **False** | **Lock data, not code.** |
| 11 | The best strategy to avoid deadlocks is to add more mutexes | **False** | More locks mean *more* deadlock risk. Use fewer locks, a consistent acquisition order, and reverse-order release. |
| 12 | A ____ is a dormant program residing on storage, ready to execute | **Binary** | |
| 13 | A thread is defined as… | **Unit of execution with a virtualized stack and program state** | "OS abstraction representing binaries in action" is a process; "dormant program" is a binary. |
| 14 | ____ uses asynchronous I/O, callbacks, and multiplexed I/O with a loop that hands callbacks to a waiting thread | **Event-driven threading** | |
| 15 | ____ is the ability of two or more threads to execute in overlapping time periods | **Concurrency** | Parallelism = *simultaneous* execution. |

---

## Quiz 5 — Signals

**Q1. Events that cause signals can occur asynchronously with respect to program execution.** ✅ **True**

**Q2. Signal handlers are handled synchronously with program execution.** ✅ **False**. "The event is asynchronous *and* the handler is asynchronous." The handler interrupts the thread at any point.

**Q3. When forking, a child resets all signal actions to defaults.** ✅ **False**. The child **inherits** the parent's signal actions on fork.

**Q4. A process created via exec() has all signal actions set to defaults, including signals ignored by the parent.** ✅ **False**. exec resets signals to defaults **except those the parent ignored**, which stay ignored.

**Q5. Types of functions to *avoid* calling in a signal handler (select all)**
✅ **Functions which are not reentrant**
✅ **Functions which call other functions that are not reentrant**
✅ **Functions which access only global data** ⚠️
- Why the third: handlers "should manipulate only stack-allocated data or data provided by the caller", and global data makes a function non-reentrant (the classic signal-safety hazard). (The one exception the course allows is setting a single global *flag* in your *own* handler, not calling functions that operate on globals.)
- **Fine to call**: functions that access only local (stack) data, reentrant functions, and functions that call only reentrant functions.

**Q6. A program with a SIGALRM handler crashes after receiving the signal. Suggested troubleshooting (select all)**
✅ **Remove any global variable access in their signal handler**
✅ **Review the list of functions called from their signal handler, to be sure all are reentrant**
(❌ Removing *local* variable access doesn't help: stack data is safe.)

**Q7. A program crashes with SIGSEGV. Recommended troubleshooting (select all)**
✅ **Look for possible null pointer accesses**
✅ **Look for cases where uninitialized pointers are used for read or write**
✅ **Attempt to obtain a stack trace with gdb** (with the core file, gdb points to the faulting line)
- ❌ Uninitialized *integers* being read: a bug, but on its own not an invalid memory access.
- ❌ Divide by zero raises **SIGFPE**, not SIGSEGV.
- ❌ Rewriting from scratch: not a troubleshooting step.

**Q8. In the signal life-cycle, which occurs first?** ✅ **Signal is raised** (then stored by the kernel, then handled).

**Q9. ____ happens when the user presses Ctrl-C.** ✅ **SIGINT**

**Q10. ____ is sent on invalid memory access.** ✅ **SIGSEGV**

**Q11. The function of sigaction() is…** ✅ **Register for a signal** (i.e., set up a handler).

**Q12. Which function waits for a signal?** ✅ **pause()** (`kill()` *sends* a signal; `sigaction()` *registers*.)

---

## Quiz 6 — Time

**Q1. How does a Linux system keep wall time correct after a long power-down? (select all)**
✅ **Uses a battery powered hardware clock** and ✅ **Uses NTP (Network Time Protocol)**
(❌ Monotonic time only counts since boot. It knows nothing about the real date.)

**Q2. Kernel timer intervals (ticks/jiffies) have the same frequency on all hardware platforms.** ✅ **False**. HZ is **platform/architecture dependent** (100–1000 Hz historically on x86).

**Q3. 32-bit systems using time_t will see a clock rollover in 2038.** ✅ **True**. A signed 32-bit `time_t` overflows on January 19, 2038 (UTC), Jan 18 in US time zones.

**Q4. Which clock should timestamp events to calculate the time between them?** ✅ **CLOCK_MONOTONIC**. It can't be changed by NTP, DST, or the user.

**Q5. Which clock gives the current system time for display in Mountain Standard Time?** ✅ **CLOCK_REALTIME** (wall-clock time; the time zone is applied when formatting).

**Q6. clock_settime isn't changing the time. Troubleshooting steps (select all)**
✅ **Ensure the clock_id argument specifies CLOCK_REALTIME** (MONOTONIC isn't settable)
✅ **Ensure your program is running as a user with appropriate permissions** (setting time needs elevated privileges)
✅ **Check the return code from clock_settime**
(❌ CLK_MONOTONIC isn't settable and isn't even a real constant name.)

**Q7. sleep1() vs sleep2() (preemption much shorter than 1 s). Select all true statements.**
*(`sleep1` = relative 1 s `clock_nanosleep(CLOCK_MONOTONIC, 0, …)`; `sleep2` = `clock_gettime(MONOTONIC)` at line 13, `+1 s`, then `clock_nanosleep(…, TIMER_ABSTIME, …)` at line 19.)*
- ✅ **Both sleep1() and sleep2() execution time may be longer than 1 second depending on preemptive scheduling.** Sleeps guarantee *at least* the time, and preemption after wakeup adds more.
- ✅ **If sleep1() is preempted at line 3, its duration is extended by the preemption.** Line 3 runs before the relative sleep starts, so the full 1 s still follows the delay.
- ❌ *If sleep2() is preempted at line 14, its duration is extended.* Line 14 is **after** `clock_gettime()` sampled the start time. TIMER_ABSTIME wakes at start + 1 s regardless, so the preemption is absorbed.
- ❌ *CLOCK_REALTIME would be better for sleeping a relative amount of time.* MONOTONIC is correct, since REALTIME can jump.
- ✅ **If sleep2() is preempted at line 12, its duration is extended.** Line 12 is **before** `clock_gettime()`, so the 1 s window hasn't started yet and the delay adds to the total.

**Q8. Concerns with the `setitimer` / SIGALRM rocket code (select all)**
*(`signal(SIGALRM, rocket_fire_handler)`; `it_value = 5 s`, `it_interval = 0`.)*
- ✅ **rocket_fire_handler() may be called in response to another program.** SIGALRM can be sent from outside the process (e.g., `kill -ALRM`), or by sleep/usleep/setitimer elsewhere in the program.
- ✅ **fire_rocket() needs analysis for signal handler safety.** It runs inside a signal handler, so it must be async-signal-safe.
- ❌ Called every 5 s / continuously: `it_interval = 0` means a **one-shot** timer that fires once.
- ❌ "No problems, ship it!"

**Q9. Execution time for a process is ____.** ✅ **Process time**

**Q10. Which is an example of absolute time?** ✅ **12:34:56 UTC 3rd August 2019** ("10 mins" is relative; "100 Hz" is a frequency.)
