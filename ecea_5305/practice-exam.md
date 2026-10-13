# ECEA 5305 — Practice Final Exam
### Linux System Programming and Introduction to Buildroot

**Format:** like the real final: True/False, multiple choice (one answer), and select-all-that-apply.
**Suggested time:** 2 hours. Close the study guide and use only your one-page notes sheet, as on the real exam.
**Answers:** in `practice-exam-answers.md`. Don't peek until you're done.

The questions are new ones written from the lecture material, not copies of the graded quizzes. They cover all four modules in roughly equal weight.

---

### Part A — True / False (1 point each)

1. The filename of a file is stored in its inode. **T / F**

2. A symbolic link can point to a file on a different filesystem, but a hard link cannot. **T / F**

3. The `fopen()` family of functions is documented in section 2 of the man pages because they are system calls. **T / F**

4. If a process opens a file with `open(path, O_WRONLY | O_CREAT, 0666)` and the process umask is `0022`, the resulting file permissions are `0644`. **T / F**

5. `read()` returning a value less than the requested count always indicates an error. **T / F**

6. `pread()` and `pwrite()` change the file position of the file descriptor, just as `lseek()` does. **T / F**

7. A successful call to `execl()` returns the exit status of the new program. **T / F**

8. After `fork()`, the parent and child initially share physical memory pages until one of them modifies a page. **T / F**

9. A zombie process is a child process that has terminated but whose parent has not yet obtained its status with `wait()`. **T / F**

10. The Linux kernel expects specific directories such as `/etc` and `/usr/bin` to exist in the root filesystem, or it will panic. **T / F**

11. errno is shared by all threads in a process, so one thread's error can overwrite another thread's errno value. **T / F**

12. A thread context switch is generally cheaper than a process context switch. **T / F**

13. Joinable threads that are never joined continue to consume system resources, causing a leak. **T / F**

14. SIGKILL can be caught by a signal handler so the process can clean up before exiting. **T / F**

15. `printf()` is safe to call from inside a signal handler. **T / F**

16. `CLOCK_MONOTONIC` can be set with `clock_settime()` to correct drift. **T / F**

17. `adjtime()` may move the system clock backwards in a single step. **T / F**

18. `SOCK_DGRAM` sockets guarantee in-order delivery with automatic retransmission. **T / F**

19. The file descriptor returned by `accept()` is the same one that was returned by `socket()`. **T / F**

20. Buildroot package definitions (Config.in and .mk files) contain the application's source code. **T / F**

---

### Part B — Multiple Choice (single answer, 1 point each)

21. Which statement best describes the ABI?
   - a) A source-level definition that lets code be recompiled on another platform
   - b) A definition of calling conventions, byte ordering, and register use that determines binary compatibility
   - c) The list of POSIX header files
   - d) The set of glibc wrapper functions

22. What is the result of `chmod 751 script.sh`?
   - a) rwxr-x--x
   - b) rwxrw---x
   - c) rwxr-xr-x
   - d) rw-r-x--x

23. Which component of the GCC toolchain contains the assembler and linker?
   - a) glibc
   - b) GCC
   - c) binutils
   - d) sysroot

24. In the toolchain prefix `aarch64-none-linux-gnu-`, what does `none` represent?
   - a) The CPU architecture
   - b) The vendor
   - c) The kernel
   - d) The operating system / C library

25. Why might you statically link an application on an embedded system?
   - a) To reduce total disk usage when many applications use glibc
   - b) Because it must run before the root filesystem (with shared libraries) is available
   - c) Because static linking is required for cross compilation
   - d) To enable `LD_LIBRARY_PATH`

26. Which syslog facility did Assignment 2's writer application use?
   - a) LOG_DAEMON
   - b) LOG_LOCAL0
   - c) LOG_USER
   - d) LOG_KERN

27. Which call synchronizes only a single file's data **and** metadata to disk?
   - a) `sync()`
   - b) `fsync(fd)`
   - c) `fdatasync(fd)`
   - d) `write(fd)` with `O_NONBLOCK`

28. Which is the *preferred* multiplexed I/O mechanism according to LSP?
   - a) `select()`
   - b) `pselect()`
   - c) `poll()`
   - d) `O_NONBLOCK` polling loop

29. What is the main risk of `system("cp $HOME/a /tmp")`?
   - a) It replaces the calling process
   - b) It uses PATH and shell expansion, creating injection risks
   - c) It never waits for the command to finish
   - d) It cannot be called from a multithreaded process

30. Why does a daemon call `setsid()`?
   - a) To change its working directory to `/`
   - b) To create a new session and process group with no controlling terminal
   - c) To lower its privileges to a non-root user
   - d) To redirect standard output to `/dev/null`

31. Which file produced by a kernel build is a compressed kernel image with a U-Boot header?
   - a) vmlinux
   - b) Image
   - c) zImage
   - d) uImage

32. What does `make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- mrproper` do?
   - a) Builds the device tree
   - b) Deep-cleans the kernel tree, including removing `.config`
   - c) Applies the default configuration for QEMU virt
   - d) Installs kernel modules into the rootfs

33. Which command creates the `/dev/console` device node for the minimal rootfs?
   - a) `mknod -m 666 dev/console c 1 3`
   - b) `mknod -m 600 dev/console c 5 1`
   - c) `mknod dev/console b 5 1`
   - d) `ln -s /dev/tty dev/console`

34. According to the FHS, which directory holds files modified at runtime that should be retained across boots?
   - a) /tmp
   - b) /var
   - c) /etc
   - d) /proc

35. Which threading model gives each request its own thread that performs blocking I/O?
   - a) Event-driven threading
   - b) Thread-per-connection
   - c) Scoped locking
   - d) State machine

36. Which is NOT a way a pthread terminates?
   - a) Returning from its start routine
   - b) Calling `pthread_exit()`
   - c) Another thread calling `pthread_cancel()`
   - d) Another thread calling `pthread_join()`

37. In Buildroot, what is the purpose of a br2-external tree?
   - a) To store the toolchain sysroot
   - b) To keep your own (often proprietary) packages outside the Buildroot source tree
   - c) To cache downloaded source tarballs
   - d) To hold the final root filesystem image

38. Which command rebuilds only your package using local source code, without pushing to git?
   - a) `make clean all`
   - b) `make AESD_ASSIGNMENTS_OVERRIDE_SRCDIR=/path aesd-assignments-rebuild`
   - c) `./clean.sh && ./build.sh`
   - d) `make savedefconfig`

39. Which signal is commonly used to tell a daemon to reread its configuration file?
   - a) SIGTERM
   - b) SIGHUP
   - c) SIGINT
   - d) SIGABRT

40. Which POSIX timer notification option runs the timer handler in a thread instead of a signal handler?
   - a) SIGEV_SIGNAL
   - b) SIGEV_NONE
   - c) SIGEV_THREAD
   - d) TIMER_ABSTIME

---

### Part C — Select All That Apply (2 points each; all correct choices required)

41. Which of the following are elements of an embedded Linux system as described in MELP Chapter 1? (select all)
   - a) Toolchain
   - b) Bootloader
   - c) Kernel
   - d) Root filesystem
   - e) Package manager (apt)

42. Which of the following are reasons to prefer a build system (Buildroot or Yocto) over a desktop distribution for an embedded product? (select all)
   - a) Distribution upgrades often assume a user is interacting with the system
   - b) Reducing the image size of a distribution can be difficult
   - c) Different upgrade paths can leave subtly different filesystems
   - d) Build systems can track the open-source licenses used
   - e) Distributions cannot run on ARM processors

43. According to the Buildroot manual (as covered in lecture), which of the following may require a **full rebuild**? (select all)
   - a) Changing the target architecture or toolchain configuration
   - b) Removing a package from the configuration
   - c) Changing a package's sub-options
   - d) Editing a source file in your own package
   - e) Running `make aesd-assignments-rebuild`

44. Which are best practices when writing a signal handler? (select all)
   - a) Save and restore errno
   - b) Call `malloc()` to allocate a buffer for logging
   - c) Minimize access to global data
   - d) Call only functions on the async-signal-safe list
   - e) Setting a single global flag that the main loop checks

45. A server using sockets must do which of the following, in order, before receiving client data? (select all that are required steps for a TCP server)
   - a) `socket()`
   - b) `bind()`
   - c) `listen()`
   - d) `accept()`
   - e) `connect()`

46. Which are valid reasons to use `CLOCK_MONOTONIC` rather than `CLOCK_REALTIME`? (select all)
   - a) Measuring how long an operation took
   - b) Computing a relative timeout
   - c) Displaying the current date to a user
   - d) Avoiding errors when NTP or the user changes the system time

47. Which of the following are true about BusyBox? (select all)
   - a) It is a single binary that implements many common Linux utilities
   - b) It uses symbolic links (e.g., `/bin/cat → busybox`) to determine which program was requested
   - c) Every option of every GNU utility is fully supported
   - d) `make CONFIG_PREFIX=<rootdir> install` installs it and creates the symlinks
   - e) It is configured using Kconfig/menuconfig

48. Which statements about a BusyBox init script placed at `/etc/init.d/S99aesdsocket` are correct? (select all)
   - a) It is run at startup by `/etc/init.d/rcS` with the argument `start`
   - b) It is run at shutdown by `/etc/init.d/rcK` with the argument `stop`
   - c) The `99` determines its order relative to other S scripts
   - d) `start-stop-daemon -K` in its stop case sends SIGKILL
   - e) It should handle `start` and `stop`, typically with a `case` statement

49. Which tools or commands help find the shared-library dependencies you must copy into a hand-built root filesystem? (select all)
   - a) `${CROSS_COMPILE}readelf -a busybox | grep "program interpreter"`
   - b) `${CROSS_COMPILE}readelf -a busybox | grep "Shared library"`
   - c) `${CROSS_COMPILE}gcc -print-sysroot` (to locate where those libraries live)
   - d) `strace busybox`
   - e) `chmod 755 busybox`

50. Your aesdsocket daemon fails with "bind: Address already in use" when restarted quickly. Which actions help? (select all)
   - a) Set the `SO_REUSEADDR` socket option before `bind()`
   - b) Ensure the previous instance closes its sockets on SIGTERM/SIGINT
   - c) Fork before calling bind() so the parent can retry
   - d) Use `SOCK_DGRAM` instead of `SOCK_STREAM`

---
*End of practice exam: 50 questions. Score: Part A (20) + Part B (20) + Part C (10 × 2 = 20) = 60 points.*
