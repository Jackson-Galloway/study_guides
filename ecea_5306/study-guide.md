# ECEA 5306 — Linux Kernel Programming and Introduction to Yocto Project
## Final Exam Study Guide

*Built from the course lecture transcripts, slides, readings, and quizzes (instructor: Dan Walkes, CU Boulder MS-ECE). Main reference: **Linux Device Drivers, 3rd Edition (LDD3)**, Chapters 1–6, plus Mastering Embedded Linux Programming for Yocto.*

**Final exam format** (from "About the Final Exam"): proctored via ProctorU, **2 hours**, **1 attempt**, "similar in format to the homework quiz assignments" (True/False, multiple choice, select-all). Worth **15%** (quizzes 20%, peer-reviewed assignments 65%).
**Allowed:** one notes sheet (front and back, ≥11pt if typed, *your own*), or an electronic copy of it, or one URL to your notes; pens/paper. **Not allowed:** textbooks, lecture slides/content, AI tools.

> The **Cheat-Sheet Candidates** box at the end of each module lists the facts most worth putting on your own notes sheet.

---

## Table of Contents
1. [Module 1 — Device Drivers, Kernel Modules, Yocto](#module-1)
2. [Module 2 — Character Drivers, Kernel Debugging](#module-2)
3. [Module 3 — Kernel Concurrency & Locking, Assignment 8](#module-3)
4. [Module 4 — ioctl, Sleeping, poll/select, Assignment 9](#module-4)
5. [Quick-Reference Tables](#quick-reference)

---

<a id="module-1"></a>
## Module 1 — Device Drivers, Kernel Modules, Yocto

### 1.1 Introduction to Device Drivers (LDD3 Ch. 1)
- **OS structures**:
  - **Monolithic**: the whole kernel runs in privileged (kernel) mode. **Linux is "monolithic modular".**
  - **Microkernel**: minimal kernel; many OS services run in user mode. Example: **FreeRTOS** (per the course).
  - **Hybrid**: a mix, e.g. **Windows, macOS**.
- **Role of a device driver**: **make hardware respond to a well-defined abstraction interface**, i.e. map system calls like `read()`, `write()`, `ioctl()` onto hardware operations. Drivers can be **built separately from the kernel and plugged in when needed**.
- **Mechanism vs policy: a driver should provide *mechanism*, not *policy*.**
  - **Mechanism** = *what capabilities are provided*. **Policy** = *how those capabilities can be used*.
  - Examples: TCP/IP sockets are the mechanism and servers/services are the policy; an HDD driver provides blocks (mechanism) and a filesystem provides the policy.
  - So **a driver should NOT enforce a specific way applications must use the hardware.**
- **Policy-free driver characteristics**:
  - Supports **both synchronous and asynchronous** operation.
  - **Can be opened multiple times at once.**
  - **No extra software layers to "simplify things"** (those tend to impose policy).
  - Often **released together with user-space programs** that provide policy, configuration, and access.
- **Kernel roles** (what the *kernel* does):
  - **Process management**: create and destroy processes, handle process I/O, schedule.
  - **Memory management**: build each process's virtual address space; back `malloc()`/`free()`.
  - **Filesystems**: "everything is a file"; many types (ext3/ext4, FAT, NTFS).
  - **Device control**: endpoint of system operations; **a driver for every device** (e.g., controlling an attached keyboard).
  - **Networking**: collect, identify, and **dispatch network packets**; routing and address resolution.
- **User-space roles** (NOT the kernel): **applications** such as an **FTP server (ftpd)**, a **graphics/display session manager**, and **utilities that interact with drivers** (e.g., the `tunelp` printer control utility).
- **Modules**:
  - A **driver** is kernel code that controls hardware; it may or may not be added at runtime.
  - **A module is kernel code added at runtime.** Modules are how **features get added to the running kernel**.
  - **`insmod`** links a module into the running kernel. **`rmmod`** removes it. **`modprobe`** links it **and loads its dependencies** too.
  - Typically implemented as modules: char, block, and network interface drivers (hardware-specific) and filesystem types.

### 1.2 Device Driver Overview: Classes, Security, Versioning, Licensing
- **Device classes** share common access code: **character, block, network**.
  - **Character devices**: accessed as a **stream of bytes**; look like files (open/close/read/write). Examples: `/dev/console`, `/dev/ttyS0`. **May or may not support seek or mmap.** The course focuses on char drivers.
  - **Block devices**: a disk that **can host a filesystem**. Device-level transfers are **always on block boundaries (≥512 bytes)**, but the kernel lets user space access smaller amounts and handles the splitting. Example: `/dev/sda1`.
  - **Network interfaces**: hardware (or software such as **loopback**). They **handle packets** and use **names like `eth0` with no `/dev` entry**. Why? **Packet I/O doesn't map well to read()/write()**, which breaks "everything is a file".
- **Kernel filesystems** map low-level disk blocks to files and directories, **independent of the transfer mechanism** (SATA, USB, SD); device drivers do the actual transfers.
- **Security**:
  - Security checks are ultimately enforced by kernel code, so **the kernel is a critical point of exploit**.
  - **Only authorized users can load modules.**
  - **Drivers typically do NOT encode security policy.** The **system admin controls access through device file permissions**. Exceptions: global resource changes (**interrupt lines, firmware updates**).
- **Security best practices**:
  - **Don't trust user data without verifying it**; don't let user input write memory it shouldn't.
  - **Zero memory obtained in the kernel before passing it to user space**, to avoid leaking old data such as keys.
  - **Length-check user data before copying**, to avoid buffer overruns.
- **Buffer overrun**: overflowing a stack buffer (e.g., `strcpy` into `char too_small[100]`) can **overwrite the return address**. An attacker supplies a **NOP sled + malicious code** and points the return address back into the payload.
- **Kernel versioning**: the even/odd scheme in the book is obsolete. The kernel now uses **time-based major.minor releases**, not feature-based (Linus was "running out of fingers and toes").
- **Licensing: GPLv2.**
  - Anyone can redistribute and sell, **as long as recipients get the source and the same rights**. **Derived works that are redistributed must also be GPL.** The goal is growth of knowledge while still allowing commercial use.
  - **Can a module be shared in binary-only form?** It's **"deliberately ambiguous"**. Kernel developers have no qualms about breaking binary modules between releases. **Seek legal advice.**
  - So if your boss wants a closed-source driver, **recommend discussing it with legal counsel**.

### 1.3 Introduction to Kernel Modules (LDD3 Ch. 2)
- Use the course's updated **`ldd3`** sample repo (not the book's FTP site). Run `make` in `misc-modules`, then `sudo insmod hello.ko`. The output appears in **`/var/log/syslog`** or `dmesg`: "Hello, world" on load, "Goodbye, cruel world" on `rmmod`.
- **"Loading out-of-tree module taints kernel"**: a **tainted** kernel tells kernel developers they lack the information to debug a bug report. Causes include non-GPL modules and out-of-tree modules.
- **hello.c anatomy**: `MODULE_LICENSE("Dual BSD/GPL")`, an init function registered with **`module_init()`**, an exit function registered with **`module_exit()`**, and **`printk(KERN_ALERT "Hello, world\n")`**.
- **Module vs application**:
  - **`module_init(fn)`** names the function run at load (insmod). **Unlike `main()`, it sets things up, *registers the module's functions with the kernel*, and returns.** After that the module is **event-driven**: the kernel calls into it.
  - **`module_exit(fn)`** names the function run at unload (rmmod). It's similar to a SIGTERM handler, but **there's no automatic cleanup**: **memory you forget to free stays allocated until reboot**.
  - **Why `printk` and not `printf`?** **There's no libc in the kernel**, so no `<stdio.h>`. Use kernel-provided functions.
  - **`MODULE_LICENSE`** declares a free license. **Some kernel functions are only available to GPL-licensed modules.**
  - **A segfault in a module can bring down the whole system**; there's no core dump or cleanup as in user space.
  - **The kernel stack is tiny**: ~2 MB for an application vs **possibly 4 KB** in the kernel, shared by all active functions. **Don't declare large automatic (stack) variables; allocate dynamically** (e.g., a buffer of several thousand bytes → `kmalloc`).
  - **Floating point is generally not supported** in the kernel.
- **How a module links in**: init registers data and callbacks with kernel structures (e.g., `blk_init_queue`, `add_disk`). The kernel later invokes the module through function pointers (e.g., `block_device_operations`). The exit function **undoes the registrations in reverse order**.
- **Entering kernel space**: a **system call** from an application (software trap) or a **hardware interrupt**. *(Running as root does NOT put code in kernel space.)*
- **User vs kernel space**: CPUs enforce **privilege levels (modalities)** with separate address spaces.
  - **Applications** run in **user space at the lowest privilege**, with regulated hardware and memory access.
  - **Drivers** run in **kernel space at the highest privilege (supervisor mode)**, entered via a syscall or interrupt.
- **Concurrency in the kernel**:
  - Many processes may use your driver at once; **interrupts and kernel timers run asynchronously**; **SMP** (symmetric multiprocessing: 2+ CPUs sharing memory and one OS instance) is supported.
  - So **all kernel code, including your driver, must be reentrant**: **always**, not only when sleeping or on SMP. Data structures must keep threads of execution separate, and shared data must be properly protected.

### 1.4 Building and Loading Kernel Modules
- **The module build differs from user-space builds** (see `Documentation/kbuild`). The module's Makefile just says **`obj-m := hello.o`**. Invoke it with **`make -C /path/to/kernel/source M=$(pwd)`**.
- **Finding the kernel source**: `/lib/modules/$(uname -r)/build` → a symlink to `/usr/src/linux-headers-…`, which contains the kernel's Makefile. (**`uname -r`** prints the running kernel version.) Install the `linux-headers` package to get it.
- **Recursive make**: your Makefile is **read twice**.
  1. **First pass: `KERNELRELEASE` isn't set**, so the Makefile calls `make -C $(KERNELDIR) M=$(PWD) modules`.
  2. **Second pass**: the kernel Makefile's `modules` target uses **`M`** to find and re-read your Makefile. Now KERNELRELEASE *is* set, and it **understands `obj-m` and builds the module**.
  - **Why it's done this way**: it **reuses the ubiquitous make tool**, and it **bootstraps** the build rules from the kernel source you're building against. If the build process changes between kernel versions, individual module Makefiles don't need updating.
- The output is a **`.ko`** (kernel object).
- **`insmod`** loads the `.ko` **after linking it to the kernel symbol table**. It's like the **`ld` linker** (it resolves symbols such as `printk`), except it **modifies an in-memory copy, not the file on disk**. You can **pass parameters** at load time.
- **`modprobe`** also loads modules and **handles dependencies automatically**, avoiding **"unresolved symbols"** failures when the needed modules exist but aren't loaded yet.
- **`rmmod`** removes a module, which **must not be in use**.
- **`lsmod`** lists loaded modules, their size, and what uses them (from **`/proc/modules`** / **`/sys/module`**).
- **`/proc` and `/sys`** are **virtual filesystems**; the kernel **populates their contents on demand**. Do they break the "kernel makes no rootfs assumptions" rule? No: **they're mounted at runtime by user space** (conventionally at `/proc` and `/sys`).

### 1.5 Kernel Module Design Basics
- **Version dependency**: a module **only loads against the kernel it was compiled for**.
  - The **vermagic** string (target kernel, **compiler version, processor, key config options**) must match.
  - A mismatch gives **`insmod: Error inserting 'x.ko': -1 Invalid module format`**. **Fix: rebuild against the correct kernel source/version.**
- **APIs break between kernel versions.** The **`LINUX_VERSION_CODE` / `KERNEL_VERSION(a,b,c)` macros** let one source file handle several kernel versions (e.g., provide your own replacement for a removed function).
- **Keeping up with kernel variations**:
  1. **Release under a GPL-compatible license and ideally get it into the mainline kernel**, so whoever changes an API must update your driver.
  2. **Distribute source with build scripts**, e.g. **DKMS (Dynamic Kernel Module Support)**.
  3. **(Embedded) build for a single target kernel configuration.**
- **Kernel symbol table**:
  - Modules can **export symbols** for other modules to use (**module stacking**); modprobe loads such dependencies.
  - **`EXPORT_SYMBOL(sym)`** makes a symbol available to any module. **`EXPORT_SYMBOL_GPL(sym)`** makes it available **only to GPL-licensed modules**.
- **Headers**: **`<linux/module.h>`** (definitions and symbols for loadable modules) and **`<linux/init.h>`** (init and cleanup macros). Plus `MODULE_LICENSE`, `MODULE_AUTHOR`, `MODULE_DESCRIPTION`, `MODULE_VERSION`.
- **The init function**:
  - Should be **`static`** (not used outside the file) and marked **`__init`**, so **the loader can drop its memory after loading**.
  - It **registers with kernel facilities** (devices, filesystems, crypto, sysfs, proc…) and **initializes data structures *before* registering**, since the kernel may call you as soon as you register.
  - **Data structures belong in allocated memory, not on the stack** (the init stack frame disappears).
- **The exit function** **unregisters everything from init, typically in reverse order**, and **frees memory allocated in init**. Mark it **`__exit`**: if the code is built into the kernel or can't be unloaded, it can be discarded.
- **Error handling in init**: **most registrations can fail**, so check every one. **On failure, undo the registrations already made, in reverse order.** **`goto` is widely used and accepted** in the kernel for this unwinding (`goto fail_x;` labels). *Don't avoid goto, and never force a reboot.*
- **Module parameters**: **`module_param(var, type, perm)`**, e.g. `S_IRUGO` (readable, not changeable) or `S_IRUGO|S_IWUSR` (root can write). Example: `hellop.c` with `howmany` and `whom`; use `insmod hellop.ko howmany=10 whom="Mom"`.

### 1.6 Introduction to Yocto
- **Yocto is a way to roll your own Linux *distribution* for embedded devices**: **not just a root filesystem**, it builds **redistributable packages for every piece of software** (like Debian does).
  - **Launched in 2011** (Buildroot dates from 2001).
  - Includes embedded build tools and the reference embedded distribution **Poky** (pronounced like "hockey").
  - The **build system is licensed MIT**, which is permissive: there's no expectation that you share your Yocto customizations. (Buildroot is GPLv2.)
  - It uses **text-file configuration and the `bitbake` tool (Python + Bash)**.
  - It **builds almost everything from source, including the build tools**, to **support binary reproducibility** (anyone checking out the same commit gets identical output).
- **Setup**:
  1. Add **poky as a git submodule** at the branch given in the assignment, then `git add` and `git commit` it **before running build.sh** (otherwise build.sh resets it to the last committed hash).
  2. **`source poky/oe-init-build-env`**.
  3. **`bitbake core-image-minimal`** (or core-image-aesd), then "work on something else for a few hours".
- **`source file`** (synonym: `. file`) runs the script **in the current shell**, so **variables it sets remain available afterwards** (`./file` would run in a new shell). `oe-init-build-env` sets environment variables for your session **and creates and `cd`s into a `build/` directory**, where all build output and logs go (tens to ~100 GB).
- **Terminology**:
  - **bitbake** performs builds. "How do you bake bits? With **recipes**."
  - A **recipe** is a set of instructions processed by the build engine, in **`.bb`** (typically **source and version information**) and **`.inc`** (common **build and deploy instructions**, with Python or Bash fragments) text files.
  - **Packages** hold the build's binary artifacts.
  - **Images** are the build outputs: the **root filesystem image, kernel image, and bootloader image (u-boot or grub)**.
- **The layer model**:
  - **Designed for both collaboration and customization.** Layers are directories **virtually "layered" into one equivalent build hierarchy**, with priority deciding precedence. Example: `meta-aesd/recipes-core/busybox/busybox.bbappend` modifies `meta/recipes-core/busybox`.
  - Layers can **override or add to recipes**: apply a hardware-specific patch, configure GUI vs terminal-only, **add new recipes** (`aesd-assignments`), or **add new images** (`core-image-aesd`).
  - This is more flexible than Buildroot's br2-external.
  - **Adding a layer**: `bitbake-layers create-layer ../../meta-aesd`, then `bitbake-layers add-layer ../../meta-aesd` (done for you by build.sh, which checks `bitbake-layers show-layers`). Config lives in `meta-aesd/conf/layer.conf`.
- **Custom image**: `core-image-aesd.bb` with **`inherit core-image`** (pulls in a core set of packages such as busybox), plus **`IMAGE_INSTALL:append` / `CORE_IMAGE_EXTRA_INSTALL += "aesd-assignments"`** to add packages (space-separated).
- **Target selection**: Buildroot picks the target with a **defconfig**. **Yocto uses the `MACHINE` variable in `build/conf/local.conf`** (e.g., `MACHINE = "qemuarm64"`). local.conf usually **isn't checked in**, so build.sh manages it.

### 1.7 Building a Yocto Image (Assignment 6)
- **What `build.sh` does**:
  1. Inits and updates submodules.
  2. **`source poky/oe-init-build-env`** (which changes into `build/`; every later command runs there).
  3. Makes sure **`MACHINE`** is in `local.conf`.
  4. Adds the **meta-aesd** layer if missing.
  5. Runs **`bitbake core-image-aesd`**.
- **Running the image**: `source poky/oe-init-build-env` then **`runqemu nographic`**, which uses the environment to find MACHINE and the image (`runqemu --help` for options).
  - The default login is **root with no password**; the assignment changes it to "root" to match Buildroot.
  - **`runqemu.sh`** uses **slirp (user-mode) networking**, which **doesn't require root** (unlike the default tap interface). Target ports are reachable via localhost (guest 10.0.2.15).
- **Adding a recipe with `devtool`** (the easy way; already done in the starter repo):
  - **`devtool add aesd-assignments <repo>`** generates a recipe in `build/workspace`.
  - **`devtool build aesd-assignments`** verifies it builds.
  - **`devtool finish aesd-assignments ../../meta-aesd`** moves the `.bb` into your layer.
- **Makefile changes for Yocto**: **Yocto expects to override `CC`, `CFLAGS`, `LDFLAGS`**, so **set them with `?=`** in your Makefile (overridable, with native defaults) and use `$(CC) $(CFLAGS) $(INCLUDES) … $(LDFLAGS)` in the compile step.
- **Recipe TODOs** (`aesd-assignments_git.bb`):
  - `SRC_URI` with the **SSH** URL. Use **ssh-agent**: the build fails if it needs an interactive passphrase.
  - **`SRCREV`** = the commit hash.
  - **`FILES:${PN}`** lists every file installed to the rootfs.
  - **`TARGET_LDFLAGS`** for linker flags.
  - A **`do_install()`** using `${D}`, `${bindir}`, `${sysconfdir}`.
- **Startup scripts in Yocto**: Yocto uses **System V init** (Buildroot uses BusyBox init). Use the **`update-rc.d` class**:
  - **`inherit update-rc.d`**
  - **`INITSCRIPT_PACKAGES = "${PN}"`** flags the package as using init scripts.
  - **`INITSCRIPT_NAME:${PN} = "aesdsocket-start-stop"`**
  - In `do_install`: `install -d ${D}${sysconfdir}/init.d` and `install -m 0755 ${S}/<script> ${D}${sysconfdir}/init.d`.
  - Newer Yocto releases use **`:`** override syntax; older documentation uses `_`.
- **Iterating on source: `devtool modify aesd-assignments`** (Yocto's equivalent of Buildroot's `OVERRIDE_SRCDIR`).
  - It prints a workspace path. Edit there, and the next bitbake/build.sh uses that source.
  - Commit and push from the workspace, update the hash in the `.bb`, then **`devtool reset aesd-assignments`** to go back to the recipe's settings.
- A working reference is the **yocto-hello-world** example branch.

### 1.8 Yocto vs Buildroot
- **Why not just cross-compile and copy binaries into a rootfs?**
  - **Hardware changes are easier to manage** with a build framework (swap a config or MACHINE).
  - **Emulation is easier** to use.
  - **Hacks become unmanageable long term.**
  - **Technical debt is a recurring cost**: you pay it on every build and release.
- **"Which is better?" is a question of religion**: there's no right answer; it's preference and bias.
- **Similarities**: both have talented, responsive developers and active mailing lists; both are well thought out and maintained; both support common platforms (x86, ARM); both have extensive documentation.
- **Differences**:

  | Criteria | Buildroot | Yocto |
  |---|---|---|
  | Simplicity / initial learning curve | **Simplest** | More complex |
  | Developer workflow support | Good | **Best** (devtool) |
  | Clean build time | **Shorter** | Longer |
  | Incremental build support | Moderate | **Very good** (fewer clean-build caveats) |
  | Platform/package support | Good | **Best** |

- **When to consider Buildroot**: a project already using it; a team familiar with it; a team with Makefile/kernel skills and little Python/Yocto experience; a team not excited to learn Yocto; a mature, well-supported platform; few and simple requirements; a well-defined roadmap.
- **When to consider Yocto**: complex requirements or an undefined roadmap; **newer hardware**, especially **if the BSP is only available for Yocto** or the manufacturer hasn't committed to Buildroot; a team excited to learn Yocto.

### 1.9 Assignment 6 Part 1: Threaded Socket Server, Linked Lists
- **Requirements**:
  - **Spawn a new thread per accepted connection.**
  - **Serialize writes to `/var/tmp/aesdsocketdata` with a mutex**, so data from simultaneous connections never interleaves (packet order between connections may vary).
  - A thread **exits when the client closes the connection or on a send/recv error**.
  - On SIGINT/SIGTERM, **request that every thread exit and join them all** before exiting.
  - **Use `pthread_join()`, not detached threads.**
  - **Append `timestamp:<RFC 2822 strftime time>\n` every 10 seconds** (wall-clock time), locked so it's atomic with respect to socket data.
  - Must be **valgrind-clean** (one "possibly lost" from `timer_create`'s helper thread is OK).
- **Linked lists**: each value is paired with a pointer to the next (and possibly previous) value; **storage needn't be sequential**.
  - **Benefits over arrays**: no need to know the size in advance; limited only by memory; add or remove nodes without moving memory.
  - **Drawbacks**: extra memory for the pointers; **indexing is O(n)**.
  - **Writing your own**: be careful with **removing the head** and **removing while iterating**. Using an existing implementation is better tested, saves time, and gives you API practice.
- **`queue.h` SLIST (singly linked list)**:
  - `SLIST_HEAD` declares the head; `SLIST_INIT` initializes it; `SLIST_INSERT_HEAD` inserts at the head; `SLIST_REMOVE` removes.
  - `SLIST_FOREACH` iterates. **`SLIST_FOREACH_SAFE`** iterates **while allowing removal** (it needs the FreeBSD version of queue.h).
  - Each element embeds an `SLIST_ENTRY(type) entries` field.
- **Suggested flow**:
  - **Main thread**: `accept()` → create the thread → add it to the list → for each thread in the list whose **complete flag** is set, **`pthread_join()`** it and free its node.
  - **Worker thread**: recv/send → set its complete flag → exit.
- **Pitfalls**:
  - **Deallocate memory in one place only** (the main thread).
  - **The list isn't thread-safe**, so access it only from the main thread.
  - **Don't forget `pthread_join`** (valgrind will report it).
  - **Detached threads are only safe with some other way to know a thread has terminated**; they cause race conditions on shutdown.
  - **valgrind hangs**: tight polling loops starve the threads under valgrind (`--fair-sched=yes` hides the problem). **Block on `accept()`** instead.

> **Module 1 Cheat-Sheet Candidates:** mechanism vs policy; kernel roles vs user-space roles; char/block/net classes (net has no /dev node); drivers don't enforce security policy; zero memory and length-check user data; GPLv2 + binary modules "deliberately ambiguous" → legal counsel; printk not printf (no libc); module_init registers and returns; module_exit has no automatic cleanup; 4 KB kernel stack → kmalloc; no floating point; reentrant always; entry via syscall or interrupt; obj-m + `make -C … M=`; recursive make reads the Makefile twice (KERNELRELEASE); insmod vs modprobe (dependencies) vs rmmod vs lsmod; vermagic → "Invalid module format"; EXPORT_SYMBOL_GPL; `__init`/`__exit`; goto unwinding in reverse; module_param; Yocto = 2011, MIT, Poky, bitbake, recipes (.bb/.inc), layers, MACHINE in local.conf, devtool add/modify/finish/reset, update-rc.d, `?=` for CC; Buildroot vs Yocto table.

---

<a id="module-2"></a>
## Module 2 — Character Drivers, Kernel Debugging

### 2.1 Connecting to User Space: Device Numbers (LDD3 Ch. 3)
- The **scull** driver ("Simple Character Utility for Loading Localities") **acts on memory as if it were a device**.
  - Benefits: **portable across architectures, not hardware dependent**. Drawback: **not a "real" driver** (no hardware interrupts).
  - Use the course's updated **ldd3** repository.
- **scull variants**:
  - **scull0–3**: four devices with a **global** memory area (data shared by all file descriptors) that is **persistent** (data survives close and reopen, until the module is unloaded).
  - **scullpipe0–3**: **FIFO pipes**, demonstrating **blocking and non-blocking** read/write.
  - **scullsingle**: only one process at a time. **scullpriv**: private per virtual console. **sculluid / scullwuid**: openable by only one user at a time.
- **How do `/dev` nodes map to drivers? Device numbers + `mknod`** (e.g., A3's `mknod -m 666 dev/null c 1 3`).
  - `ls -l /dev` shows **two numbers: major, minor**.
  - **Major = type: identifies the *driver*.** **Minor = identifies the *specific device*** that driver controls.
  - **Example**: `crw-rw-rw- 1 root root 1, 3 … null` means **major 1, minor 3**. `/dev/zero` is also major 1 (minor 5), so it uses the same driver as null.
  - **Example**: `tty1` is **4, 1** (major 4, minor 1). `ttys0` (4, 64) and `ttyS1` (4, 65) share major 4, so **they most likely use the same driver**.
  - A minor number is an *instance identifier*, **not a device count**.
- **Allocating a major number**:
  - Common devices have **dedicated (static) majors** listed in `Documentation/admin-guide/devices.txt`. For example, SCSI disks (`sd`) are major 8. **If your device is a common type, use its assigned number.**
  - **Everyone else allocates dynamically at runtime.** *(You don't request a new static number from kernel.org.)*
- **Register and unregister**:
  - `register_chrdev(major, name, &fops)`: with **major = 0 the kernel allocates one dynamically and returns it** (simple example: `sleepy.c`).
  - **You are responsible for unregistering**: call **`unregister_chrdev`** in exit. Balance every register with an unregister.
  - scull uses **`alloc_chrdev_region`** / `register_chrdev_region` for a range of devices, with `goto` cleanup if a later step (e.g., `kmalloc`) fails.
- **`dev_t`** holds both numbers (just a typedef for a **32-bit integer**: 12 bits major, 20 bits minor). **`MKDEV(major, minor)`** builds one; **`MAJOR(dev)`** and **`MINOR(dev)`** extract them.
- **`/proc/devices`** lists the currently allocated majors and driver names. Dynamically registered drivers appear there with the name passed to register_chrdev.
- **Dynamic allocation means you can't create device nodes in advance.**
  - Load the module, then **parse `/proc/devices` to find its major**, then `mknod`. See the `module_load` and `scull_load` scripts.
  - At init time, also **set the owner, group, and permissions** on the node.
  - **awk** (named after its developers, not an acronym) is ideal for space-separated fields: `major=$(awk "\$2==\"$module\" {print \$1}" /proc/devices)`, then **`mknod /dev/$module c $major 0`**.
  - Example output: "Allocated major 241", `/dev/sleepy` = 241, 0.

### 2.2 Device Driver File Operations
- **`struct file_operations` ("fops")**:
  - **Connects driver operations to device numbers** (open, read, write, llseek, unlocked_ioctl, poll, release…). **It maps driver functions to system calls.**
  - It's **object-oriented C**: the `struct file` is the object and the functions are its methods.
  - It's registered with register_chrdev / cdev.
  - It uses **C tagged structure initialization** (`.read = sleepy_read, .write = sleepy_write`). **Unset entries are NULL, and the kernel applies a default** (often "not supported").
- **The book's prototypes are outdated: "RTSL — Read The Source, Luke!"** Use the **Linux kernel git source** (github.com/torvalds/linux; `git blame` and commit messages are very informative) and the **Bootlin Elixir cross-reference** (elixir.bootlin.com).
- **Avoid globals** in drivers. Keep per-device state in a structure linked from the `struct file`, so multiple opens don't interfere.
- **`struct file`** (filp):
  - **Not related to the C library's `FILE*`.** It **represents an *open file***: one specific open/close instance (an open file descriptor).
  - Fields: **`f_mode`** (readable/writable), **`f_pos`** (current read/write position), **`f_flags`** (O_RDONLY, **O_NONBLOCK**, O_SYNC), **`f_op`** (pointer to the fops), and **`void *private_data`** (your driver's per-open data pointer).
- **`struct inode`** **represents a file in any state, not an open descriptor**. It contains `i_rdev` (the device number) and **`i_cdev`** (pointer to the `struct cdev`).
- **Char device registration (`struct cdev`)**:
  - Option 1: **`cdev_alloc()`** **obtains a standalone cdev structure at runtime**; then set `->ops`.
  - Option 2 (more typical): **embed a `struct cdev` in your own device struct** and call **`cdev_init(&dev->cdev, &fops)`** (initializes a previously allocated cdev).
  - Then **`cdev_add(&cdev, devno, count)`** **informs the kernel** that the device is live. **`cdev_del()`** removes it.
- **The open method** (`int open(struct inode *inode, struct file *filp)`) typically:
  - **checks for device errors** (not ready, hardware problems),
  - **initializes the device on first open**,
  - **updates `f_op`** if needed,
  - **allocates and fills `filp->private_data`**.
  - The prototype doesn't include the cdev. Get it from `inode->i_cdev`.
- **`container_of(ptr, type, member)`** **computes the address of the enclosing structure from a pointer to one of its members** (pointer arithmetic using the member's offset). The scull pattern:
  ```c
  dev = container_of(inode->i_cdev, struct scull_dev, cdev);
  filp->private_data = dev;   // for other methods
  ```
  So `private_data` is set to **the `scull_dev` that contains the cdev at `inode->i_cdev`**, *not* to the cdev address itself.
- **The release method** is the **reverse of open**: deallocate anything open allocated in `private_data`.
  - scull's release does **nothing**, because its private_data points to memory allocated in **init** (freed in module exit).
  - If a program forgets to close, **the kernel cleans up automatically at process exit**. Your driver is guaranteed **exactly one `release()` per `open()`**.
  - **After fork or dup, several processes may call `close()`, but only the *last* close triggers `release()`.** So **not every `close()` causes a release**, whereas **every `open()` does invoke the driver's open method**.

### 2.3 Device Driver and User Space, Continued
- **The full module load path**:
  1. `make` produces `mydriver.ko` with **unresolved addresses** (e.g., `register_chrdev`).
  2. **`insmod`/`modprobe` (optionally with params)** resolves them against the kernel symbol table.
  3. The **module entry function** calls `register_chrdev(…, &my_fops)`, which allocates a chrdev, **adds "241 mydriver" to `/proc/devices`**, and maps major 241 to the fops.
  4. The **load script** reads `/proc/devices` and runs `mknod /dev/mydriver c 241 0`.
  5. A user program's `open("/dev/mydriver")` follows the device number to the fops and calls your **open** function.
- **Kernel memory**:
  - **`kmalloc(size, flags)`**: use **`GFP_KERNEL`** for now (it may sleep). Free with **`kfree(ptr)`**.
  - **Why not malloc/free?** Those are **glibc** functions, which aren't available in kernel space.
  - **Allocate large buffers dynamically**, never on the small kernel stack.
- **read/write prototypes**: `ssize_t read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)` (write takes a `const char __user *`).
- **`buf` is a `__user` pointer: user-space memory that kernel code can't dereference directly.** Why not?
  - On some architectures it **may not be valid or accessible from the kernel at all**.
  - It **may be in a RAM page that isn't currently present** (paged out).
  - It was **supplied by an untrusted user program**, so direct access risks privilege escalation.
- **Access it with `copy_to_user(void __user *to, const void *from, n)` and `copy_from_user(void *to, const void __user *from, n)`**: like memcpy, but between kernel and user space, handling architecture details and faults. **They return the number of bytes *not* copied**; non-zero → `-EFAULT`.
- **Direction is from user space's point of view**: **read = kernel → user (`copy_to_user`)**; **write = user → kernel (`copy_from_user`)**. Return the byte count and **update `*f_pos`**.

### 2.4 Assignment 7: Circular Buffer, rootfs-overlay, Kernel Module Builds
- **Buildroot rootfs-overlay**: a directory whose contents are **copied over the rootfs** (adding or overriding files). **Use a relative path** (e.g., `../base_external/rootfs_overlay`) so it works outside your own directory.
- **Buildroot kernel-module packages**: `$(eval $(kernel-module))` + `$(eval $(generic-package))`, with **`<PKG>_MODULE_SUBDIRS = misc-modules scull`** to build modules from subdirectories (out-of-tree modules).
- **A7 Part 1**: implement a **circular buffer** in user space first (`aesd-circular-buffer.c`), because kernel debugging is hard.
  - Each **entry** stores a **`const char *buffptr` and a `size`**.
  - **`in_offs`** is where the next write goes; **`out_offs`** is the oldest entry. **in == out means empty *or* full**, so track a **`full` flag**.
  - **Add**: write at `in_offs` and advance it (wrapping around).
  - **When full, overwrite the oldest entry and advance `out_offs` too**, so the buffer always holds the **10 most recent** entries (`AESDCHAR_MAX_WRITE_OPERATIONS_SUPPORTED`). **Doesn't block.**
  - **`aesd_circular_buffer_find_entry_offset_for_fpos(buf, char_offset, &entry_offset_byte_rtn)`**: treats all entries concatenated from `out_offs` as one byte stream, and returns the entry containing `char_offset` plus the byte offset within it.
    - Example, entries "str1" (size 4) and "nextstr" (size 7): offset 2 → entry 0, byte 2 ('r'); offset 6 → entry 1, byte 2 ('x').
- **A7 Part 2**:
  - Fork **ldd3** and add your GitHub username to hello.c's printk.
  - Add an **`ldd`** Buildroot package building **misc-modules and scull**.
  - Add a rootfs-overlay init script **`S98lddmodules`** that loads scull (scull_load), faulty (module_load), and hello (`modprobe`) at start, and unloads them at stop.
  - Run **`echo "hello_world" > /dev/faulty`** and analyze the resulting **oops** in `faulty-oops.md`.
  - Validation: `lsmod` shows hello, faulty, scull; `/dev/scull*` and `/dev/faulty` exist; `S98lddmodules stop` unloads everything.

### 2.5 Kernel Debugging (LDD3 Ch. 4)
- **Why kernel debugging is hard**:
  - It's **not easily executed under a debugger**. **gdb can *analyze* a running kernel via `/proc/kcore`, but cannot halt it, set breakpoints, or modify memory.** (kgdb exists but needs setup.) So it's **not the typical first method**.
  - It's **not easily traced**.
  - Bugs are **often hard to reproduce**, especially timing-related ones, so design carefully up front.
  - **Bugs often crash the entire system and destroy evidence.**
- **Enable the "kernel hacking" options in menuconfig**:
  - `CONFIG_DEBUG_KERNEL`, **`CONFIG_DEBUG_SPINLOCK`** (lock debugging), **`CONFIG_DEBUG_INFO`** (needed for kgdb), **`CONFIG_MAGIC_SYSRQ`**, and others.
  - **They're usually off in production kernels because of their performance impact**; consider that before enabling them.
- **Printing is the most common debug method**: **`printk(KERN_DEBUG "msg\n")`**. Note there's **no comma** after the level macro, which is a string prefix.
  - **Levels 0–7: EMERG, ALERT, CRIT, ERR, WARNING, NOTICE, INFO, DEBUG.**
  - **Why not printf? No glibc.**
- **Where printk output goes**:
  - **klogd/syslogd** route it to **`/var/log/messages`** (or another file) **or the console**, depending on the syslog configuration.
  - **`dmesg`** prints the kernel ring buffer (from `/proc/kmsg`).
  - **`/proc/sys/kernel/printk`** *controls* which levels reach the console; it doesn't display messages. It holds 4 values: current (4 = WARNING and more severe), default (4), minimum (1 = ALERT and EMERG), boot-time (7 = DEBUG).
- **printk is safe anywhere, including interrupt handlers**: it just writes into a **circular buffer** that another task drains.
- **Debug-only prints** shouldn't ship in production:
  - **Option 1: a `PDEBUG` macro** controlled by `SCULL_DEBUG`. If **`SCULL_DEBUG` and `__KERNEL__` are defined**, it becomes `printk(KERN_DEBUG "scull: " …)` and goes **to the printk location set by syslog**. If SCULL_DEBUG is defined without `__KERNEL__` (user space), it becomes `fprintf(stderr, …)`. **If SCULL_DEBUG isn't defined, it expands to nothing**, so the text goes **nowhere**.
  - **Option 2: dynamic debug (`pr_debug`)**: turn individual prints on and off **at runtime** from the command line. **Benefit: no recompiling and no special debug builds.**
  - **`printk_ratelimited`** (replaces the book's `printk_ratelimit`) **keeps repeated failure prints from flooding the logs**.
- **strace** on a user program shows its driver interactions. For example, `cat /dev/scull0` shows **`openat(…"/dev/scull0"…) = 3`**, **`read(3, "hello_world\n", …) = 12`**, then `write(1, …) = 12` to stdout. It's the go-to tool for checking what your fops receive and return.
- **System faults**: a bug in a driver may cause a **panic** (the kernel stops) or leave the system **unusable**. **Unloading and reloading the driver may not be enough; you may need to reboot.**
- **Oops messages**: e.g., a **NULL pointer dereference** or invalid pointer use (a **page fault**). Reading one:
  - First line: "Unable to handle kernel NULL pointer dereference at virtual address 0000…".
  - The **PC/EIP** line gives the location, e.g. **`faulty_write+0x4/0x10 [faulty]`**: **4 bytes into `faulty_write`, which is 0x10 bytes long, in module `faulty`**. Then registers and the **call trace**.
  - Use **`objdump -S module.ko`** (source mixed with assembly; needs debug info) to find the faulting instruction. For an ARM target use the cross objdump, e.g. **`buildroot/output/host/bin/aarch64-linux-objdump`**.

> **Module 2 Cheat-Sheet Candidates:** major = driver, minor = device; static vs dynamic majors; /proc/devices + awk + mknod; dev_t/MKDEV/MAJOR/MINOR; register_chrdev ↔ unregister_chrdev; fops maps functions to syscalls; struct file = open instance (f_pos, f_flags, private_data) vs inode = file in any state (i_cdev); cdev_alloc vs cdev_init vs cdev_add vs cdev_del; container_of; one release per open (last close); kmalloc(GFP_KERNEL)/kfree; __user → copy_to_user/copy_from_user (return bytes NOT copied); read = to user, write = from user; circular buffer semantics; gdb + /proc/kcore = read-only analysis; debug configs off in production; printk levels 0–7, dmesg, /var/log/messages, console; /proc/sys/kernel/printk controls the console level; PDEBUG behavior; dynamic debug; oops "func+off/len [module]"; objdump -S.

---

<a id="module-3"></a>
## Module 3 — Kernel Concurrency & Locking, Assignment 8

### 3.1 Concurrency and Race Conditions (LDD3 Ch. 5)
- Concurrency bugs are "**the easiest to create and some of the hardest to find**", and **SMP (symmetric multiprocessing) made them ubiquitous**.
  - **SMP** = multiple CPUs **sharing the same RAM**. **AMP** (asymmetric) = each CPU has its own RAM.
  - Why not use AMP to avoid the problem? It's **generally not supported by Linux** (and harder to write an OS for).
- **The scull race example** (from a write path):
  ```c
  if (!dptr->data) {                                   // 1
      dptr->data = kmalloc(quantum, GFP_KERNEL);       // 2
      if (!dptr->data) goto out;                       // 3
  }
  ```
  - **kmalloc with GFP_KERNEL can sleep**, so process A can be switched out *inside* kmalloc.
  - Process B then also sees `data == NULL` and kmallocs too. Both assign `dptr->data`; **the last writer wins** and **the other allocation is leaked** (never kfree'd).
- **"I'll ignore it because it's so unlikely"? No.**
  - **"One-in-a-million events can happen every few seconds"** at GHz clock speeds.
  - **Probabilities change drastically and unexpectedly** when hardware or other software changes.
  - "**It usually happens the first time your boss (or customer) runs your software.**"
- **Sources of concurrency in the kernel**:
  - **multiple user-space processes**,
  - **SMP**: your code runs **simultaneously on different CPUs**,
  - **kernel code is preemptible**: driver code can lose the CPU at any time,
  - **device interrupts are completely asynchronous**,
  - **your device can disappear while you're using it** (hot-unplug).
- **Avoiding race conditions**:
  - **Avoid shared resources** where possible (e.g., avoid globals).
  - But **no globals doesn't mean no sharing**: allocating memory and passing the pointer to the kernel creates sharing too.
  - Ask: **is the resource shared beyond a single thread of execution?** And **could a thread see an inconsistent view of it?** If both answers are yes, **you must explicitly manage access with locking / mutual exclusion**.
- **Kernel resource rules**:
  - **No object can be made available to the kernel until it can function properly.** Initialize your data **and locks** before registering (e.g., before `cdev_add`).
  - **Once the kernel knows about an object, it must keep existing and functioning until no outside references remain**, so **references must be tracked** (often the kernel tracks them for you).
  - So the claim "allocate an object, give the kernel a reference, then initialize it" is **false**.

### 3.2 Critical Sections and Semaphores
- **The goal of a critical section is to make non-atomic operations atomic** (all at once as far as other threads are concerned).
  - `a |= b;` is **typically not atomic** (load, OR, store).
  - `a++;` is **typically not atomic**.
  - `a = b;` **may or may not be**, depending on the architecture and variable size (e.g., a 64-bit value on a 32-bit CPU takes two stores).
- **Choosing a primitive starts with: is it safe to sleep in this critical section?**
  - **Semaphores and mutexes** work with sleep (a waiter sleeps).
  - **Spinlocks** work in all cases, **including when you can't sleep**.
  - **Reasons it's unsafe to sleep**: the code **may be called from an interrupt handler**; **latency requirements**; **holding other critical resources** (e.g., a spinlock).
  - Also ask: **will my critical section call a function that may sleep?** **kmalloc(GFP_KERNEL) can sleep.**
- **Semaphores are the right choice when** you need mutual exclusion **and the process may sleep while holding it** (the critical section sleeps or calls functions that may sleep).
- **Semaphore = an integer + P() and V()**:
  - **P()**: if value > 0, decrement and continue; **if value ≤ 0, block until it's > 0**.
  - **V()**: **increment**.
  - **As a mutex**: initialize to **1** (exactly one holder). P = lock, V = unlock.
  - That's the old way; **the kernel now has a dedicated `struct mutex`**.
- **Initialization**:
  - Option 1: **`DEFINE_MUTEX(name)`** / `DEFINE_SEMAPHORE`, typically a global.
  - Option 2 (scull): **embed it in your struct and call `mutex_init(&dev->lock)` / `sema_init(&sem, val)` at runtime**, **before** notifying the kernel (`cdev_add`).
- **Acquiring (P)**:
  - **`down(&sem)`**: blocks until available (uninterruptible).
  - **`down_interruptible(&sem)`**: **returns 0 if you got it, non-zero if interrupted by a signal** (e.g., SIGTERM, SIGKILL), in which case **you do NOT hold the semaphore**, so check the return value. **Use the interruptible versions in most cases**, otherwise a stuck process can't be killed.
  - Exception (2008 guidance): **file I/O, especially filesystem drivers, should stay non-interruptible** because existing applications assume it. **Everything else should be interruptible if possible.**
  - **`down_trylock(&sem)`**: **never sleeps**; **0 = obtained, 1 = someone else holds it**.
  - A thread that completed `down` "**holds**" / has "**taken out**" / has "**acquired**" the semaphore.
- **Releasing (V)**: **`up(&sem)`**. After it returns you no longer hold the semaphore.
  - **One `down` → exactly one `up`.** So **"call `down` to release it" is wrong**, and so is "up at least once": it's **up exactly one time**.
  - **You must release in every error path too** (goto-style cleanup helps).

### 3.3 Mutexes, Reader/Writer Semaphores, Completions
- **Mutex** (added to the kernel after LDD3; the course's scull uses it). Mapping to semaphores:
  - **`mutex_lock` = down**
  - **`mutex_trylock` = down_trylock**
  - **`mutex_lock_interruptible` = down_interruptible**
  - **`mutex_unlock` = up**
  - `mutex_lock_nested` handles lock ordering between multiple locks; `mutex_is_locked` checks the state.
  - scull initializes it with **`mutex_init()` in module init, before `scull_setup_cdev`/`cdev_add`**.
- **When an interruptible lock fails**:
  - Return **`-ERESTARTSYS`** when **retrying the operation is the right thing to do** (undo any user-visible changes first).
  - Return **`-EINTR`** when **you can't undo** what was already done.
  - **Balance every lock with an unlock**, including on error (e.g., release lock 1 if lock 2 was interrupted).
- **Reader/writer semaphores (`rw_semaphore`)**:
  - How many threads can read a non-atomic variable while writes are blocked? **Infinite.** While a write is in progress? **0.**
  - So **only writers need exclusive access**: an rwsem allows **one writer *or* any number of readers**.
  - **`down_read`/`up_read`, `down_write`/`up_write`**. **`down_read_trylock`/`down_write_trylock` return 1 on success** (opposite of the usual convention).
  - **`downgrade_write`** converts a held write lock into a read lock. **Writers get priority.**
  - **Use them when write access is rarely needed and held briefly** (i.e., **reads are much more common than writes**, not the other way around).
- **Completions (`struct completion`)**: "**wait for some activity to complete**".
  - **Semaphores are optimized for the *available* case** (the lock is usually free). **Completions are optimized for the *not available* case**, e.g. you just started a thread and wait for it to finish.
  - So "completions are optimized for when the lock is available" is **false**.
  - **`complete()`/`complete_all()` can be called from interrupt handlers.**
  - "Any time you think of using **`yield()` or a quirky `msleep(1)` loop**, you probably want `wait_for_completion()` and `complete()`."
  - **API**: `DECLARE_COMPLETION(c)` / `init_completion(&c)` → pass it to a worker → **`wait_for_completion(&c)`** (variants: `_interruptible`, `_timeout`); the worker calls **`complete(&c)`**, which wakes one waiter.
  - **`complete_all()`** wakes **all current *and future* waiters**: anyone who waits afterwards returns immediately.

### 3.4 Spinlocks, Locking Rules, Lock Ordering
- **Spinlocks**:
  - **Usable in code that cannot sleep, e.g. interrupt handlers**, where they're **the only option**.
  - **Higher performance than semaphores when used properly** (short holds).
  - Conceptually **a single bit + an atomic test-and-set + a tight loop spin**: **waiting CPUs busy-loop**, they **don't sleep**.
  - So "spinlocks let the processor sleep while waiting" is **false**: that describes semaphores.
- **`spin_lock_irqsave(&lock, flags)` / `spin_unlock_irqrestore(&lock, flags)`**:
  - **Always safe in interrupt or non-interrupt context.** It disables local interrupts if they were enabled and saves the previous state in `flags`; irqrestore re-enables them only if they were enabled before.
  - **Use it whenever the lock may also be taken in an interrupt.**
- **Plain `spin_lock`/`spin_unlock`** don't touch interrupts. They're only safe if **the lock is never used in an interrupt**, or interrupts are known to be blocked.
  - **If an interrupt on the same CPU tries to take a lock already held with plain spin_lock → deadlock** (it spins forever, and the holder can't run).
- **Spinlocks on uniprocessor systems?** The *code* is still needed on any **preemptible** kernel: on UP, spin_lock disables preemption, and irqsave disables interrupts. **"Spinlocks are only needed on multiprocessors" is false.**
- **Atomic-context rules while holding a spinlock**:
  - **Core rule: the code must be atomic.** **It can't sleep** and can't give up the CPU (except to interrupts that don't take this lock).
  - **How do you know a called kernel function won't sleep?** It's hard to know. **Most functions that may allocate memory may also sleep.**
  - **Second rule: hold it for the minimum time possible**, because waiters burn CPU spinning.
- **Choosing a primitive**:
  - A resource accessed **briefly, calling no other kernel functions** → **spinlock**.
  - **Allocating memory (may sleep) while holding the lock** → **mutex** (never a spinlock).
  - A lock **used from an interrupt** → **spin_lock_irqsave**.
- **Locking rules**:
  - **They must be unambiguous.** **Define a lock to control access to specific data.**
  - **Design locking in from the beginning.** Don't wait for problems to appear and then add locks.
  - **Clearly enforce the rules** so nested functions don't try to acquire the same lock (self-deadlock).
  - **Write functions that assume the caller holds the lock, and document that assumption explicitly.**
  - Applied to the `do_something`/`do_something_else` quiz: **acquire the lock once in `somekernelfunc()` around the `do_something()` call**, and have do_something and do_something_else require the caller to hold it. Locking inside both functions would self-deadlock on the nested call.
- **Lock ordering (deadlock)**:
  - Thread 1 does `lock1` then `lock2`; Thread 2 does `lock2` then `lock1`. If thread 1 holds lock1 and thread 2 grabs lock2, **they deadlock**.
  - **Always acquire locks in the same order; release in the opposite order.** Kernel lock ordering is poorly documented, so read the source.
  - **Rules of thumb**:
    - **Avoid needing multiple locks whenever possible.**
    - **Take your driver's locks before locks used elsewhere in the kernel**, to minimize the time you hold the "popular" lock.
    - **Take semaphores/mutexes before spinlocks**, because a semaphore may sleep and you can't sleep while holding a spinlock.
- **Alternatives to locking**:
  - **Atomic variables (`atomic_t`) and atomic bit operations** (`test_and_set_bit`), guaranteed atomic on all architectures. (Caveat from the quiz: operations on a *single* atomic value need no extra lock, but **a sequence of operations or several related values does**.)
  - **Lock-free algorithms**:
    - **A circular buffer with exactly 2 threads (one reader, one writer) and atomic counts can work without locks.**
    - **RCU (Read-Copy-Update)**: readers keep using the old copy while a writer publishes a new one; **old copies are cleaned up when the last reference is released**.

### 3.5 Assignment 8: `/dev/aesdchar` Char Driver
- **Requirements**:
  - **Allocate memory (kmalloc) for each write command** as it arrives; any length kmalloc can handle.
  - **Commands are `\n`-terminated.** **Writes without `\n` are saved and appended to by later writes**, and only added to the circular buffer once terminated.
  - **Keep the 10 most recent commands**; **free the memory of older ones**.
  - **Ignore the write position / f_pos on write.**
  - **read returns the stored commands in order**, using **f_pos to find where to start** and **honoring `count`**.
  - **Lock properly** for multi-thread and multi-process safety; a full write must complete before the next is accepted.
- **Freeing memory**: you must free when **adding to an already-full buffer** (the overwritten entry). **Suggestion: have `aesd_circular_buffer_add_entry()` return the replaced entry's `buffptr` (or NULL)**, so the caller can kfree it.
- **`aesd-circular-buffer.c/.h` use `#ifdef __KERNEL__`** to include either kernel or user headers, so the **same code compiles for the unit test and for the driver**. User-space debugging is far easier.
- **Using `PDEBUG`** (the `AESD_DEBUG` switch) prints via printk or stderr.
- **`struct aesd_dev`**: holds the **circular buffer**, a **working `aesd_buffer_entry` for the partial (unterminated) write**, and a **lock** (e.g., a mutex, since you kmalloc while holding it). **Initialize all of it in `aesd_init_module`**, and undo all of it in cleanup (free entries, etc.).
- **open**: `dev = container_of(inode->i_cdev, struct aesd_dev, cdev); filp->private_data = dev;`.
- **read**:
  - Get `dev` from `filp->private_data`; **`buf` needs `copy_to_user`**; `count` = max bytes; **`*f_pos` = char_offset** into the concatenated contents. Look it up with `find_entry_offset_for_fpos`, copy, and **advance `*f_pos`**.
  - **Return values**: `== count` (everything transferred); `0 < ret < count` (partial); **`0` = EOF**; negative = error (**`-ERESTARTSYS`, `-EINTR`, `-EFAULT`**).
  - **Partial read rule**: **return just one entry (or part of one) per call**. As long as the return value and `*f_pos` are right, user space (cat/fread) calls again until you return 0.
- **write return values**: `== count` (success); positive but less than count (partial, caller may retry); **`0` = nothing written, caller retries**, which is why an app **hangs on its first write** when write wrongly returns 0; negative = error (**`-ENOMEM`, `-EFAULT`**).
- **drivertest.sh**:
  - `echo "write1"` … `"write10"` > /dev/aesdchar, then compares `cat` output with an expected file built with a **heredoc (`<< EOF`)**.
  - An **11th write rotates out write1**.
  - **`echo -n`** partial writes must combine into one command.
- **aesdsocket changes**: build switch **`USE_AESD_CHAR_DEVICE=1`** removes the timestamps and reads/writes `/dev/aesdchar` instead of `/var/tmp/aesdsocketdata`. **Open the device only when accessing it**: holding it open prevents `rmmod` ("resource temporarily unavailable").
- **Debug tips**:
  - Interactive debuggers are time-consuming ("**Linus is not a fan**"). **kdb/kgdb** are now built in (`CONFIG_KGDB_KDB`): kdb is a shell, usually over a serial port, entered automatically after an oops or with **SysRq-G**. Other tools: Linux Trace Toolkit, SystemTap (replaced dynamic probing); User Mode Linux is unmaintained.
  - **debugfs**: info for developers, "**not a stable ABI**". **/proc** is out of favor for new work (meant for process info). **sysfs** is highly organized.
  - **kmemleak**: enable "kernel memory leak detector" in `make linux-menuconfig`, then `echo scan > /sys/kernel/debug/kmemleak` and `cat` it. Verify first with `kmemleak-test`.
  - **`bash -x` / `sh -x`** prints each command before executing it. **`strace -o file -f ./aesdsocket`** shows driver read/write calls and returns (e.g., `write(1,"write1\n",7) = -1 ENOMEM` means your write returned -ENOMEM).
- **FAQ**:
  - **read `count` = 131072 (0x20000)**: just user space's default buffer size.
  - **read called forever**: you **never return 0** at EOF.
  - **`fseek`/`stat` don't work on `/dev/aesdchar`**: no seek support, and it's not a regular file. Read in a loop until EOF instead.
  - **System hangs after adding locking**: you **returned while holding the mutex**. Unlock on every path (goto).
  - **Oops address math**: `objdump -S aesdchar.ko`, then function start (0x669) + offset (0x1fc) = 0x865, the faulting line.
  - **`sudo echo x > /dev/aesdchar`**: the redirect runs as *you*. Fix the device permissions, or add your user to the **staff** group (`sudo usermod -a -G staff $USER`).
  - **Work in stages**: simplest version (1 write = 1 command) → test with echo/cat → **commit and push** → add locking → repeat → add partial writes. **If the driver crashes, look very closely at what changed since the last working commit.** Take **VM snapshots**.

> **Module 3 Cheat-Sheet Candidates:** SMP vs AMP; scull kmalloc race (leak); sources of concurrency (5); "both yes → lock"; init before exposing to the kernel; a++/a|=b not atomic, a=b depends; sleep-safe? → mutex/semaphore, can't sleep → spinlock; kmalloc(GFP_KERNEL) sleeps; semaphore P/V; down/down_interruptible (0 = got it)/down_trylock (0 = got it); one down ↔ one up; interruptible by default (except filesystem I/O); -ERESTARTSYS vs -EINTR; rwsem (1 writer or N readers, read-mostly, trylock returns 1); completions (not-available case, complete_all includes future waiters); spinlock = busy-wait, never sleep, short holds; irqsave whenever an ISR uses the lock; same-CPU ISR + plain spin_lock = deadlock; lock in the caller and document it; same-order acquire / reverse release; driver locks before kernel locks; semaphores before spinlocks; atomic_t/bitops, lock-free 2-thread ring, RCU; A8 read/write return-value rules, partial read rule.

---

<a id="module-4"></a>
## Module 4 — ioctl, Sleeping, poll/select, Assignment 9

### 4.1 ioctl (LDD3 Ch. 6)
- **What it is**: **a system call on a file descriptor that is passed to the driver**. It identifies **a command to perform plus an optional argument (typically a pointer)**. You can't issue it from the shell with cat or echo; you **need a program** (C, Python…).
- **The common interface for device control**: requests other than read/write, e.g. **lock a door, eject media, report information**. It's "out-of-band" control.
- **User prototype**: `int ioctl(int fd, unsigned long request, ...);`. **The `...` isn't really varargs**: it's **a single optional argument `argp`**, historical baggage from before `void *` existed. The **`request` describes how argp is used**.
- **It's the easiest and most straightforward choice, but it has fallen "out of favor among kernel developers"**: these are **unstructured system calls** that are **difficult to audit** and **often poorly documented**. So "ioctl is the mechanism most kernel developers recommend" is **false**.
  - **Alternatives**: **embed commands in the data stream** (e.g., `echo "start" > /dev/dev`, using write), or **virtual filesystems like sysfs**.
- **The fops signature changed (2005)**: the book's `ioctl(inode, filp, cmd, arg)` is gone. Now there's **`long (*unlocked_ioctl)(struct file *, unsigned int cmd, unsigned long arg)`**, and the inode is available at **`filp->f_inode`**.
  - **`compat_ioctl`** handles **32-bit programs on 64-bit kernels**: arg is always passed as an unsigned long, **which assumes `sizeof(unsigned long) == sizeof(void*)`**.
  - In A9 you **must implement `unlocked_ioctl`** (compat alone isn't enough). With 6.x kernels also set `.compat_ioctl = compat_ptr_ioctl`.
  - "Inappropriate ioctl for device" usually means unlocked_ioctl is missing or the `/dev` node isn't the right device.
- **Driver implementation**: a **`switch (cmd)`** over the supported commands.
- **Choosing command numbers**: **don't just use 0, 1, 2…**. Use the **`_IO` macros**, which encode a driver-specific **magic number** so **a valid command for one driver isn't accepted by the wrong device**.
  - Pick the magic number from `Documentation/userspace-api/ioctl/ioctl-number.rst` (scull uses `'k'`).
  - **Avoid magic `'T'`**: some ioctls are **decoded by the kernel before they reach your driver**.
- **The macros pack 4 fields into `cmd`: magic number (type), sequence (command) number, data-transfer direction, and argument size.**
  - **`_IO(magic, nr)`**: no data transfer (e.g., `SCULL_IOCRESET`).
  - **`_IOR(magic, nr, type)`**: `_IOC_READ`, **kernel → user**.
  - **`_IOW(magic, nr, type)`**: `_IOC_WRITE`, **user → kernel** (e.g., `SCULL_IOCSQUANTUM` with `int`).
  - **`_IOWR`**: both directions.
  - **Direction is from user space's point of view.**
- **Using the user pointer argument**:
  - **`_IOC_READ`** (data goes to user space): **check that the user memory is safe to *write*, then copy kernel → user** (`copy_to_user`; older code used `access_ok(VERIFY_WRITE,…)`).
  - **`_IOC_WRITE`** (data comes from user space): **check that the user memory is safe to *read*, then copy user → kernel** (`copy_from_user`).
- **For devices that don't transfer data**, writing control strings with **write()** can be simpler (scriptable with echo), though it may complicate the driver. *(So "ioctl is the best method for devices that don't transfer data" is false; the book suggests write as the alternative.)*

### 4.2 Sleeping in the Kernel
- **Blocking I/O**: when a driver can't satisfy a request yet, it **blocks the process, which sleeps until the request can proceed**. **Sleeping = removed from the scheduler's run queue**, waiting for some state change.
- **Sleeping rules**:
  1. **Never sleep while holding a spinlock, seqlock, or RCU lock.** Waiters for those spin and burn the CPU, which risks deadlock.
  2. **Never sleep with interrupts disabled**: interrupt latency would suffer.
  3. **Avoid sleeping (or keep it short) while holding a semaphore or mutex**, and consider whether it could deadlock. It's *allowed*, because other waiters sleep rather than spin. It's safe only if **nothing trying to take that semaphore would prevent your wake-up condition**. **You can't be sure that stays true in the future, so avoiding it is best.** Lock data, not whole functions.
  4. **Make no assumptions about the system's state after waking up.** Re-check conditions and retry locks, then sleep again if needed.
  5. **Before sleeping, make sure some *other* process, interrupt, or kernel event will wake you up.** **Code in your own (sleeping) thread can't.**
  6. **Never require atomic context**: atomic context is exactly where you *can't* sleep.
- **Quiz framing**: "**do not sleep holding a spinlock** / **do not sleep with interrupts disabled**, under any circumstances"; "**sleep holding a semaphore/mutex only after careful evaluation**".
- **Wait queues**:
  - **`wait_queue_head_t q; init_waitqueue_head(&q);`** (or `DECLARE_WAIT_QUEUE_HEAD`).
  - **`wait_event(q, condition)`**, **`wait_event_interruptible(q, condition)`** (exits on signals; returns non-zero, usually → `-ERESTARTSYS`), and the `_timeout` variants.
  - **The condition is an arbitrary boolean *expression*, re-evaluated by macro magic** every time the task wakes. It's **checked before sleeping**, so you never sleep if it's already true.
- **Waking**:
  - **`wake_up(&q)`** wakes **all** waiters on the queue. **`wake_up_interruptible(&q)`** wakes only **interruptible** sleepers.
  - It must be called by **a different process or an interrupt**.
  - In scullpipe: **wake the reader queue in write()** (new data), and **wake the writer queue in read()** (space freed).
- **scullpipe pattern**:
  ```c
  while (dev->rp == dev->wp) {           /* nothing to read */
      mutex_unlock(&dev->lock);          /* don't sleep holding the lock */
      if (filp->f_flags & O_NONBLOCK) return -EAGAIN;
      if (wait_event_interruptible(dev->inq, dev->rp != dev->wp))
          return -ERESTARTSYS;
      if (mutex_lock_interruptible(&dev->lock)) return -ERESTARTSYS;
  }
  ```
- **Input/output buffering** in drivers means blocking is on the **buffer**, not the device. **That matters most when device transfers are much slower than memory.**
- **How a process sleeps**:
  - **Task states**: `TASK_RUNNING` (runnable, not necessarily executing), and **`TASK_INTERRUPTIBLE` / `TASK_UNINTERRUPTIBLE`** (the two kinds of sleep).
  - **`schedule()`** runs the scheduler and **yields the CPU** to another process, which effectively puts the caller to sleep. **It does NOT wait for your specific condition**; the wait_event loop re-checks that.
  - The internals are `prepare_to_wait_event()` → check the condition → `schedule()` → `finish_wait()`.
- **Non-blocking I/O**:
  - If user space opened with **`O_NONBLOCK`**, it's visible in **`filp->f_flags`**. Instead of sleeping, return **`-EAGAIN`**, and user space polls.
  - Test with the **`nbtest`** program / `nonblock.sh` (sets stdin and stdout to O_NONBLOCK); strace shows repeated `EAGAIN`.
- **When to block**:
  - **read** should block when **no data is available** (with no O_NONBLOCK). **If some data is available, even less than requested, return it** rather than blocking.
  - **write** should block when **no space is available** in the driver's buffer. If there's *some* space, accept what fits rather than blocking.
  - With **O_NONBLOCK, return -EAGAIN** instead of blocking.

### 4.3 poll/select, Seeking, Access Control
- **`poll`/`select` (and epoll)** let user space ask **whether a read or write would block**, or wait for an fd to become ready. They're **implemented by the driver's `poll` method**: `__poll_t (*poll)(struct file *, struct poll_table_struct *)`.
  - **Call `poll_wait(filp, &queue, wait)` on every wait queue that could change the poll status** (e.g., inq and outq). The kernel handles the waiting.
  - **Return a bitmask of the operations currently possible.**
- **Read rules**:
  - **Data available**: read returns **immediately with at least 1 byte**; poll returns **`POLLIN | POLLRDNORM`**.
  - **No data**: read **blocks until ≥1 byte**, or returns **`-EAGAIN` with O_NONBLOCK**; poll reports **unreadable** (read flags all 0).
  - **End of file**: read returns **0 immediately**; poll reports **`POLLHUP`**.
- **Write rules**:
  - **Space available**: write returns **without delay, accepting ≥1 byte**; poll reports **`POLLOUT | POLLWRNORM`**.
  - **Buffer full**: write **blocks until space is freed**, or **`-EAGAIN` with O_NONBLOCK**; poll reports **not writable**.
  - **Never make write wait for the data transfer to the device itself, even without O_NONBLOCK.** **To guarantee data reached the device, the driver provides `fsync`.** (So "fsync is only needed for O_NONBLOCK drivers" is **false**.)
- **Seeking**:
  - **The default lseek just updates `filp->f_pos`.** Implement the **`llseek`** fop for custom behavior. (Strictly, a driver doesn't *have* to implement llseek to support seeking, since the default works if read/write honor f_pos. But custom seek semantics, like A9's, need it.)
  - **Call `nonseekable_open()`** in open when seeking makes no sense, i.e. a **data flow rather than a data area** (serial port, keyboard).
- **Access control in open**:
  - **Single-open** (scullsingle): take an atomic in open, release it in release.
  - **Single-user**: compare the process UID in open; **return `-EBUSY`** while someone else is using it.
  - **Alternative: block in open** until it's available.
  - **Device unavailable → return `-EBUSY` or block.** *Not* `-EPERM`, which means permission denied.

### 4.4 Assignment 9: llseek + ioctl in aesdchar
- **`f_pos`** = "the current reading or writing position", a **64-bit signed byte offset** (`loff_t`).
  - It's **always 0 in write when you use `echo x > /dev/aesdchar`**, because `>` opens with truncation and starts at offset 0.
  - **Why does it exist in two places?** The **`*f_pos` argument is updated by your read/write implementation**; **`filp->f_pos` is updated by the shared VFS code** based on read/write. Normally only **`llseek`** modifies filp->f_pos directly; A9 also does it in ioctl, as a testing convenience.
- **Seek semantics**: `SEEK_SET` (absolute), `SEEK_CUR` (relative), `SEEK_END` (from the end). **`_llseek` exists alongside `lseek` to guarantee long long (64-bit) offsets.**
- **The default llseek modifies `filp->f_pos`** if the method is missing. **For lseek to work, read and write must use and update the offset argument**: `*f_pos += retcount`.
- **Implementation options**:
  1. Leave llseek NULL (would need seek support in write: hard).
  2. **Your own llseek with locking and logging, using `fixed_size_llseek(filp, off, whence, size)` with size = total bytes in the circular buffer** (recommended).
  3. Hand-code every whence case.
- **Example**: stored writes "Grass\n" (6), "Sentosa\n" (8), "Singapore\n" (10) give offsets **0–5, 6–13, 14–23**. **Offset 15 = 'i' in "Singapore".**
- **ioctl `AESDCHAR_IOCSEEKTO`** (defined in the shared **`aesd_ioctl.h`**: magic number, `_IOWR`, `struct aesd_seekto { uint32_t write_cmd; uint32_t write_cmd_offset; }`):
  - **The driver uses `copy_from_user`** to get the struct, then **validates** it: return **`-EINVAL`** if `write_cmd` **hasn't been written**, is **out of range (≥10)**, or **`write_cmd_offset ≥ command length`**.
  - It **sums the lengths of the entries from `out_offs` up to `write_cmd`**, **adds `write_cmd_offset`**, and saves the result in **`filp->f_pos`**. (cmd 0 offset 2 in "Grass" = 'a'.)
- **aesdsocket**: when it receives **`AESDCHAR_IOCSEEKTO:X,Y`**, it **issues the ioctl (doesn't write the string to the device)**, then reads the device from the new position **using the same fd** and sends the data back.
  - Use **`fileno(FILE*)`** to get an fd from a FILE*.
  - **Open `/dev/aesdchar` only when it's accessed** (holding it open breaks `rmmod`).
- **Testing**: seek from the command line with **`dd`** (`skip=`/`seek=`), which issues llseek. New `assignment9/drivertest.sh` and `sockettest.sh`; the old tests must still pass.

> **Module 4 Cheat-Sheet Candidates:** ioctl = fd syscall + cmd + optional arg, needs a program, out of favor (hard to audit, undocumented) → alternatives: data stream / sysfs; unlocked_ioctl(filp, cmd, arg), inode at filp->f_inode, compat_ioctl for 32-on-64; cmd = magic + nr + dir + size via _IO/_IOR(k→u)/_IOW(u→k)/_IOWR; magic separates drivers, avoid 'T'; _IOC_READ → verify write access + copy_to_user; _IOC_WRITE → verify read access + copy_from_user; sleep rules (never with spinlock or IRQs off; careful with semaphores; re-check state after wakeup; someone else must wake you); wait_event(_interruptible)(q, cond) re-evaluates cond; wake_up vs wake_up_interruptible; reader woken by write, writer woken by read; schedule() just yields; O_NONBLOCK → -EAGAIN; poll_wait + mask (POLLIN|POLLRDNORM, POLLOUT|POLLWRNORM, POLLHUP); never block write on device transfer → fsync; default llseek sets f_pos, nonseekable_open for streams; open busy → -EBUSY or block; A9 offsets, fixed_size_llseek, -EINVAL validation.

---

<a id="quick-reference"></a>
## Quick-Reference Tables

### Module tools
| Tool | Purpose |
|---|---|
| `insmod x.ko [param=val]` | Link a module into the running kernel (resolves symbols in an in-memory copy) |
| `modprobe x` | Same, **plus loads dependencies** |
| `rmmod x` | Remove a module (must not be in use) |
| `lsmod` | List modules (from /proc/modules) |
| `dmesg` | Kernel log ring buffer |
| `cat /proc/devices` | Allocated majors → driver names |
| `mknod /dev/x c MAJ MIN` | Create a char device node |
| `objdump -S x.ko` | Disassembly + source, for oops analysis |
| `strace -f -o out ./prog` | Trace syscalls between a program and the driver |

### Locking primitive selection
| Situation | Use |
|---|---|
| Critical section may sleep (kmalloc GFP_KERNEL, copy_to_user) | **mutex** (or semaphore) |
| Can't sleep / very short / no kernel calls | **spinlock** |
| Lock also used in an interrupt handler | **spin_lock_irqsave / irqrestore** |
| Many readers, rare brief writes | **rw_semaphore** (down_read/down_write) |
| Wait for another thread or IRQ to finish something | **completion** (wait_for_completion / complete) |
| Single integer counter or flag | **atomic_t** / bit ops |
| One reader + one writer ring buffer | lock-free circular buffer |
| Read-mostly pointer-based data | **RCU** |

### Semaphore / mutex API mapping
| Generic | Semaphore | Mutex | Return value |
|---|---|---|---|
| P (lock) | `down` | `mutex_lock` | void |
| P, interruptible | `down_interruptible` | `mutex_lock_interruptible` | 0 = acquired, non-zero = interrupted |
| P, try | `down_trylock` | `mutex_trylock` | sem: 0 = acquired / mutex: 1 = acquired |
| V (unlock) | `up` | `mutex_unlock` | void |
| rwsem try | `down_read_trylock`/`down_write_trylock` | – | **1 = acquired** |

### Driver method return conventions
| Method | Success | Partial | Special | Errors |
|---|---|---|---|---|
| read | `== count` | `0 < n < count` (partial read rule) | **0 = EOF** | -ERESTARTSYS, -EINTR, -EFAULT, -EAGAIN (non-blocking) |
| write | `== count` | `0 < n < count` (caller retries) | 0 = nothing written (caller retries → app "hangs") | -ENOMEM, -EFAULT, -EAGAIN |
| open | 0 | – | – | -EBUSY (in use), -ENODEV… |
| ioctl | 0 / value | – | – | -EINVAL (bad args), -ENOTTY (unknown cmd), -EFAULT |

### Key structs
| Struct | Represents | Key members |
|---|---|---|
| `file_operations` | Driver method table ("fops") | .open .release .read .write .llseek .unlocked_ioctl .poll |
| `file` (filp) | One **open** instance | f_mode, f_pos, f_flags, f_op, private_data, f_inode |
| `inode` | A file in any state | i_rdev (dev_t), i_cdev |
| `cdev` | Char device registered with the kernel | cdev_alloc / cdev_init / cdev_add / cdev_del |
| `dev_t` | Major + minor (32-bit) | MKDEV / MAJOR / MINOR |

### Yocto vs Buildroot vocabulary
| Concept | Buildroot | Yocto |
|---|---|---|
| Build tool | make + Kconfig | bitbake (Python/Bash) |
| Package definition | Config.in + pkg.mk | recipe .bb (+ .inc, .bbappend) |
| Custom content location | br2-external (base_external) | layer (meta-aesd) |
| Target selection | defconfig (qemu_aarch64_virt_defconfig) | MACHINE in conf/local.conf |
| Local source override | `PKG_OVERRIDE_SRCDIR` | `devtool modify` |
| Init system default | BusyBox init (S##scripts in /etc/init.d) | SysV init (update-rc.d class) |
| License of build system | GPLv2 | MIT |
| Started | 2001 | 2011 |
