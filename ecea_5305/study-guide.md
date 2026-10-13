# ECEA 5305 — Linux System Programming and Introduction to Buildroot
## Final Exam Study Guide

*Built from the course lecture transcripts, slides, readings, and quizzes (instructor: Dan Walkes, CU Boulder MS-ECE).*

**Final exam format** (from "About the Final Exam"): proctored via ProctorU, **2 hours**, **1 attempt / 1 submission**, same question style as the graded quizzes (True/False, multiple choice, select-all-that-apply). Worth 15% of the course grade.
**Allowed:** one notes sheet (front and back, ≥11pt if typed, *your own notes*), or an electronic copy of it, or one URL containing your notes; pens/paper.
**Not allowed:** textbooks, lecture slides/lecture content, AI tools.

> Tip: the allowed notes sheet must be your own. Use this guide to study, then write your own condensed sheet from it. The **Cheat-Sheet Candidates** box at the end of each module points to the facts most worth putting on it.

**Course reference books** (none required): *Linux System Programming, 2nd ed.* (Robert Love, "LSP"), *Mastering Embedded Linux Programming, 2nd ed.* ("MELP"), *Linux Device Drivers, 3rd ed.* ("LDD").

---

## Table of Contents
1. [Module 1 — Linux Background, Course Setup, Intro to System Programming](#module-1)
2. [Module 2 — File I/O, Processes, Basics of Embedded Linux](#module-2)
3. [Module 3 — Threads & Synchronization, Buildroot, Signals](#module-3)
4. [Module 4 — Time, Sockets, Embedded Socket Server](#module-4)
5. [Quick-Reference Tables](#quick-reference)

---

<a id="module-1"></a>
## Module 1 — Linux Background, Course Setup, Intro to System Programming

### 1.1 Linux Overview & History
- Embedded Linux started with the **TiVo (1999)**, one of the first consumer products running Linux (it time-shifted live TV onto a hard drive).
- **~2 billion devices** ran Linux as of 2017.
- **Why run Linux on an embedded device?**
  - **Moore's Law**: processing power keeps growing, so the extra overhead of Linux matters less every year.
  - **Functionality you don't have to write yourself**: scheduler, network stack (TCP/IP), USB, WiFi, Bluetooth, storage.
  - **Ported to a wide range of architectures** (e.g., ARM).
  - **Open source and modifiable**: you can fix device-specific problems yourself.
  - **Active community** (mailing lists, experts per project).
  - **No vendor lock-in**: no single company controls the roadmap.
- **Considerations / requirements**:
  - **32-bit processor**, roughly **16+ MB RAM** and **8 MB flash**.
  - A **Memory Management Unit (MMU)** "for all practical purposes."
  - Engineers with Linux skills.
  - **May not suit some real-time applications** (or needs real-time kernel tuning).
- **The players**: the open-source community (developers from non-profits, academia, and companies, with a group behind each application) and **CPU architects / SoC vendors / board vendors**, who create reference hardware and **BSPs (Board Support Packages)**.
- **Keeping up to date matters**: BSPs often ship old kernels and aren't kept current. **Heartbleed (2014)** was an OpenSSL bug; ~200,000 devices were still unpatched in 2017 because their makers couldn't update easily.

### 1.2 The Linux Command Line
Why use it? It's the **only thing every Linux system has** (including your embedded device, which may have no display). It's often all you have when troubleshooting (e.g., early boot), and it's the easiest, fastest, and most reliable way to access a system remotely.

| Command | Purpose |
|---|---|
| `echo` | print text to the terminal |
| `cat` | con**cat**enate, i.e. print a file to the terminal |
| `pwd` | print working directory |
| `ls` / `ls -l` | list a directory / long listing with permissions |
| `cd` | change directory |
| `mkdir` | make a directory |
| `mv` | move a file or directory |
| `cp` / `cp -r` | copy; `-r` is needed to copy directories recursively |
| `rm` | remove |
| `touch` | update a file's timestamp; creates an empty file if it doesn't exist |
| `man <cmd>` | manual pages (dating back to the 1971 Unix Programmer's Manual). The number in parentheses is the manual section: **(1) user commands, (2) system calls, (3) C library functions** |
| `find . -name x` | locate files or directories by name or type |
| `grep -r "text" *` | search *inside* files for content |
| `tree` | show a directory tree (`sudo apt install tree`) |

- **vi / vim**: worth knowing because it's **the one editor you can always count on being present** (e.g., when stuck at a boot prompt). It dates from 1976 (the "visual" mode of an early editor). **Esc** enters normal (command) mode; `i` inserts, `d` deletes; **h/j/k/l = left/down/up/right**; `:w` writes, `:q` quits.
- **Shell interpreter**: handles commands from the terminal or from a script. The **Bourne shell (`sh`)** dates from 1979; **bash ("Bourne Again SHell")** replaced it, adding features from ksh and csh.
- **Wildcards**: `*` matches any characters. **The shell expands the wildcard** into the list of matching filenames *before* the command runs.
- **Pipe `|`**: sends one command's stdout into the next command's stdin (`cat file | grep x`).
- **Redirection**: `>` writes output to a file (**overwrites**); `>>` **appends**. stderr is fd 2 and redirects separately (`2>`).
- **Permissions**: three levels (**User/owner, Group, Others/world**), each with **r/w/x**. `ls -l` shows them, e.g. `-rwxr-x---`.
  - **New files don't get execute permission by default**, so `./hello.sh` gives "Permission denied" until you `chmod u+x hello.sh`.
  - **Octal**: **r=4, w=2, x=1**, one digit each for user, group, others.
    - `755` = rwxr-xr-x (owner rwx; group and others r-x)
    - `640` = rw-r----- (owner rw; group r; others nothing)
    - `766` = rwxrw-rw-
    - `600` = rw------- (owner rw only)
- **Remote access**: `ssh -p <port> user@host` (default port **22**; VirtualBox port-forwarding often maps host 2222 to guest 22). `scp -P <port> src user@host:~` copies files (note the **capital P** for scp). On Windows, use WSL or msys2.

### 1.3 Shell Scripting
- Scripts **automate frequently used commands** and customize them with variables and arguments. **Treat scripts as documentation**: if you're listing commands for someone to type, write a script instead.
- Script files conventionally end in **`.sh`**.
- **Shebang**: `#!/bin/sh` (or `#!/bin/bash`) on the first line tells the system which interpreter runs the script.
- **Special variables**: **`$0`** = invoked name of the script (including path), **`$1`–`$9`** = first nine arguments, **`$#`** = number of arguments, **`$?`** = exit status of the most recent command.
- **Conditionals**: `if [ condition ]; then …; else …; fi`. **`[ ]` is shorthand for the `test` command** (see `man test`); for example, `[ -d "$dir" ]` is true if the path exists and is a directory.
- **Exit status**: `exit 0` = success; **any non-zero exit (e.g., `exit 1`) = failure**.

### 1.4 Course Format, Licensing, Environment Setup
- **Prerequisites**: C programming and Makefiles (ideally the command line and shell scripting too).
- Build environment: an Ubuntu VM (VirtualBox, ≥4 GB RAM and 100 GB disk; the host needs ~8 GB RAM), a dedicated Linux laptop (≥8 GB RAM, 100 GB disk), or a cloud runner. Assignments are submitted via **git and GitHub Classroom**, with automated testing through **GitHub Actions on a self-hosted runner** (Linux, Docker installed). The test code is pulled in as a **git submodule** (`git submodule update --init --recursive`).
- **VirtualBox snapshots**: take one once your system is set up, so you can recover if you corrupt the VM (especially during kernel driver work).
- **Licensing: "free as in freedom, not free as in beer."**
  - **Permissive licenses** (**BSD, MIT**): you can use and modify the code **without sharing your changes or your source**. Obligations: keep the copyright and permission notice, and pass on the no-liability clause ("if it breaks, don't sue me").
  - **Copyleft licenses** (**GPL**): if you distribute a **derivative work**, you must provide its source code.
  - **LGPL**: typically used for **libraries** (e.g., the GCC/C libraries). You can **link a proprietary application without sharing your app's source**, but changes *to the library itself* must be shared.
  - **GPLv2 vs GPLv3**: TiVo published its GPL source but locked its hardware so modified binaries wouldn't run ("tivoization"). **GPLv3 forbids this. The Linux kernel remains GPLv2.**
  - Contributing to open source means written communication on mailing lists, answering questions about your code, and following each project's detailed submission rules.
- **Distributions** are collections of preconfigured software maintained by a community or company. Software versions are managed as **packages**.
  - **Why packages instead of downloaded binary installers?** Package managers are the precursor to app stores: a **centralized, trusted source**, and they **solve fragmentation**, since packages are customized per distribution.
  - **Ubuntu**: pronounced "oo-BOON-too," an African word meaning "humanity to others." First released in **2004**, based on **Debian**. Compared with Debian it's arguably **less stable but more up to date**, and **more focused on usability than licensing**. Package managers: **`apt`** (traditional) and **`snap`** (newer).
- **Root user (UID 0)** can modify anything. Default accounts are restricted so that mistakes like `rm -rf /${undefined_variable}` fail. **Ubuntu's philosophy**: your account isn't root, but can run single commands as root with **`sudo`** ("super user do"), which asks for *your* password.

### 1.5 System Programming Overview
- **System software** interfaces directly with the **kernel and the C library**. Examples: GUI, compiler, debugger, web server, database.
  - Compared with **application software**, system software **doesn't use higher-level libraries** and has **less OS/hardware detail abstracted away**. *Both* may interact with the kernel and C library.
- **Three cornerstones of system programming:**
  1. **System calls (syscalls)**: **kernel function invocations from user space** (e.g., `read()`, `write()`). There are **~300 in total, and ~90% are implemented by all architectures**.
  2. **The C library (libc / glibc, "gee-lib-see")**: **wrappers for system calls, threading support**, and basic application facilities.
  3. **The C compiler and linker (toolchain)**: **GCC** (GNU Compiler Collection).
- **You cannot link a user-space application directly with the Linux kernel** (for security and reliability reasons). System calls are invoked with a **trap, typically a software interrupt**. For example, `printf()` calls the C library, which traps into the kernel to write to the terminal.
- **API (Application Programming Interface)**: a **definition** of how one piece of software talks to another **at the source level**. The software providing it is the *implementation*. **APIs ensure *source* compatibility**: the same source can be **recompiled** for x86 or ARM and work. (They do *not* let you copy an executable to another platform and run it.) Example: `printf()`, `strcpy()`. Exception: the ~10% of syscalls that differ between architectures.
- **ABI (Application Binary Interface)**: covers **calling conventions, byte ordering, and register use**. **ABIs ensure *binary* compatibility.** They are **defined and implemented by the kernel and the toolchain** (enforced by the toolchain for a platform). **An ABI defines how an application interacts with itself, with libraries, and with the kernel.** Binary portability requires a matching **hardware architecture, software libraries (especially the glibc version), and compiler**.
- **POSIX** ("PAHZ-icks", **Portable Operating System Interface**): **C APIs for Unix-like operating systems**, started by the **IEEE in the late 1980s to settle the "Unix Wars."**
  - The **Open Software Foundation's Single UNIX Specification (SUS)** was free and eventually incorporated POSIX, which helped adoption.
  - POSIX defines the APIs for **file and directory operations, clocks and timers, semaphores, shared memory, and threads**; headers include `<stdio.h>` and `<stdint.h>`.
  - **Linus Torvalds announced Linux on the Minix mailing list on 1991-08-25.**

### 1.6 Linux Filesystems
- **"Everything is a file"**: much of the interaction with the kernel happens by reading and writing files. **Devices are accessed like files** (e.g., `/dev/ttyUSB0` for a USB serial port) using the same open/read/write/close pattern.
- **Paths**: **absolute** paths start with `/` (from the filesystem root), e.g. `/etc/httpd/httpd.conf`. **Relative** paths have no leading `/` and start from the working directory, e.g. `httpd/httpd.conf`, `usr/bin/env`, `home/user/Documents`.
- **Regular files** are a byte stream with **no structure enforced by the system**. A **file position (offset)** tracks where you are. Files can be **truncated smaller *or larger*** than the original (larger without using disk space). A file **can be opened more than once**, by the same or different processes.
- **Inode (index node)**: a **fixed, small (~128-byte)** metadata record per file. It holds the **timestamps, owner (UID), group (GID), size, mode (permissions), and location of the data on disk**. **It does NOT contain the filename.**
- **Directories** **map human-readable names to inode numbers**. A directory is itself just a file (with its own inode) containing name→inode entries, starting at the root `/`.
- **Why keep filenames out of inodes?** So **multiple names can share the same content without duplicating the inode**. That's what makes links possible.
- **Hard links** map **directly to an inode**, so they **only work within the same filesystem**. A hard link is conceptually **the same as a directory entry**, and **the file isn't deleted until all references are removed**.
- **Symbolic (soft) links / symlinks** are a **regular file whose content is a path**. They **can point anywhere, including other filesystems, and can break**. Resolving one takes an **extra step**.
- **Special files** are **kernel objects represented as files**, mapping devices into "everything is a file":
  - **Character devices**: a **linear queue of bytes** (e.g., a **keyboard**).
  - **Block devices**: an **array of bytes addressable in sectors/blocks** (e.g., a **hard disk**).
  - **Named pipes (FIFOs)** and **sockets**: interprocess communication.
  - *Directories and regular links are **not** special files.* Character and block devices are **both** special files, but **only character devices are byte-addressable**; block devices work on block/sector boundaries.
- **Filesystem types** are tied to storage: **NFS** (network), **ext4** (block device), **FAT** (Microsoft format). Filesystems are **mounted and unmounted** onto the root filesystem.
- **The smallest addressable unit is a block**, which is a **power-of-two multiple of the sector size** (historically **512 bytes**). Why blocks? Fewer address bits: a 1 TB disk needs **34 bits** per byte address, **30 bits** with 512 B blocks, and **28 bits** with 4 KB blocks.

### 1.7 Processes and Threads (Intro)
- A **process** is **executable object code running on hardware**, usually in **ELF (Executable and Linkable Format)**. The **kernel allocates and manages its resources through system calls** (timers, files, hardware access). Each process is identified by a **process ID (pid)**.
- **Each process runs on a virtualized processor and virtualized memory**: it appears to have its **own dedicated, single linear address space and CPU**, while in reality RAM is physically scattered and the CPU is time-shared.
- A **thread** is a **unit of activity within a process**. A process is **single-threaded** (a default hello-world program) or **multithreaded**.
  - Each thread has its **own stack** (local variables and call stack) and its **own processor state / current location**.
  - **All threads in a process share the same memory address space**, so two threads *can* access each other's memory.
- To **share memory between threads**, access it directly **with synchronization**. To **share between processes**, use **IPC (interprocess communication)**.
- **Signals** are **one-way asynchronous notifications**, sent from the **kernel to a process**, from **one process to another**, or from **a process to itself**. Processes set up **signal handlers** (e.g., **Ctrl-C sends SIGINT**). Handlers must only call **async-signal-safe functions**: process code can be interrupted at any moment, and **global variables can introduce unsafe scenarios**.
- **IPC mechanisms**: **pipes, semaphores, message queues, shared memory** (signals are also a simple form of IPC).

### 1.8 Users and Groups
- A user **logs in with a username and password** (authentication). Each user has a **UID**, and **each process runs with the UID of the user running it**. **`/etc/passwd` maps usernames to UIDs.**
- **UID 0 = root** (can do almost anything). *(UID 1 is NOT root.)*
- Each user belongs to **one or more groups**, each with a **GID**. **`/etc/group`** maps group names to GIDs.
- File permission bits by octal value: **owner r/w/x = 0400/0200/0100, group = 040/020/010, others = 04/02/01**.

### 1.9 Error Handling: `errno`
- The C library reports errors through **`errno`** (`<errno.h>`). For example, `fopen()` returns **NULL and sets errno**.
- **ENOENT** (value **2**) = "**No such file or directory**". **`errno -l`** on the command line lists all error names and values.
- Print errors with **`perror("msg")`**, or **`strerror(errno)`** to get the string.
- **Pitfall**: other calls (e.g., **`fprintf`**) **may modify errno** before you print it. **Fix**: call `perror` first, or **save errno to a local variable** right away.
- **errno is thread-safe**: POSIX makes it **thread-local storage**, so each thread has its own copy.

### 1.10 Embedded Linux Toolchain
- A **toolchain** = **compiler + linker + run-time libraries**. Common choices: **GCC or Clang** (the course uses GCC).
- **GCC toolchain components**: **binutils** (binary utilities, including the **assembler and linker**), **GCC** (GNU Compiler Collection), and the **C library** (an API based on POSIX).
- **Setting one up**: (1) manually download, build, and install the components, or (2) **use a build system (Buildroot or Yocto)**. *You probably shouldn't hand-build one*; Assignment 2 does it only to demystify the process.
- **Native toolchain**: runs on the same system as the program it generates. **Cross toolchain**: produces output for a **different architecture** than the host (**cross compiling**).
  - **Why cross compile?** The host is usually **more powerful (faster builds)**, and you **don't want, or can't fit, development tools in the target image**.
- **GNU prefix naming**: `aarch64-none-linux-gnu-gcc` = **CPU** aarch64 (64-bit ARM), **vendor** none (a common set of ARM CPUs), **kernel** linux, **OS** gnu. Use the same prefix for every tool (`…-gdb`, `…-ld`, `…-objdump`).
- **Sysroot**: the **root filesystem of the cross toolchain**, holding files **specific to the *target***. It mirrors a host root filesystem. Find it with **`aarch64-none-linux-gnu-gcc -print-sysroot`**.
  - `lib/` holds **shared objects for the C library** (these go **on the target** at runtime).
  - `usr/lib/` holds **static library archives**.
  - `usr/include/` holds **headers** (e.g., `<stdio.h>`).
  - `usr/(s)bin/` holds **utility programs for the cross toolchain**.
  - **Is the cross-compiler itself in the sysroot? No.** It runs on the host, so it lives in the host filesystem.
- **Other toolchain tools**:
  - `gcc`/`g++` (compilers), `gdb` (debugger), `ld` (linker)
  - **`addr2line`**: converts program addresses to file names and line numbers
  - **`objdump`**: disassembles object files and relates them to source
  - **`strip`**: removes debug information to make binaries smaller
  - **`readelf`**: shows executable details, such as shared-library dependencies and the program interpreter
  - **`file`**: shows the binary's architecture (e.g., "ARM aarch64")
- **Static vs dynamic linking** (gcc always links with glibc):
  - **Static**: all library code is **copied into your executable**.
  - **Dynamic**: linking happens **at runtime**; one shared copy on disk serves all programs.
  - **Use static linking when** you have **few applications or just one (e.g., BusyBox)**, or you must **run before the root filesystem is available** (e.g., at boot, while loading storage drivers).
- **Finding shared-library dependencies**: `readelf -a writer | grep "Shared library"` shows e.g. **libc.so.6**, and `grep "program interpreter"` shows **ld-linux-aarch64.so.1**. **Both must be on the target's root filesystem.**
- **The dynamic linker searches** `/lib`, `/lib64`, `/usr/lib`, `/usr/lib64`, and the directories in **`LD_LIBRARY_PATH`**.

### 1.11 Logging and syslog
- `printf` debugging works for interactive programs, but **headless embedded systems need a logging framework** that stores messages in log files.
- **syslogd / rsyslogd** is a **daemon** that applies a **configuration file** (logs usually go to **`/var/log`**) to messages that applications send with the **`syslog()` API**.
- **Usage**: `openlog(ident, option, facility)`, then **`syslog(priority, fmt, ...)`**, which works like `printf`.
  - **Facilities**: `LOG_USER`, `LOG_DAEMON`, `LOG_LOCAL0`–`LOG_LOCAL7`.
  - **Priorities**: `LOG_ERR`, `LOG_WARNING`, `LOG_INFO`, `LOG_DEBUG`.
- On Ubuntu, routing rules live in **`/etc/rsyslog.d/*default.conf`**, and `LOG_USER` messages land in **`/var/log/syslog`**. (On WSL, start it with `sudo service rsyslog start`.)

### 1.12 Assignments 1 & 2 (what they exercised)
- **A1**: GitHub Classroom setup, Unity unit test, `finder.sh` (counts files and matching lines; exits 1 on bad arguments or a non-directory), and `writer.sh` (creates the path and writes a string; exits 1 on errors).
- **A2**: install the ARM aarch64 cross toolchain and record `-print-sysroot` and `-v` output. Rewrite `writer` in **C using file I/O**, logging with **`LOG_USER`** facility: success as `LOG_DEBUG` ("Writing <string> to <file>"), errors as `LOG_ERR`. Write a **Makefile** with a default target, a `clean` target, and **`CROSS_COMPILE` support**: `make CROSS_COMPILE=aarch64-none-linux-gnu-` (note the trailing dash) prefixes `gcc`. Show `file` output proving an aarch64 executable.

> **Module 1 Cheat-Sheet Candidates:** octal permissions (r4 w2 x1); `$0 $1 $# $?`; API = source compatibility, ABI = binary compatibility (calling convention, byte order, registers; defined by kernel and toolchain); what an inode contains (no filename); hard vs soft links; special file types; UID 0 = root, /etc/passwd, /etc/group; ENOENT = 2; errno is thread-local; sysroot directories; static linking use cases; man sections 2 vs 3.

---

<a id="module-2"></a>
## Module 2 — File I/O, Processes, Basics of Embedded Linux

### 2.1 Linux File I/O Basics
- Files are accessed through **file descriptors (fds)**, which are plain **integers**. The **kernel keeps a per-process file table** indexed by fd.
- **Every process starts with 3 open fds**: **stdin = 0** (`STDIN_FILENO`, terminal input), **stdout = 1** (`STDOUT_FILENO`, terminal display), **stderr = 2** (`STDERR_FILENO`, terminal display, but separately redirectable).
- **System calls**: `open()` (possibly with `O_CREAT`), `read()`, `write()`, `close()`.
- **`fopen`/`fread`/`fwrite` vs `open`/`read`/`write`**:
  - The **"f" functions are buffered I/O** from `<stdio.h>`. They return a **`FILE*`**, live in **man section 3**, and provide **platform-independent user-space buffering** that **reduces the number of system calls**.
  - The **non-"f" functions are the actual system calls**. They return an **int fd** and live in **man section 2**.
  - Under the hood, glibc's `fopen()` eventually calls **`openat()`**.
  - Using the "f" versions in assignments is fine; the course covers the syscalls to show the user/kernel boundary.
- **`open(path, flags[, mode])`** maps a pathname to an fd.
  - **flags** set the access mode (**`O_RDONLY`, `O_WRONLY`, `O_RDWR`**) plus options (`O_CREAT`, `O_APPEND`, `O_TRUNC`, `O_SYNC`, `O_NONBLOCK`…).
  - **mode** gives the **permissions for a newly created file** and is **only meaningful with `O_CREAT`**: `S_IRWXU` (user rwx), `S_IRWXG` (group rwx), `S_IROTH` (others read), and so on.

### 2.2 Reading, Writing, umask, strace
- **`read(fd, buf, count)`** returns the **number of bytes read, `0` at EOF, or `-1` on error (and sets errno)**. A return value **less than `count`** means no more bytes are available *now* (end of file, or interrupted by a signal). **Always check the return value against what you asked for**, not just `!= -1`.
- **`write(fd, buf, count)`** returns the **number of bytes written** or -1. A *partial write* is possible, so compare the result with `count` (or `sizeof(x)`). **Don't hard-code `4`**: `sizeof(unsigned long)` is 8 on 64-bit systems.
- **umask** is a **per-process restriction on the permissions of newly created files**: final mode = `mode & ~umask`.
  - Ubuntu's default umask of **0002 clears others' write**.
  - **umask 022** clears group and others write.
  - **umask 0003** clears others' write *and* execute.
- **strace** logs **every system call** a program makes and what the kernel returns, like **a protocol analyzer for the user/kernel boundary**. **It needs no source code** and works on any binary. Example: `openat(...) = 3`, then `read(3, "", 1) = 0`.
- **Non-blocking I/O**: open a FIFO or pipe with **`O_NONBLOCK`**. When no data is available, `read()` returns -1 with **`errno == EAGAIN`** instead of blocking. This lets your thread **do other work while waiting**. You do **not** need `O_NONBLOCK` when the thread has nothing else to do.
- **Synchronized I/O**:
  - **A successful `write()` does NOT mean the data is on disk.** It is **in a kernel buffer**, because **disks are slow** (seeks take tens to hundreds of ms).
  - **Risk**: **unclean shutdown** (e.g., power loss) leaves the disk out of step with what your program reported.
  - **Mitigation**: **`fsync(fd)`** (syncs data and metadata for **one fd**; the best choice for a single file), **`fdatasync(fd)`** (data only), **`O_SYNC`** on open, or configuring disk write caching.
  - **Avoid `sync()`** for performance reasons: it syncs **everything system-wide**.

### 2.3 Positional I/O, Sparse Files, Multiplexed I/O
- **`lseek(fd, pos, origin)`**:
  - **`SEEK_SET`**: absolute offset from the start of the file.
  - **`SEEK_CUR`**: relative to the current position.
  - **`SEEK_END`**: relative to the end of the file. **`lseek(fd, 0, SEEK_END)` moves to the end of the file**; `lseek(fd, 0, SEEK_CUR)` doesn't move (and returns the current position).
- **`pread()`/`pwrite()`** take an absolute offset with **no origin argument** (always equivalent to SEEK_SET) and **don't change the file position**. That **avoids race conditions when several threads share an fd**.
- **Sparse files**: **seeking past EOF is allowed**. `lseek(fd, 1000000000, SEEK_END); write(fd, &v, sizeof v);` extends the file by ~1 GB, but **Linux creates a "hole" that uses no disk space**, and **reading the hole returns zeros**. **`ls -lh` shows ~954 MB, but `du -sh` shows 4 KB.**
- **Handling input from several fds at once**:
  1. one thread per fd doing **blocking reads**,
  2. **`O_NONBLOCK` + polling** (wastes cycles), or
  3. **multiplexed I/O (better than polling)**: **sleep until any fd in a set is ready**, watching several fds **in a single call**. Options are **`select()`, `pselect()`, `poll()`**; **`poll()` is preferred** for its flexibility.

### 2.4 Process Management (LSP Ch. 5)
- A **process** is a **running program**: a **binary image in memory**, a **virtualized memory instance**, **open files**, an **associated user**, and **one or more threads**.
- The **pid** uniquely identifies a process **at any single point in time**. **The kernel reuses pids** after a process terminates.
- **Special pids**:
  - **pid 0 = the idle process**: **one per processor**; handles power control.
  - **pid 1 = the init process**: **initializes the system, starts services, and launches the login program**. Other processes are started based on init's work.
- **`execl()`** (and the exec family) **replaces the current process image** with a new program. **It does not return on success**, so any code after a successful `execl()` (e.g., a `printf`) **never runs**; its return value is only useful on failure (-1). Everything is replaced **except the pid, priority, and owning user and group**.
  - **Family members** `execlp`, `execle`, `execv`, `execvp`, `execve` differ in **how arguments are passed** (list "l" vs vector "v"), **whether the environment is specified** ("e"), and **whether PATH is searched** ("p").
- **`PATH`** is a **colon-separated list of directories searched for executables** named without a path (`which ls` → `/usr/bin/ls`).
  - **PATH injection**: someone modifies your PATH so `/home/hacker/ls` runs instead of `/usr/bin/ls`.
  - **Solution: avoid relying on PATH and use absolute paths**, which is easy in embedded work because you build the whole root filesystem.
  - So the **"p" variants (`execlp`, `execvp`) are *more* susceptible to PATH injection, not safer.**
- **`fork()`** **creates a new child process** running the **same image**, with a **new pid**. **The parent keeps running.** Check the return value: **0 in the child**, the child's pid in the parent, -1 on error.
  - Fork is often used to **start daemons**.
  - It uses **copy-on-write**: memory (local variables, kernel resources such as open file handles) is **shared until either side modifies it**.
  - **fork and exec are NOT similar**: fork creates a new process and keeps the original; exec *replaces* the current process.
  - Tip from A3: call `fflush(stdout)` before `fork()` to avoid duplicated printf output.
- **Zombie process**: a **child that has died before the parent was informed** (the parent hasn't called `wait()`). It stays in the zombie state until reaped.
- **`wait()`** lets the parent **collect a terminated child's status**: the **reason it terminated** (e.g., a signal) and its **return code**. **`waitpid()`/`waitid()`** add options such as waiting for a specific child.
- **Launching a new program *and* waiting for it** = **`fork()` + `exec()` (in the child) + `wait()` (in the parent)**.
- **`system(cmd)`** does **fork + exec + wait for you**, i.e. it **launches a new process and waits for it to terminate**. **Drawbacks**: it **uses PATH (injection risk)** and **runs through the shell, which expands input** such as `$HOME`.
- **Summary**: **exec = replace** the process; **fork = launch a child and continue the parent**; **system = launch and wait**.

### 2.5 Processes, Permissions, Sessions, Daemons
- **Multiple user and group IDs per process**:
  - The **real UID/GID** is **who started the process** (`getuid()`, `getgid()`).
  - The **effective UID/GID** is **who the process is currently running as** (`geteuid()`, `getegid()`); this is usually the most useful one.
  - **The effective UID can be changed after the process starts** with `setuid()`/`setgid()`.
- **Doctrine of least-privileged rights**: a process should run with the **minimum level of rights possible**. Why? It's **less susceptible to exploits** and **less susceptible to your own errors** (e.g., `rm -rf ${MISTYPEDD_VAR}/` run as root). *Using the maximum rights is NOT a best practice.*
- **A process group is a collection of processes; a session is a collection of process groups.**
  - A session is **associated with a controlling terminal (tty, "TeleTypewriter")**, the device for terminal I/O (keyboard, login, command I/O). **A session is created for the login shell on a tty** and can include many processes.
  - **Each session has exactly one foreground process group**. That's the one **Ctrl-C interrupts**: SIGINT goes to **all processes in the session's foreground process group**.
  - There are **zero or more background process groups**. **Appending `&`** to a shell command puts it in a background process group.
- **`ps`** shows process status, e.g. `ps aux | less`. **Bracketed names `[kthreadd]` are kernel threads (they always run as root).**
- **Daemons** (pronounced "dee-men" or "day-men") are processes that **run in the background and do NOT connect to a controlling terminal**. They're typically **started at boot**, run as **root or a special user**, often have names **ending in "d"** (`sshd`, `crond`), and often run **as a child of init**. *A daemon is NOT associated with a terminal.*
- **How to create a daemon (the classic LSP method):**
  1. **`fork()`**; the **child becomes the daemon**.
  2. **`exit()` in the parent**, so the daemon's grandparent (init) can continue.
  3. **`setsid()`**: **creates a new session and process group**, so there's **no controlling tty**.
  4. **`chdir("/")`**: change to the **root** directory, so the daemon doesn't keep a directory (or a filesystem being unmounted) "in use".
  5. **Close all open file descriptors.**
  6. **Redirect stdin, stdout, and stderr to `/dev/null`**. If you didn't, output would go to the terminal it was started from. **Not to a log file**: a file could block unmounting its filesystem. **Report status and errors with syslog instead.**
- (Newer approaches, such as systemd, exist and people disagree on daemon design. The course follows the book's classic method.)

### 2.6 Basics of Embedded Linux — Kernel, QEMU, Kernel Build, Root Filesystem
**Linux kernel overview (MELP Ch. 4)**
- **Main jobs**: **manage resources, interface with hardware, and provide an API for user-space programs**. It's typically tailored to specific hardware to some extent.
- The kernel runs at **a high CPU privilege level**, with **access to all memory addresses and all CPU registers**. Least-privilege doesn't exist in kernel space, so be careful.
- **Entry into the kernel**: **system calls from user space** (a software trap/interrupt) or **interrupts from hardware**.
- Linus Torvalds' tree is on **git.kernel.org**. The kernel has ~**57,000 files and 14 million lines of code**.
- **Kernel source and licensing (GPLv2)**:
  - Not all kernels come from kernel.org; **SoC vendors** ship their own.
  - **A closed-source Linux kernel is not legal.**
  - **You must make the source available even if you didn't change it.**
  - A **custom kernel module "probably" must be shared**. Get legal review if you don't want to share, and don't expect community help without source.
- **Source directories**:
  - `arch`: architecture-specific code
  - `Documentation`
  - **`drivers`**: hardware support and loadable modules
  - `fs`: filesystems
  - `include`: headers
  - `init`: system initialization
  - **`kernel`**: core functions (scheduling, locking, timers)
  - **`mm`**: memory management
  - `net`: network protocols
  - `scripts`, `tools`

**Elements of embedded Linux (MELP Ch. 1)**
1. **Toolchain**: the compiler and tools that create code for your device.
2. **Bootloader**: **initializes the board and loads the kernel**.
3. **Kernel**: manages system resources and interfaces with hardware.
4. **Root filesystem**: the **libraries and programs that run after the kernel finishes initializing**.

- **Hardware support**: **30+ architectures, all at least 32-bit**. Most have an **MMU** (exception: **uClinux**). You need **~16 MB RAM** and **~8 MB+ non-volatile storage**.
- **QEMU** **emulates a processor architecture and a board** built on it, which lets you develop before hardware exists. `qemu-system-aarch64` emulates 64-bit ARM. Example command:
  `qemu-system-aarch64 -m 256M -nographic -M virt -kernel /path/to/Image -append "rdinit=/bin/sh" -initrd /path/to/initramfs.cpio.gz`
  - **`-m 256M`**: 256 MB of RAM.
  - **`-M virt`**: emulate the "virt" board.
  - **`-kernel`**: boot this kernel image directly, **bypassing the bootloader**.
  - **`-append`**: the kernel command line (console, and **init program = /bin/sh**).
  - **`-initrd`**: the **root filesystem as a ramdisk** (a cpio archive).
  - `-dtb` (older example) supplies the device tree.

**Building the kernel**
- **Kconfig** is the **text format that controls kernel configuration** (also used by BusyBox). Each entry has a `config` label, a type (bool…), a default, and help text.
  - **`make menuconfig`** is a **terminal-based graphical tool** driven by the Kconfig files.
  - **Changes are stored in `.config`** at the kernel source root (a hidden file), e.g. `CONFIG_DEVMEM=y`.
- **Three modes for most drivers**:
  - **off**: not built.
  - **on (`*`, `=y`)**: built into the kernel.
  - **module (`M`, `=m`)**: **loadable at runtime** (modprobe).
- **defconfig**: a file that sets up `.config` with **a set of configurations known to work for a specific hardware platform**, typically provided by the board manufacturer. Use it as your starting point. For QEMU's virt board on arm64, use plain **`make defconfig`**.
- **Kbuild makefiles**: **`obj-y` = built in, `obj-m` = module**. `obj-$(CONFIG_TTY_PRINTK) += ttyprintk.o` expands based on `.config`.
- **Build artifacts**:
  - **`vmlinux`**: an **ELF binary, possibly with debugging symbols**.
  - In `arch/$ARCH/boot`: **`Image`** (binary vmlinux), **`zImage`** (compressed), **`uImage`** (compressed with a **U-Boot header**).
- **Assignment 3 build sequence**:
  1. `make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- mrproper`: **deep clean, removes `.config`**.
  2. `… defconfig`: configure for the virt board.
  3. `make -j4 … all`: build the kernel image (`-j4` = 4 parallel jobs).
  4. `… modules`: build modules.
  5. `… dtbs`: build the device tree.

**Root filesystem (MELP Ch. 5)**
- Booting a kernel without a rootfs gives a **kernel panic: "unable to mount root fs"**.
- **Rootfs options**:
  - **initramfs**: **a set of files extracted into RAM**, typically **used to set up your real root filesystem** (e.g., load the modules needed to reach storage). A3 uses it for simplicity.
  - **A block device** given on the kernel command line with **`root=`**.
- **Minimal rootfs contents**:
  - **init** (starts the system, usually scripts), a **shell**, **daemons**
  - **shared libraries**, **config files (`/etc`)**, **device nodes (`/dev`)**, the **`/proc` and `/sys` pseudo-filesystems**
  - **kernel modules (`/lib/modules/<kernel version>`)**: modules are built for one specific kernel version.
- **Where does the kernel expect files in the rootfs? It doesn't!** It only needs the **init program named on the kernel command line**. The kernel and rootfs are deliberately decoupled.
- **Programs' expectations are defined by the Filesystem Hierarchy Standard (FHS)**:
  - **`/bin`**: programs for all users, used at boot
  - **`/dev`**: device nodes
  - **`/etc`**: system configuration
  - **`/lib`**: shared libraries
  - **`/proc`, `/sys`**: the proc and sysfs filesystems
  - **`/sbin`**: programs for the system administrator, used at boot
  - **`/tmp`**: temporary files, **may be deleted on boot**
  - **`/usr`, `/usr/bin`, `/usr/sbin`**: additional programs, libraries, and utilities
  - **`/var`**: **files modified at runtime that are retained after boot** (e.g., `/var/log`)
- **Building a rootfs**:
  1. **Create the directory tree** with mkdir.
  2. **BusyBox**: a **single binary that implements many essential Linux programs**. **Symbolic links** (e.g., `/bin/cat → busybox`) tell it which program was requested. Not every option of every command is supported. Build it with `make distclean`, `make defconfig`, `make ARCH= CROSS_COMPILE=`, then **`make CONFIG_PREFIX=/path/to/rootdir … install`**, which also creates the symlinks.
  3. **Add the shared libraries from the toolchain sysroot.** Use `${CROSS_COMPILE}readelf -a busybox` to find the **program interpreter** (place it in **`lib/`**) and the **shared libraries** (place them in **`lib64/`** for a 64-bit arch).
  4. **Device nodes** with **`mknod <name> <type> <major> <minor>`**: **`/dev/null` = char, major 1, minor 3** (`mknod -m 666 dev/null c 1 3`); **`/dev/console` = char, major 5, minor 1**.
  5. **Make the contents owned by root** (`--owner root:root`), because **your user account doesn't exist on the target**.
  6. **Package it**: `find . | cpio -H newc -ov --owner root:root > initramfs.cpio`, then `gzip`.
- **Using the rootfs on the target**: a **ramdisk** (an image loaded into RAM by the bootloader; A3 uses this, with QEMU acting as the bootloader), a **disk image** (SD card), or **NFS**.
- **A3 Part 1**: implement `do_system()` (`system()`), `do_exec()` (fork + execv + wait), and `do_exec_redirect()` (redirect stdout to a file).
- **A3 Part 2**: the `manual-linux.sh` script (default outdir `/tmp/aeld`) clones the kernel and builds the Image, then builds the rootfs, adds the cross-compiled writer and finder scripts, creates `initramfs.cpio.gz`, and boots with `start-qemu-terminal.sh`.
  - **Troubleshooting**: "finder.sh not found" means the script uses **`#!/bin/bash`, but BusyBox only provides `/bin/sh`**. "VFS: Unable to mount root fs" means a bad cpio (it must be created from the target rootfs's root directory).

> **Module 2 Cheat-Sheet Candidates:** fd 0/1/2; f-functions = buffered (man 3) vs syscalls (man 2); read/write return values; umask math; strace; `O_NONBLOCK` + EAGAIN; write ≠ on disk → fsync(fd); lseek origins; sparse files (ls vs du); poll preferred; pid 0 idle / pid 1 init; exec replaces, fork = child + continue, system = fork+exec+wait; PATH injection ("p" variants not safer); real vs effective UID; least privilege; foreground process group gets SIGINT; daemon steps (fork, parent exit, setsid, chdir("/"), close fds, /dev/null); 4 elements of embedded Linux; Kconfig off/on/module; obj-y/obj-m; vmlinux/Image/zImage/uImage; FHS directories; BusyBox symlinks; mknod null 1,3 / console 5,1.

---

<a id="module-3"></a>
## Module 3 — Threads & Synchronization, Buildroot, Signals

### 3.1 Threads and Multithreading (LSP Ch. 7)
- **Binary**: a **dormant program residing on storage**, ready to execute (e.g., an ELF file).
- **Process**: an **OS abstraction representing a binary in action**. It contains one or more threads (one = single-threaded, more = multithreaded) and has **dedicated virtualized memory**.
- **Thread**: a **unit of execution within a process**, with its own **virtualized processor, stack, and program state**. Threads **share the process's virtualized memory**. *(Threads are NOT like processes in having their own unique view of memory.)* The **thread scheduler** time-slices threads from all processes across the available CPUs.
- **Why use multithreading?**
  - **An alternative to state machines.** Instead of saving state and coming back after an I/O wait, a dedicated thread waits while another keeps working.
  - **Parallelism**: it scales your process across multiple processors. *State machines can't give that parallelism.*
  - **Responsiveness**: **long-running operations and blocking I/O don't belong in UI threads**. The fixes for a freezing UI are to **move blocking work to a dedicated worker thread**, or to **use non-blocking I/O with a state machine**.
  - **Context switching**: a **thread switch is cheaper than a process switch**, because no memory-space swap is needed.
  - **Memory savings**: threads share one memory space.
- **Why NOT use multithreading?** **Complexity** in writing, understanding, and debugging, so worries about that are justified. **Alternatives**: non-blocking I/O, or multiple processes (which need IPC instead of shared memory).
- **Concurrency vs parallelism**:
  - **Concurrency**: **the ability of two or more threads to execute in overlapping time periods**. It's a programming pattern that works even on one CPU via time slicing.
  - **Parallelism**: **the ability to execute two or more threads *simultaneously***. It **requires multiple processors (hardware)**, so **it's impossible on a single-processor/single-execution-unit platform**.
- **Threading models**:
  - **Thread-per-connection**: **each unit of work (request or connection) gets its own thread**, which does **blocking I/O**. It **may need more threads than CPUs** for maximum parallelism and **breaks down when requests become very numerous**. Example: **Apache**.
  - **Event-driven threading**: **asynchronous I/O, callbacks, and an event loop** (multiplexed I/O hands callbacks to waiting threads). Examples: **NGINX, Node.js**. **No waiting happens in threads, so there's no reason to have more threads than CPUs**; adding threads beyond the CPU count gives no improvement. **The author suggests trying this model first.**
- **Amdahl's Law**: the theoretical speedup is limited by the fraction of execution time that benefits from the improvement. It ties back to event-driven threading: once every CPU has a non-blocking thread, there is nothing left to parallelize.

### 3.2 Synchronization
- **Race condition**: **different program behavior depending on which thread gets there first**, i.e. **unsynchronized access to a shared resource causes erroneous behavior**. The shared resource can be hardware, a kernel resource, or memory. Memory races (**data races**) are the most common form.
- **Critical region**: code that **needs synchronization**. **Races are eliminated by synchronizing thread access to critical regions**, which **makes the critical region atomic**.
- **Race conditions happen even without parallelism**: on a single CPU, **concurrency (preemption) still causes them**. (Example: `s->x++` is a load, add, store sequence; a thread switch between the load and the store loses one increment.)
- **Atomic operation**: **indivisible, can't be interleaved with other operations, appears instantaneous**. **`++` is NOT atomic.**
- **Mutex (mutual-exclusion lock)**: `pthread_mutex_init()`, then **`pthread_mutex_lock(&m)` → critical section → `pthread_mutex_unlock(&m)`**. In the bank `withdraw()` example, the lock protects only the shared account struct. `disburse_money(amount)` needs no lock because it touches no shared memory.
- **Deadlock**: **two threads each waiting for the other**, or **one thread blocking on a mutex it already holds**. **How to avoid it**:
  - **Lock data, not code**: associate each lock with a specific data structure, e.g. put the mutex inside the account struct.
  - Question whether multiple locks are really needed. *Adding more mutexes is NOT the fix.*
  - When several locks are needed, **always acquire them in a fixed order and release them in the opposite order**.
- **pthreads API**:
  - **`pthread_create(&thread, attr, start_routine, arg)`**: `attr` is NULL for defaults or sets stack size, scheduling, and detached state; `thread` receives the thread ID; `arg` is passed to `start_routine`.
  - **A thread ends when** it **returns from its start routine**, **calls `pthread_exit()`**, or **another thread calls `pthread_cancel()`**. **When the process exits, all its threads are killed.**
  - **`pthread_join(thread, &retval)`**: **blocks until that thread terminates**.
  - **Joinable threads consume system resources until joined**, so **forgetting to join causes a memory leak**. **Detached threads** (`pthread_detach()` or the detached attribute) **can't be joined** and clean up on their own.
- **Scoped locks (C++ RAII, "Resource Acquisition Is Initialization")**: the constructor locks the mutex and the destructor unlocks it when the object goes out of scope, so you can't forget to unlock.
- **A4 Part 1**: implement `threadfunc()` and `start_thread_obtaining_mutex()`. The thread waits, obtains the mutex, waits, then releases it.

### 3.3 Buildroot
- An embedded system needs a **toolchain, bootloader, kernel, and root filesystem**. Ways to produce them:
  - **A build system** (**Buildroot, Yocto, OpenWRT**)
  - **Roll Your Own (RYO)**, e.g. crosstool-NG plus a hand-built rootfs as in A3
  - **An existing distribution** (Debian, Ubuntu)
- **Why a build system over RYO?** "Why reinvent the wheel?" Is your project really so unique it can't reuse others' work? **What is your strategy for keeping up with security patches** (CVEs) across every package? RYO is good for learning, not for production.
- **Why a build system over a distribution?**
  - A distribution's **goals are different: it's focused on users and desktops**.
  - **Upgrades assume someone is interacting with the system** (prompts for configuration, time zone).
  - **Package management isn't designed for embedded development.**
  - **Reducing image size is hard.**
  - Generating a production image takes **customization work** (distributions install, then add packages; embedded wants one flashable image).
  - **Different upgrade paths (v1→v3 vs v1→v2→v3) can leave subtly different filesystems.**
- **What a build system does**:
  1. **Downloads source** for common packages from upstream.
  2. **Applies patches** (for cross-compilation and arch-dependent bugs).
  3. **Builds** the components.
  4. **Assembles the rootfs in a staging area.**
  5. **Creates image files.**
- **Other build-system features**: **add your own packages** (apps or drivers, often proprietary; e.g. a private repo via SSH key), **select system profiles** (e.g., with or without graphics, which shrinks the image), and **track open-source licenses** to help with license compliance.
- **Buildroot**:
  - A **builder of root filesystems for embedded devices**: a **collection of Linux packages with build instructions, for both host and target**.
  - **Started in 2001**; **focuses on simplicity** (simpler than Yocto, which is why the course starts with it).
  - **Licensed GPLv2**: you're expected to share changes to Buildroot itself.
  - It **leverages make files and Kconfig/menuconfig**, the same configuration mechanism as the kernel, BusyBox, and crosstool-NG. You need its mandatory host packages, plus **ncurses (libncurses5-dev)** for menuconfig.
- **Buildroot packages**:
  - Build and install instructions written in **GNU Make syntax**. Built-in packages live in the **`package/` directory**.
  - **br2-external trees** keep your own packages **outside the Buildroot tree**. Use them **for packages you can't share upstream (proprietary)**. Packages can fetch source from **git repositories** (for the assignments, use the **SSH URL**, not HTTPS).
  - **Every package needs at least two files**:
    - **`Config.in`**: Kconfig code adding the package to menuconfig.
    - **`<package_name>.mk`**: the make recipe.
  - **Package definitions do NOT contain the source code**, only the **instructions to get the code, compile it, and add it to the rootfs**.
  - **`.mk` details**:
    - `$(eval $(generic-package))` knows how to **cross-compile using `<PKG>_XXX` variables** (`LIBFOO_VERSION`, `_SITE`, `_SITE_METHOD = git`, `_BUILD_CMDS`, `_INSTALL_TARGET_CMDS`).
    - `$(TARGET_CONFIGURE_OPTS)` sets the cross-compile variables.
    - **`$(INSTALL) -m 0755 -D $(@D)/file $(TARGET_DIR)/usr/bin`** installs a file: `$(@D)` is the build directory, `$(TARGET_DIR)` is the staging rootfs, and `install` is "a glorified cp" that also sets the mode.
  - **base_external setup**: `base_external/Config.in`, `external.mk`, `external.desc` (external name **`project_base`**).
- **Git submodules**:
  - **Add one git repository as a subdirectory of another**, keeping the projects separate while one references the other. It **keeps your project-specific changes out of the shared project** and makes **retargeting to a new Buildroot release easy**.
  - `git submodule add <url>`, check out the release tag (e.g., **2024.02.13**), then `git add buildroot` to **record the specific commit hash**.
  - GitHub shows the submodule as `buildroot @ <hash>`.
- **A4 workflow**:
  1. `./build.sh` (first run): creates `.config` from **`qemu_aarch64_virt_defconfig`**.
  2. **`./save-config.sh`**: runs `make savedefconfig` into **`base_external/configs/aesd_qemu_defconfig`**. Commit it, or others (and the autograder) won't get your configuration.
  3. `make menuconfig` → External options → select **aesd-assignments** (adds `BR2_PACKAGE_AESD_ASSIGNMENTS=y`), then save the config again.
  4. `./build.sh` again: the full build (takes hours).
  5. `./runqemu.sh` (from `board/qemu/aarch64-virt`), and `./clean.sh` (`make distclean`).
  6. Add **dropbear** (SSH; search for it in menuconfig with **`/`**), set the root password to "root", and save the config.
  7. **SSH via host port 10022 → guest port 22** (`hostfwd`): `ssh -p 10022 root@localhost`. **`scp -P 10022`** uses a capital P. Exit QEMU with **Ctrl-A, then X** (or Ctrl-A C, then q).
  8. Optional speedups: **`BR2_DL_DIR`** (e.g., `${HOME}/.dl`) keeps downloads outside the tree so they survive distclean and can be shared; **enable the compiler cache**.
- **Buildroot tips**:
  - **`cd \`dirname $0\``**: `$0` is the script's full invoked path and `dirname` strips the filename, so the script `cd`s into its own directory.
  - **Single-package rebuild**: `make aesd-assignments-rebuild` or `make aesd-assignments-reconfigure`.
  - **Local builds**: `make AESD_ASSIGNMENTS_OVERRIDE_SRCDIR=/path/to/local aesd-assignments-rebuild` ignores the git URL and builds from a local directory. **Don't do a clean build every time you change something.**
  - **GNU screen** keeps long builds alive if your SSH session drops: `screen -S buildroot-build`, then Ctrl-A d to detach, `screen -r buildroot-build` to reattach.
  - **ssh-agent** stops the build stalling at your package waiting for a key passphrase: `` eval `ssh-agent` `` then `ssh-add ~/.ssh/key`.
  - **Avoid spaces in paths.**
  - **When is a full rebuild needed?** "It is the responsibility of the user to know." Examples: **target architecture or toolchain changes** (e.g., glibc → musl), **a package is removed**, some cases of **adding packages that reference other packages**, and **changed package sub-options**. `./clean.sh && ./build.sh` is a **troubleshooting step, not for incremental changes**.
  - **`output/target`** holds the "almost complete" rootfs content ("This is not your root filesystem": permissions may differ). Use it for sanity checks; you can delete files there to avoid a full rebuild.
  - **Don't check binaries into the repo** (use .gitignore): Buildroot sees the object file and won't rebuild it, leaving a wrong-architecture binary.
  - **Use `$(CC)` in your Makefile, not a hard-coded `gcc`**, so Buildroot can cross-compile ("writer not found" in QEMU usually means this went wrong).
  - **Debugging strategies**: put a deliberate error in a `.mk`/`Config.in` file to prove it's being read; examine every character side by side with a working example; test natively first; start early (the long build-system tasks come first); and step away for a while.

### 3.4 Signals (LSP Ch. 10)
- **Signals are software interrupts for handling asynchronous events**: events outside the program (Ctrl-C), events from the program or kernel (divide by zero), and **a rudimentary IPC method**. **Both the event and the handler are asynchronous** with respect to program execution.
- **Signal lifecycle**: (1) **raised** (sent or generated) → (2) **stored by the kernel** → (3) **handled by the kernel based on the process's request**.
- **Handling options**:
  - **Ignore**: possible for most signals, **except SIGKILL and SIGSTOP**.
  - **Catch and handle**: the kernel **suspends the process's execution (including other running signal handlers!)** and **jumps to the registered function**. SIGINT and SIGTERM are commonly caught.
  - **Default action**: often **terminate**, sometimes with a **core dump** (a capture of the process's memory). In the man page: **Term** = terminate, **Ign** = ignore, **Core** = terminate and dump core, **Stop** = stop.
- **Common signals**:
  - **SIGABRT**: from **`assert()`**; terminates and generates a core file.
  - **SIGHUP**: "hang up"; **often used to make a daemon reread its config files**.
  - **SIGINT**: **Ctrl-C**.
  - **SIGKILL**: **cannot be ignored or caught**; unconditionally terminates the process.
  - **SIGSEGV**: **segmentation fault** (null pointer or **invalid memory access**); default action is **terminate and generate a core file**.
  - **SIGTERM**: **graceful termination**; the process can catch it and clean up (the "nice" way to stop a daemon).
  - **SIGSTOP**: **unconditionally pauses** the process (can't be ignored).
  - **SIGALRM**: sent when an `ITIMER_REAL` timer or `alarm()` expires.
- **Core dumps**: capture the state of a failing program when it terminates, for analysis with **gdb**, which points straight to the faulting line. Ubuntu needs setup first (core pattern / apport, `apport-unpack`).
- **Troubleshooting SIGSEGV**: look for **null pointer accesses**, **uninitialized pointers being read or written**, and **get a stack trace with gdb** (core file). *Uninitialized integers or divide-by-zero don't cause SIGSEGV; divide-by-zero raises SIGFPE.*
- **`signal()` vs `sigaction()`**: the man page **recommends `sigaction()`**, because `signal()` behavior varies across versions. sigaction is a **POSIX** alternative (not in the original C standard library) with **better signal management**:
  - It can **block new signals during the handler**.
  - It can **retrieve state information**.
  - **In simple cases you only need to set `sa_handler`.**
  - **The function of `sigaction()` is to *register* for a signal.**
- **`pause()` waits for a signal.**
- **`kill(pid, sig)`** sends **any** signal (not just SIGKILL). It's both a system call and a shell command (`kill -TERM 31236`), and only works on **processes owned by the same user, or when the sender is root**.
- **`sigqueue()`** sends a **payload with a signal** (rudimentary IPC).
- **On fork, the child inherits the parent's signal actions.** **`exec()` resets all signals to their default actions, except signals the parent ignored**, which stay ignored.
- **Signal handlers run in the *existing* thread**: it's **not a thread switch or a new thread**. That limits what a handler may call:
  - It may only call **async-signal-safe** functions: they must be **reentrant *and* block signals at any point that could cause inconsistencies**. Check the table of allowed functions.
  - **`malloc`, `free`, anything that calls them, and stdio functions such as `printf` are NOT safe.**
  - **Must not manipulate static data that's also used outside the handler** (e.g., `strsignal()`).
  - **Should touch only stack-allocated data or data provided by the caller.** **Setting a global flag is acceptable**: set it in the handler and read it outside.
- **Handler best practices**: **minimize global data access** (confirm any access is async-signal-safe), **save and restore errno**, and **call an absolute minimum of functions, only from the approved list**.
- **Avoid calling in a handler**: non-reentrant functions and functions that call non-reentrant functions. (Functions touching only local stack data, or reentrant functions, are fine.) **Troubleshooting a crashing SIGALRM handler**: **remove global variable access** and **review the functions it calls to be sure all are reentrant**. Local variable access is fine.
- **Why not use signals?** They're an "**old, antiquated mechanism for kernel-to-user communication**", and **signal-safety bugs are easy to introduce and hard to track down**. You still must handle them for important notifications, **especially termination**. They're not the best IPC mechanism.

> **Module 3 Cheat-Sheet Candidates:** binary vs process vs thread definitions; concurrency (overlapping time) vs parallelism (simultaneous, needs multiple CPUs); thread-per-connection (Apache) vs event-driven (NGINX/Node, ≤ #CPUs threads); races happen even on 1 CPU; `++` not atomic; lock data not code; lock order; join or detach (else memory leak); 3 ways threads end; Buildroot = 2001, simplicity, GPLv2, Kconfig; package = Config.in + .mk (no code); br2-external for proprietary; build system 5 steps; full-rebuild cases; `-rebuild` / `OVERRIDE_SRCDIR`; signal lifecycle; SIGKILL/SIGSTOP can't be ignored; SIGHUP reread config; SIGSEGV + core; sigaction registers, pause waits, kill sends any signal; fork inherits / exec resets (except ignored); async-signal-safe rules.

---

<a id="module-4"></a>
## Module 4 — Time, Sockets, Embedded Socket Server

### 4.1 Time (LSP Ch. 11)
- **Three time measurement types**:
  - **Wall time**: the **actual real-world date and time**, used for user interaction ("what time is it?", "when did X happen?"). It's **localized** (time zone, daylight saving time), **may be saved and restored with a battery-powered hardware clock (RTC)**, and is **updated with NTP (Network Time Protocol)**. Those two are how a system keeps wall time correct after a long power-down; *monotonic time does not help with that*.
  - **Process time**: **time spent executing on a processor** (i.e., the execution time of a process). It **can be less than wall time** because of multitasking, or **more than wall time** when running on multiple processors.
  - **Monotonic time**: **always increasing**, like **system uptime**.
- **Measurement formats**:
  - **Relative time**: relative to some reference such as now, e.g. "**5 seconds from now**", "**10 minutes ago**", "10 mins". **Typically use monotonic time for these**, because changing the time reference (NTP, the user, DST) would be a problem.
  - **Absolute time**: **not relative to a benchmark**; a specific calendar point, e.g. **"12:34:56 UTC 3rd August 2019"** or "Sep 24 2019 3:00 PM MDT". **Linux tracks it relative to the epoch: midnight, Jan 1, 1970 (UTC).** ("100 Hz" is a frequency, not a time.)
- **The software clock**: the kernel's **system timer** fires at a fixed interval and increments elapsed time by one **"tick" or "jiffy"** each interval.
  - The **jiffies counter is 64 bits**.
  - **HZ** = the system timer frequency. **It's platform/architecture dependent** (historically 100–1000 Hz on x86), so **ticks are NOT the same frequency on all platforms**.
  - Jiffy rollover is ~5.8 billion years away (2^64 × 0.004 s).
  - **The POSIX time APIs convert for you, so user space doesn't need HZ.**
- **`time_t`** (`typedef long time_t;`) = **seconds since the epoch**. **On 32-bit systems it overflows on Monday, January 18 (19 UTC), 2038**: the **Y2038 problem**, a second Y2K.
- **POSIX clocks**:
  - **`CLOCK_REALTIME`**: **system-wide wall-clock time, settable**. Use it to **display the current time** (e.g., in Mountain Standard Time) and with **`clock_settime()`**.
  - **`CLOCK_MONOTONIC`**: **elapsed time since a starting point (e.g., boot), not settable**. **Use it to measure intervals or timestamp the time between two events.** Don't use REALTIME for intervals, because the system time may be set backwards or forwards.
- **Higher-resolution representations**:
  - **`timeval`** (**microseconds**): `gettimeofday`/`settimeofday`, `adjtime`.
  - **`timespec`** (**nanoseconds**): **`clock_gettime`, `clock_settime`, `nanosleep`, `clock_nanosleep`**.
  - **`struct tm`** (human-readable broken-down time): `asctime`, `mktime`, `gmtime`, `localtime`. **The `_r` versions are thread-safe**; consider **`strftime`** instead of `asctime`.
- **Setting time**: use **`clock_settime(CLOCK_REALTIME, …)`**, which **may require elevated permissions**.
  - **Troubleshooting when the time doesn't change**: make sure the **clock_id is `CLOCK_REALTIME`**, the program **runs with appropriate permissions**, and **check the return code**.
  - **`adjtime(delta)`** **slowly adjusts** the clock toward a more accurate time **without moving it backwards**. Useful for **NTP daemons**.

### 4.2 Sleeping and Timers
- **`sleep(seconds)`** **returns the number of seconds *not* slept**. It **exits early if a signal is received**.
- **`usleep(usec)`** sleeps **at least** `usec` microseconds (probably more).
- **`nanosleep(req, rem)`** gives nanosecond resolution. If interrupted, it **stores the remaining time in `*rem`**.
- **`clock_nanosleep(clock, flags, req, rem)`** supports **`TIMER_ABSTIME`**: sleep **until an absolute time**.
  - This makes **sleep sequences more precise**, because it **avoids the inaccuracy between `clock_gettime()` and the sleep call**. Any preemption or delay before the sleep is absorbed.
  - Example (A5 time quiz):
    - **`sleep1()`** uses a *relative* 1 s `clock_nanosleep`. **Preemption before the sleep call extends the total duration.**
    - **`sleep2()`** reads CLOCK_MONOTONIC, adds 1 s, and sleeps with TIMER_ABSTIME. **Preemption *after* `clock_gettime()` (e.g., at the `if (ret)` check) does NOT extend it.** Preemption *before* `clock_gettime()` (line 12) does, because the start time hasn't been sampled yet.
    - **Either function can take longer than 1 s** under preemptive scheduling (e.g., preemption after the wakeup).
- **Measure sleep durations with `CLOCK_MONOTONIC`.**
- **Sleep accuracy below ~1 ms isn't guaranteed** for any of the functions on a typical system. `clock_nanosleep` isn't inherently less accurate.
- **Alternatives to sleeping**:
  - Sleep is appropriate for **short (<1 s), infrequent events**.
  - **Code laced with sleeps to busy-wait is poor design.** **Blocking on a file descriptor** (letting the kernel sleep and wake you) is better.
  - **Don't use sleep as a band-aid for a poorly understood problem** ("I got an error, added a sleep, and it went away").
  - **Consider timers instead.**
- **Timers**:
  - **`alarm(seconds)`**: the simplest. It **delivers SIGALRM after the time expires**. **Concerns**: signal-handler reentrancy limits; **SIGALRM can also come from `sleep()`, `usleep()`, or `setitimer()`**, and **from outside the process**.
  - **Interval timers, `setitimer(ITIMER_REAL, …)`**: deliver SIGALRM after `it_value`, then **optionally re-arm every `it_interval`** (if `it_interval` is 0, it fires **once**). Same signal-handler issues.
    - The A5 time-quiz code sets **`it_value = 5 s` and `it_interval = 0`**, so `fire_rocket()` is called **once after 5 seconds, not continuously**.
    - The real concerns there: **`rocket_fire_handler()` may be triggered by another program** (SIGALRM can come from outside), and **`fire_rocket()` needs analysis for signal-handler safety**.
  - **POSIX timers (`timer_create`)** with **`SIGEV_THREAD`**: the expiry runs **a function in a thread instead of a signal handler**, and they **support `TIMER_ABSTIME`**. Because it's not a signal handler, **you can call non-signal-safe but thread-safe functions** such as `pthread_mutex_lock()`.

### 4.3 Sockets
- **Sockets are a form of IPC** that **can communicate across different systems over TCP/IP**. They're **more versatile than signals**, which need a pid on the same machine. Also called **BSD / Berkeley sockets**; **supported by all major operating systems** (including Windows).
- **Paths**: two hosts over Ethernet; **localhost loopback**; or **port forwarding into a VM/emulator**, e.g. QEMU's **`hostfwd=tcp::X-:Y`** (host port X → guest port Y).
- **TCP (Transmission Control Protocol)**: **connection-oriented**; the connection is kept for the whole exchange. It accepts packets, **manages flow control**, **retransmits dropped packets**, and **handles acknowledgments**.
- **IP**: **addresses packets and routes them** between sender and receiver.
  - **IPv4**: X.X.X.X, **32 bits (~4.3 billion addresses)**, now running out.
  - **IPv6**: **128 bits** (~340 undecillion addresses, ~4,000 per person).
  - A **port** provides local addressing for a specific application.
- **Socket types**:
  - **`SOCK_STREAM`**: **reliable, two-way, connected (TCP) streams**, **in order**, retried as necessary. Used in the course.
  - **`SOCK_DGRAM`**: **connectionless, UDP**. Used when **latency matters more than delivery** (e.g., video, where dropping a frame beats stuttering).
- **Everything is a file**: you interact through a **socket file descriptor**, obtained from **`socket()`**.
- **Server call sequence: `socket()` → `bind()` → `listen()` → `accept()` → `recv()`/`send()` → `close()`.** The client does `socket()` → `connect()`.
  - **`socket(domain, type, protocol)`**: domain **`PF_INET`/`PF_INET6`** (IPv4/IPv6), type **`SOCK_STREAM`/`SOCK_DGRAM`**, protocol **0** (picks the right one for the type).
  - **`bind(sockfd, addr, addrlen)`**: **assigns an address (IP and port) to the socket**; can optionally select a specific network adapter. In `sockaddr`, `sa_family` is AF_INET/AF_INET6 and `sa_data` holds the address and port; `addrlen = sizeof(struct sockaddr)`.
  - **Building a `sockaddr`**: either fill in a **`sockaddr_in` and cast it**, or use **`getaddrinfo()`** (newer, more flexible).
    - Set the hints to **`ai_flags = AI_PASSIVE`, `ai_family = AF_INET`, `ai_socktype = SOCK_STREAM`**, with **`node = NULL`**, and **`service = "9000"`** (the port as a string).
    - Pass **`&res`**, a pointer to a pointer: getaddrinfo **mallocs the addrinfo** and stores its address in your stack pointer.
    - Use **`res->ai_addr`** for bind, then **`freeaddrinfo(res)`**. **Forgetting to free it is a memory leak.**
  - **`listen(sockfd, backlog)`**: backlog = **the number of pending connections allowed before refusing**.
  - **`accept(sockfd, &addr, &addrlen)`**: blocks for a client and **returns a *new* fd for the accepted connection**. `addr` receives the client's address (for logging its IP); `addrlen` is in/out.
  - **`recv()`/`send()`**: like `read()`/`write()` (which also work on sockets), plus a **flags argument** (e.g., non-blocking). **Use the accepted fd**, not the listening fd.
- **A5 tips**:
  - **Compile with `-Wall -Werror`**: catches bugs such as calling an undeclared function with missing arguments, which would otherwise segfault.
  - **Close the fds from both `socket()` and `accept()`** to avoid leaks.
  - **Consider `shutdown()` in your signal-handling strategy.**
  - **Use `SO_REUSEADDR`** to avoid bind failing with **"Address already in use"** after a quick restart.
  - **Valgrind** ("val-grinned") checks for memory leaks: `valgrind --leak-check=full --show-leak-kinds=all --track-origins=yes --verbose --log-file=/tmp/valgrind-out.txt ./aesdsocket`. Look for **"All heap blocks were freed -- no leaks are possible"**.
- **A5 Part 1 (aesdsocket)**:
  - **Binds a stream socket to port 9000** (returns -1 on failure); listens and accepts; logs **"Accepted connection from <IP>"** to syslog.
  - **Appends received data to `/var/tmp/aesdsocketdata`**; a **newline marks the end of a packet**.
  - **Returns the file's full contents** after each complete packet, read line by line, since you can't assume it fits in RAM.
  - Logs **"Closed connection from <IP>"**, then loops to accept new clients.
  - On **SIGINT or SIGTERM**: logs **"Caught signal, exiting"**, finishes open connections, closes sockets, **deletes the data file**, and exits.
  - **`-d` runs it as a daemon. Fork only *after* successfully binding port 9000.**
  - Test with **netcat** (`nc`) / `sockettest.sh`.

### 4.4 Linux System Initialization (MELP Ch. 10)
- **Init options**:
  - **BusyBox init**: the **default for Buildroot**.
  - **System V (SysV) init**: the **default for Yocto**.
  - **systemd**: a more sophisticated startup manager (not covered).
- **System V init**:
  - A **1980s-era Unix system** with the concept of **runlevels**: **S = startup tasks, K = shutdown (kill) tasks**.
  - **Runlevels 1–5** are single- or multi-user modes, with or without graphical login.
- **In both SysV and BusyBox, initialization is driven by shell scripts.**
  - **BusyBox `/etc/init.d/rcS`** runs at **startup**: it runs every **`/etc/init.d/S??*`** script **in numeric order** with the **`start`** argument.
  - **`/etc/init.d/rcK`** runs at **shutdown**: it runs the same scripts **in reverse order** (`ls -r`) with **`stop`**.
- **An init script must be named `S<NN><name>`** (the number sets its order) and **must handle `start` and `stop`**, typically in a **`case` statement**.
- **Use `start-stop-daemon`**:
  - **`-S`** starts the daemon (if it isn't already running), e.g. `start-stop-daemon -S -n aesdsocket -a /usr/bin/aesdsocket -- -d`.
  - **`-K`** **sends SIGTERM** for a graceful exit, e.g. `start-stop-daemon -K -n aesdsocket`.
- **Test manually**: `/etc/init.d/S99aesdsocket start` and `/etc/init.d/S99aesdsocket stop`, then check with `ps`.

### 4.5 Embedded Debugging Strategies (MELP Ch. 14)
- Compile with **`-g`** for debug symbols, and **`-O0`** to disable optimizations for easier single-stepping.
- **gdb command line** (useful on remote systems without a GUI):

  | Command | What it does |
  |---|---|
  | `r <args>` | run |
  | `b` | set a breakpoint |
  | `p` | print a variable (can also call functions) |
  | `n` | step over the next statement |
  | `finish` | step out of the current function |
  | `info b` | list breakpoints |
  | `delete <x>` | delete a breakpoint |
  | `bt` | backtrace (shows where a segfault happened, even without debug info) |

- **Remote gdb** (for target-only problems, or when a production image can't hold full gdb):
  - Buildroot options: `BR2_PACKAGE_HOST_GDB`, `BR2_PACKAGE_GDB`, `BR2_PACKAGE_GDB_SERVER`.
  - **On the target**: `gdbserver :<port> ./app`.
  - **On the host**: run the **cross gdb** (in Buildroot, under **`output/host/bin/`**) with the cross-compiled binary, then **`target remote <target-ip>:<port>`**.

### 4.6 Assignment 5 Part 2 (Buildroot embedded socket server)
- Write an **`aesdsocket-start-stop`** script using start-stop-daemon (start with `-d`, stop with SIGTERM).
- Update the Buildroot **aesd-assignments** package to cross-compile `server/` and install **`aesdsocket` to `/usr/bin`** and the start script to **`/etc/init.d/S99aesdsocket`**.
- Update **`runqemu.sh`** to forward **host 9000 → guest 9000** and **host 10022 → guest 22**.
- **After a graceful `halt` and restart, `/var/tmp/aesdsocketdata` must not contain old data**: the stop script terminates the daemon, which deletes the file.

> **Module 4 Cheat-Sheet Candidates:** wall/process/monotonic; RTC + NTP keep wall time; relative vs absolute; epoch 1970-01-01; jiffies/HZ platform-dependent; Y2038 (32-bit time_t); REALTIME (settable, display) vs MONOTONIC (intervals, not settable); timeval µs vs timespec ns; `_r` thread-safe; clock_settime needs permissions; adjtime never goes backwards; sleep returns unslept seconds; TIMER_ABSTIME; sleep alternatives; alarm/setitimer/POSIX timer + SIGEV_THREAD; SOCK_STREAM TCP vs SOCK_DGRAM UDP; IPv4 32-bit vs IPv6 128-bit; socket→bind→listen→accept→recv/send; getaddrinfo AI_PASSIVE + freeaddrinfo; SO_REUSEADDR; valgrind; BusyBox rcS/rcK, S##name start/stop, start-stop-daemon -S/-K (SIGTERM); gdb commands; gdbserver + target remote.

---

<a id="quick-reference"></a>
## Quick-Reference Tables

### Process / program terms
| Term | Definition |
|---|---|
| Binary | Dormant program residing on storage, ready to execute |
| Process | OS abstraction: a binary in action; virtualized memory + ≥1 thread |
| Thread | Unit of execution with a virtualized processor, stack, and program state; shares process memory |
| Daemon | Background process with no controlling terminal; often ends in "d"; child of init |
| Zombie | Child that exited before the parent called wait() |
| pid 0 / pid 1 | idle process (one per CPU, power control) / init (starts services and login) |

### exec vs fork vs system
| Call | Effect | Returns? |
|---|---|---|
| `exec*()` | Replaces the current process image (same pid, priority, user/group) | Only on failure |
| `fork()` | Creates a child running the same image (copy-on-write); parent continues | 0 in child, child pid in parent |
| `system()` | fork + exec + wait via the shell (PATH, `$VAR` expansion risks) | Child's status |

### Signals
| Signal | Cause / meaning | Default action | Catchable? |
|---|---|---|---|
| SIGINT | Ctrl-C (foreground process group) | Terminate | Yes |
| SIGTERM | Graceful termination request (`kill`, start-stop-daemon -K) | Terminate | Yes |
| SIGKILL | Unconditional kill | Terminate | **No** |
| SIGSTOP | Unconditional pause | Stop | **No** |
| SIGSEGV | Invalid memory access / null pointer | Terminate + core | Yes |
| SIGABRT | `assert()` failure / `abort()` | Terminate + core | Yes |
| SIGHUP | Hang-up; often "reread config" | Terminate | Yes |
| SIGALRM | `alarm()` / `setitimer(ITIMER_REAL)` expiry | Terminate | Yes |

### Clocks and time types
| Item | Notes |
|---|---|
| CLOCK_REALTIME | Wall clock; settable; use for "current time" and clock_settime |
| CLOCK_MONOTONIC | Since boot; not settable; use for intervals and timestamps between events |
| time_t | Seconds since 1970-01-01 UTC; 32-bit rolls over Jan 2038 |
| timeval / timespec / tm | µs / ns / human-readable broken-down time |

### Socket server sequence
`socket(PF_INET, SOCK_STREAM, 0)` → `getaddrinfo(NULL, "9000", &hints{AI_PASSIVE}, &res)` → `bind(fd, res->ai_addr, res->ai_addrlen)` → `freeaddrinfo(res)` → `listen(fd, backlog)` → `newfd = accept(fd, &client, &len)` → `recv/send(newfd)` → `close(newfd)`; loop until SIGINT/SIGTERM → `close(fd)`.

### Daemon recipe
`fork()` → parent `exit()` → child `setsid()` → `chdir("/")` → close fds → redirect stdin/out/err to `/dev/null` → log via syslog.

### Filesystem Hierarchy Standard
`/bin` user programs (boot) · `/sbin` admin programs (boot) · `/dev` device nodes · `/etc` config · `/lib` shared libs · `/proc` `/sys` kernel pseudo-filesystems · `/tmp` temp (may clear on boot) · `/usr` extra programs/libs · `/var` runtime-modified files kept across boots (`/var/log`)

### Kconfig / Kbuild
`=y` / `*` built in (`obj-y`) · `=m` / `M` module (`obj-m`) · unset = off · `.config` holds the selections · defconfig = known-good starting config · `make menuconfig` · `make savedefconfig` (Buildroot: `save-config.sh`)
