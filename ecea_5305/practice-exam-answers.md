# ECEA 5305 — Practice Final Exam: Answer Key

Each answer gives the study-guide section to review if you missed it.

## Part A — True / False

| # | Answer | Explanation | Review |
|---|---|---|---|
| 1 | **F** | The inode holds the timestamps, owner, size, mode, and data location, but **not the filename**. Directories map names to inodes. | §1.6 |
| 2 | **T** | Hard links point directly to an inode, which belongs to one filesystem. Symlinks store a path, so they can point anywhere (and can break). | §1.6 |
| 3 | **F** | `fopen` is buffered stdio in **section 3** (library). The syscalls `open`/`read`/`write` are in section 2. | §2.1 |
| 4 | **T** | 0666 & ~0022 = 0644 (rw-r--r--). | §2.2 |
| 5 | **F** | A short read can mean EOF, or that fewer bytes are available now (e.g., interrupted by a signal). Only **-1** is an error. | §2.2 |
| 6 | **F** | pread/pwrite take an absolute offset and **don't change** the file position, which avoids races between threads. | §2.3 |
| 7 | **F** | exec **never returns on success**; it replaces the process image. | §2.4 |
| 8 | **T** | Copy-on-write: pages are shared until one side modifies them. | §2.4 |
| 9 | **T** | That's the definition of a zombie. | §2.4 |
| 10 | **F** | The kernel has **no expectations** about the rootfs layout except the init program named on the command line. The FHS is for user-space programs. | §2.6 |
| 11 | **F** | POSIX makes errno **thread-local**. | §1.9 |
| 12 | **T** | Threads share memory, so no address-space swap is needed. | §3.1 |
| 13 | **T** | Join them or detach them; otherwise it's a memory leak. | §3.2 |
| 14 | **F** | SIGKILL (and SIGSTOP) **cannot be caught or ignored**. | §3.4 |
| 15 | **F** | stdio functions such as printf aren't async-signal-safe. | §3.4 |
| 16 | **F** | MONOTONIC **isn't settable**; only CLOCK_REALTIME is. | §4.1 |
| 17 | **F** | adjtime adjusts slowly and **never moves the clock backwards** (it's meant for NTP daemons). | §4.1 |
| 18 | **F** | DGRAM is connectionless UDP with no ordering or retransmission guarantee. That describes SOCK_STREAM (TCP). | §4.3 |
| 19 | **F** | accept() returns a **new** fd for the connection; the listening fd stays open for more clients. | §4.3 |
| 20 | **F** | Packages contain **instructions** to get the code, compile it, and install it, never the code itself. | §3.3 |

## Part B — Multiple Choice

| # | Answer | Explanation | Review |
|---|---|---|---|
| 21 | **b** | ABI = binary compatibility: calling conventions, byte order, registers. (a) describes an API. | §1.5 |
| 22 | **a** | 7 = rwx, 5 = r-x, 1 = --x. | §1.2 |
| 23 | **c** | binutils = the binary utilities, including the assembler and linker. | §1.10 |
| 24 | **b** | Format is CPU-vendor-kernel-OS; `none` is the vendor. | §1.10 |
| 25 | **b** | Static linking suits code that runs before the rootfs (and its shared libraries) is available, or a single-binary system like BusyBox. (a) is a reason for *dynamic* linking. | §1.10 |
| 26 | **c** | A2 used LOG_USER, with LOG_DEBUG for success and LOG_ERR for errors. | §1.11 |
| 27 | **b** | fsync = data + metadata for one fd. fdatasync = data only. sync = system-wide. | §2.2 |
| 28 | **c** | poll() is preferred for its flexibility. | §2.3 |
| 29 | **b** | system() uses PATH (injection risk) and runs through the shell, which expands `$HOME`. | §2.4 |
| 30 | **b** | setsid() creates a new session and process group with no controlling tty. | §2.5 |
| 31 | **d** | uImage = compressed + U-Boot header. zImage = compressed. Image = raw binary. vmlinux = ELF. | §2.6 |
| 32 | **b** | mrproper deep-cleans the tree and removes `.config`. | §2.6 |
| 33 | **b** | The console device is char major 5, minor 1. (Major 1, minor 3 is /dev/null.) | §2.6 |
| 34 | **b** | /var holds runtime-modified data that's retained (e.g., /var/log). /tmp may be cleared on boot. | §2.6 |
| 35 | **b** | Thread-per-connection (e.g., Apache). | §3.1 |
| 36 | **d** | pthread_join waits for a thread to end; it doesn't terminate it. | §3.2 |
| 37 | **b** | br2-external keeps custom or proprietary packages outside Buildroot. | §3.3 |
| 38 | **b** | `OVERRIDE_SRCDIR` + `-rebuild` builds from a local directory and avoids a clean build. | §3.3 |
| 39 | **b** | SIGHUP ("hang up") is conventionally used to make a process reread its config. | §3.4 |
| 40 | **c** | SIGEV_THREAD runs the handler in a thread, so you can use thread-safe (not just signal-safe) calls. | §4.2 |

## Part C — Select All That Apply

| # | Answers | Explanation | Review |
|---|---|---|---|
| 41 | **a, b, c, d** | Toolchain, bootloader, kernel, root filesystem. A package manager isn't one of the four. | §2.6 |
| 42 | **a, b, c, d** | All are lecture points. (e) is false: distributions like Debian run on ARM. | §3.3 |
| 43 | **a, b, c** | Toolchain/arch change, package removal, and sub-option changes may need a full rebuild. Editing your own source only needs a single-package rebuild. | §3.3 |
| 44 | **a, c, d, e** | Setting a global flag is the accepted pattern. malloc isn't async-signal-safe. | §3.4 |
| 45 | **a, b, c, d** | socket → bind → listen → accept. connect() is the *client* call. | §4.3 |
| 46 | **a, b, d** | Displaying the current date needs CLOCK_REALTIME. | §4.1 |
| 47 | **a, b, d, e** | Not every option is supported ("often not all arguments are supported"). | §2.6 |
| 48 | **a, b, c, e** | `start-stop-daemon -K` sends **SIGTERM** (graceful), not SIGKILL. | §4.4 |
| 49 | **a, b, c** | readelf shows the interpreter and needed libraries, and the sysroot is where you copy them from. strace shows runtime syscalls (it doesn't help you build the rootfs), and chmod is unrelated. | §1.10, §2.6 |
| 50 | **a, b** | SO_REUSEADDR plus graceful cleanup. The spec says to fork **after** a successful bind, and DGRAM doesn't meet the requirements. | §4.3 |

---
**Scoring guide:** 54–60 = exam-ready · 45–53 = review the sections you missed · below 45 = re-read the study guide modules flagged above.
