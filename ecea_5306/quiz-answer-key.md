# ECEA 5306 — Quiz Answer Key
### Linux Kernel Programming and Introduction to Yocto Project

Answers to the six graded quizzes: Intro to Device Drivers, Building and Running Kernel Modules, Character Device Drivers, Kernel Debugging Techniques, Concurrency, Advanced Char Driver Operations. The downloaded files include only the questions, so these answers were worked out from the lectures, slides, and LDD3. Each answer has a short **why** to study from. ⚠️ marks answers where the wording is ambiguous; the reasoning is spelled out.

> **Note on use:** The course's "Full AI" policy allows AI tools on coursework if you **document the usage**. Cite this key if you use it on a quiz you haven't submitted. AI tools are **not** allowed on the proctored final.

---

## Quiz 1 — Intro to Device Drivers

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | A driver's role is to provide mechanism, not policy | **True** | LDD3 Ch. 1, the core design principle. |
| 2 | A programmer should write driver code that enforces a specific way applications use the hardware | **False** | That's policy. Drivers should impose as few restrictions as possible. |
| 3 | Roles performed by the kernel (select all) | **Creating and destroying processes; Setting up virtual memory addressing for processes; Supporting an ext3 filesystem; Controlling an attached keyboard; Dispatching networking packets** | These are process management, memory management, filesystems, device control, and networking. An **ftpd server, a display session manager, and the tunelp utility are user-space applications.** |
| 4 | Features can be added to the kernel at runtime using… | **modules** | A module is kernel code added at runtime. rmmod *removes* modules. |
| 5 | Driver writers typically enforce security policy in the driver | **False** | Drivers usually don't encode security policy; the admin controls access through device file permissions (exceptions: global resources such as IRQ lines and firmware updates). |
| 6 | Boss wants a closed-source driver. Your response? | **You recommend discussing with legal counsel** | Binary-only modules are "deliberately ambiguous" under GPLv2, so seek legal advice. The other options state the law as settled one way or the other, which it isn't. |

---

## Quiz 2 — Building and Running Kernel Modules

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | Which prints "Hello, world" to the kernel log? Option 1: `printk(KERN_ALERT "Hello, world\n");` Option 2: `printf("Hello, world\n");` | **Option 1** | No libc in the kernel, so no printf. Use printk. |
| 2 | Purpose of the init function (module_init) | **Registers the module for use by the kernel** | It sets things up, registers, and **returns**. It does *not* stay running to handle requests. |
| 3 | Purpose of the exit function (module_exit) | **De-registers module functions with the kernel which were set up by the init function** | No automatic cleanup: you undo everything init did. (The kernel doesn't "automatically undo" parts, and exit doesn't wait for users; rmmod refuses if the module is in use.) |
| 4 | Why not `#include <stdio.h>`? | **libc is not available in the kernel module build environment** | |
| 5 | Why does execution transfer from user to kernel space? (select all) | **An application issues a system call; A hardware interrupt occurs** | Running as root does *not* put code in kernel space. |
| 6 | You need to write reentrant code in your driver… | **always** | Concurrency comes from multiple processes, preemption, interrupts, and SMP. "Reentrancy must be considered even when not sleeping." |
| 7 | Driver needs a buffer of several thousand bytes. How to allocate? | **Use dynamic memory allocation** | The kernel stack may be only 4 KB, shared by the whole call chain. Use kmalloc, never a large local or stack variable. |
| 8 | When use modprobe instead of insmod? | **When you want modules needed by a requested module to be loaded automatically** | modprobe handles dependencies. |
| 9 | "Invalid module format" from insmod. Suggestion? | **Make sure they're building against the correct kernel version** | The vermagic (kernel version, compiler, configuration) must match the running kernel. |
| 10 | When writing an init error handler you must (select all) | **Undo any registration activities performed before a failure** | Never force a reboot. **goto is widely used and encouraged** for this unwinding. |

---

## Quiz 3 — Character Device Drivers

**Q1. `ls -l /dev` with `1, 3` highlighted on the `null` line. What do the numbers show? (select all)**
✅ **The major number for /dev/null is 1** and ✅ **The minor number for /dev/null is 3**
(❌ Same driver as tty1: tty1 is major **4**. ❌ "3 null devices": a minor number is an *identifier*, not a count.)

**Q2. `4, 1` highlighted on the `tty1` line. (select all)**
✅ **tty1, ttys0 and ttyS1 most likely use the same driver** (all are major 4)
✅ **/dev/tty1 minor number is 1**
✅ **/dev/tty1 major number is 4**
(❌ "4 tty1 devices in use": not a count.)

**Q3. How should you allocate a major number for your device driver? (select all possible)**
✅ **Find a static assigned number which matches the device type (if your device is a common type)**
✅ **Use dynamic allocation to obtain a device number at run time**
(❌ You don't request a new static device type from kernel.org.)

**Q4. With dynamic majors, how do you find the number for your loaded driver?**
✅ **Use the value in /proc/devices for the loaded driver** (parse it with awk, then mknod)

**Q5. Best description of struct inode**
✅ **Represents files in any state**
(The *open* instance is `struct file`. fops maps driver functions to syscalls. Neither maps to libc's FILE.)

**Q6. Purpose of cdev_alloc**
✅ **Obtain a character device structure at runtime**
(`cdev_init` initializes a previously allocated cdev; `cdev_add` informs the kernel; `cdev_del` removes it.)

**Q7. Given `dev = container_of(inode->i_cdev, struct scull_dev, cdev); filp->private_data = dev;` (select all)**
✅ **filp->private_data will be set to the scull_dev structure containing the cdev member at address inode->i_cdev**
✅ **container_of() will calculate the start of the scull_dev structure in memory based on the location of the cdev structure**
(❌ private_data isn't the cdev's address; it's the address of the *enclosing* scull_dev.)

**Q8. Every open() system call results in an open file method invocation on the driver.**
✅ **True**. Each open creates a new `struct file` and calls the driver's open.

**Q9. Every close() system call results in a release method invocation.**
✅ **False**. After fork/dup, several closes can happen, but **only the last close** of that struct file triggers release (exactly one release per open).

**Q10. True statements about a `__user` pointer**
✅ **Access by kernel drivers should be performed through dedicated copy functions** (copy_to_user / copy_from_user)
✅ **It corresponds to user-space memory**
(❌ It is NOT safe to dereference directly: it may be invalid, paged out, or malicious.)

---

## Quiz 4 — Kernel Debugging Techniques

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | Challenges with kernel debugging (select all) | **difficult to execute under a debugger; bugs are often hard to reproduce; difficult to trace; bugs may crash the entire system** | ❌ "gdb can set breakpoints and modify the running kernel via /proc/kcore" is false: kcore is analysis-only. |
| 2 | Production kernels usually have debug features on by default | **False** | They're off for performance. |
| 3 | Debug features may have performance impacts to consider for production | **True** | |
| 4 | You can debug kernel code with the same printf() as user space | **False** | No libc. |
| 5 | Debug prints can be added with printk() | **True** | |
| 6 | Ways to view printk output (select all) | **dmesg; /var/log files via klogd/syslog; on the console** | ❌ `/proc/sys/kernel/printk` *controls* which levels reach the console; it doesn't show the messages. |
| 7 | With SCULL_DEBUG and `__KERNEL__` defined, where does `PDEBUG("test from driver")` go? | **To the printk location based on syslog configuration** | It expands to `printk(KERN_DEBUG "scull: " …)`. |
| 8 | With SCULL_DEBUG not defined? | **Nowhere** | It expands to nothing (`/* not debugging: nothing */`). |
| 9 | The first step in kernel debugging is usually a debugger | **False** | Printing is the most common first step. |
| 10 | GDB can analyze a running kernel via /proc/kcore | **True** | Read-only analysis. |
| 11 | gdb can halt, set breakpoints, or modify memory via /proc/kcore | **False** | That needs kgdb, not kcore. |

---

## Quiz 5 — Concurrency

*Code for Q1–Q2* (`dptr->data` used in a driver's write(), accessed by multiple processes):
```c
1 if (!dptr->data) {
2     dptr->data = kmalloc(100, GFP_KERNEL);
3     if (dptr->data) {
4         kfree(dptr->data);
5         dptr->data = NULL;
6     }
7 }
```

**Q1. True statements (select all)**
✅ **The processor may sleep at line 2.** `kmalloc(GFP_KERNEL)` can sleep.
✅ **The memory allocated at line 2 may never be released with kfree.** Process A allocates pA and is preempted before line 4. Process B (which also passed line 1) stores pB, overwriting pA. One kfree frees pB, and **pA is leaked**.
✅ **kfree at line 4 may be called with a null argument** ⚠️. A passes the line 3 check, then is preempted. B runs lines 2–5 and sets `data = NULL`. A resumes and calls `kfree(dptr->data)` = `kfree(NULL)`. (kfree(NULL) is harmless, but the scenario can happen.)
❌ No locking is needed.

**Q2. Appropriate locking**
✅ **Locking should be added around the entire block from line 1 through 7.** The check (line 1) and the modification must be atomic together. Use a **mutex**, since kmalloc may sleep, which rules out a spinlock.

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 3 | A driver typically allocates an object, gives the kernel a reference, then initializes it | **False** | "No object can be made available to the kernel until it can function properly." Initialize first. |
| 4 | References given to the kernel must be tracked, because drivers must honor requests as long as the object exists | **True** | |
| 5 | Processes may sleep while waiting for I/O to complete | **True** | Blocking I/O. |
| 6 | To obtain a semaphore a driver uses… | **The "down" functions to decrement and obtain the semaphore** | down = P. up releases it. |
| 7 | Interruptible semaphore versions should be used… | **In most cases** | Otherwise the process can't be interrupted (e.g., by SIGTERM). Exception: filesystem I/O. |
| 8 | A thread that takes a semaphore needs to call… | **up exactly one time to release it** | One down → exactly one up. |
| 9 | Consider a unique semaphore per device if devices don't share resources | **True** | Lock data (each device's own state), not code. More concurrency. |
| 10 | R/W semaphores allow one writer or an unlimited number of readers | **True** | |
| 11 | R/W semaphores are most useful when writes are more common than reads | **False** | Use them when writes are **rare** and brief. |
| 12 | Completions differ from semaphores because they're optimized for when the lock is available | **False** | Backwards: **semaphores** are optimized for the available case, **completions** for the *not-available* case. |

**Q13. Match the locking type to its best use**
- "Protect access to a resource, with actions that do **not** include sleeping" → **spinlock**
- "Wait for a potentially long-running task to end" → **completion**
- "Protect access to a resource, with actions that **require** sleep" → **mutex** (or semaphore)

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 14 | Spinlocks are only needed on multiprocessor machines | **False** | On a preemptible uniprocessor kernel, spinlocks still disable preemption (and irqsave disables interrupts), so the code is still required for correctness. |
| 15 | Accessing a shared resource and allocating memory while doing so: which primitive? | **mutex** | kmalloc may sleep, so you can't hold a spinlock. atomic doesn't protect a multi-step section. |
| 16 | Very brief access, no other kernel calls | **spinlock** | Short, non-sleeping, and the highest performance. |
| 17 | spin_lock_irqsave should be used… | **when the associated spinlock may be used in an interrupt** | irqsave is always safe; it's required when an ISR can take the same lock (otherwise same-CPU deadlock). |
| 18 | Spinlocks differ from semaphores because spinlocks let the processor sleep while waiting | **False** | Spinlocks **busy-wait**. Semaphore waiters sleep. |
| 19 | Best locking strategy: wait for issues to appear, then debug and add locks | **False** | Design locking in from the beginning. |

**Q20. Locking for `do_something_else()` (var2++), `do_something()` (var1++, then calls do_something_else), and `somekernelfunc()` (calls do_something), protecting `struct my_data` with a single lock inside the struct**
✅ **The lock should be acquired and released in somekernelfunc around the do_something call, and do_something and do_something_else should require callers to hold the lock.**
- Locking inside each function would make the nested call try to take the same lock again, a **self-deadlock**.
- Lock in the caller and **document that the callee assumes the lock is held**. One lock is enough, and `++` isn't atomic.

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 21 | A properly designed circular buffer can be used without locks with one reader thread and one writer thread | **True** | The LDD3 lock-free algorithm (2 threads, atomic indexes). |
| 22 | Operations on a single atomic value may need additional locking on some architectures | **False** ⚠️ | `atomic_t` operations are guaranteed atomic on **all** architectures. Extra locking is only needed when several values or steps must change together. |

---

## Quiz 6 — Advanced Char Driver Operations

| # | Question (abridged) | Answer | Why |
|---|---|---|---|
| 1 | ioctl lets you interact with your driver from user space, passing a command and an optional argument pointer | **True** | |
| 2 | ioctl is the mechanism most kernel developers recommend | **False** | It's "fallen out of favor": unstructured, hard to audit, poorly documented. |
| 3 | Purpose of an ioctl magic number | **Associate ioctl command arguments with specific device drivers** | It keeps the right command from reaching the wrong device. (Direction and size are separate fields.) |
| 4 | Magic number, sequence number, direction, and size are combined in `cmd` via `_IO`, `_IOR`, `_IOW`, `_IOWR` | **True** | |
| 5 | Allocate ioctl command numbers starting at 0, sequentially | **False** | Use the `_IO*` macros with a unique magic number. |
| 6 | Steps for an argument pointer with direction **_IOC_READ** | **Check for safe write access to user memory and copy from kernel memory to user memory** | READ = the user reads, so data goes kernel → user, and the kernel *writes* user memory. |
| 7 | Steps for **_IOC_WRITE** | **Check for safe read access to user memory and copy from user memory to kernel memory** | WRITE = the user writes, so data goes user → kernel, and the kernel *reads* user memory. |
| 8 | The best method for controlling devices that don't transfer data is ioctl | **False** ⚠️ | LDD3 suggests **writing control sequences** to the device as the alternative, "especially useful for devices which don't transfer data". |
| 9 | Rules for sleeping in a driver (select all) | **Don't make assumptions about the state of the system after waking up; Ensure some other process or kernel event will wake you up** | ❌ Your own (sleeping) thread can't wake you. ❌ Atomic context means you *can't* sleep. |
| 10 | Blocking read() should block if… (select all) | **No data is available to return to the user space program** | If *some* data is available (even less than requested), return it now. O_NONBLOCK means return -EAGAIN instead. "No buffer space" is a write condition. |
| 11 | Blocking write() should block if… (select all) | **No space is available in the device driver buffer to store the data** | If *some* space exists ("not enough"), accept at least 1 byte and return a partial count. O_NONBLOCK means -EAGAIN. "No data to return" is a read condition. |
| 12 | Input/output buffering matters most when device transfers are much slower than memory transfers | **True** | |
| 13 | True statements about sleep rules (select all) | **Do not sleep when disabling interrupts under any circumstances; You should only sleep while holding a semaphore or mutex after careful evaluation; Do not sleep when holding a spinlock under any circumstances** | Spinlock or IRQs-off means never sleep. Semaphores and mutexes: allowed, but think carefully (risk of deadlock or blocking the wake-up). |
| 14 | schedule() yields the CPU and returns only when your driver's condition occurs | **False** | schedule() just yields. **Your wait loop (wait_event) re-checks the condition** after each wake-up. |
| 15 | A driver's poll function reports whether reads or writes would block | **True** | It returns a mask (POLLIN, POLLOUT…) after poll_wait on its queues. |
| 16 | fsync is only needed for drivers that support O_NONBLOCK | **False** | fsync lets user space ensure data reached the device. **write never waits for device transmission**, blocking or not. |
| 17 | To support seeking, a driver must implement llseek | **False** ⚠️ | If llseek is missing, the **default implementation updates `filp->f_pos`**, and seeking works as long as read/write honor `*f_pos`. Implement llseek only for custom behavior (or call `nonseekable_open` to forbid seeking). |
| 18 | Ways to handle open() for a device that isn't currently available (select all) | **Return -EBUSY until access is available; Blocking until access is available** | -EPERM means "operation not permitted", which is the wrong error. |
