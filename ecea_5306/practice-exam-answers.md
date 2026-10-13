# ECEA 5306 — Practice Final Exam: Answer Key

## Part A — True / False

| # | Ans | Explanation | Review |
|---|---|---|---|
| 1 | **T** | The course labels Linux "monolithic modular". | §1.1 |
| 2 | **F** | Network interfaces use names like eth0 with **no /dev node**; packet I/O doesn't map to read/write. | §1.2 |
| 3 | **F** | There's **no automatic cleanup** in the kernel: leaked memory stays allocated until reboot. | §1.3 |
| 4 | **T** | That's what EXPORT_SYMBOL_GPL does. | §1.5 |
| 5 | **T** | `__init` code can be dropped after loading. | §1.5 |
| 6 | **T** | First pass: KERNELRELEASE unset, so it calls the kernel Makefile. Second pass: obj-m is understood. | §1.4 |
| 7 | **F** | Yocto is **MIT**-licensed (permissive). Buildroot is GPLv2. | §1.6 |
| 8 | **F** | `./script` runs in a subshell, so its variables are lost. You must `source` it. | §1.6 |
| 9 | **T** | MACHINE in local.conf (Buildroot uses a defconfig). | §1.6 |
| 10 | **F** | It returns the number of bytes **not** copied (0 = success). | §2.3 |
| 11 | **F** | `struct file` is the kernel's open-file object, unrelated to libc FILE*. | §2.2 |
| 12 | **T** | printk writes to a circular buffer, so it's safe anywhere. | §2.5 |
| 13 | **T** | "Loading out-of-tree module taints kernel." | §1.3 |
| 14 | **T** | complete_all releases current **and future** waiters. | §3.3 |
| 15 | **F** | trylock never sleeps. | §3.2 |
| 16 | **T** | That's why kmalloc can't be called while holding a spinlock. | §3.2 |
| 17 | **T** | Mutexes allow sleeping (copy_from_user may fault and sleep). | §3.2 |
| 18 | **T** | wake_up wakes all waiters; wake_up_interruptible wakes only interruptible ones. | §4.2 |
| 19 | **F** | 0 means nothing was written, so user space retries, and the app appears to hang. | §3.5 |
| 20 | **T** | It's for data *flows* (serial port, keyboard) where seeking makes no sense. | §4.3 |

## Part B — Multiple Choice

| # | Ans | Explanation | Review |
|---|---|---|---|
| 21 | **b** | Mechanism = capabilities; policy = how they're used (leave that to user space). | §1.1 |
| 22 | **d** | An FTP server is a user-space application. | §1.1 |
| 23 | **b** | vermagic mismatch, so rebuild against the target kernel. | §1.5 |
| 24 | **c** | MKDEV builds it; MAJOR/MINOR extract the parts. | §2.1 |
| 25 | **b** | /proc/devices lists majors and driver names. | §2.1 |
| 26 | **c** | cdev_add makes the device live. | §2.2 |
| 27 | **b** | container_of on inode->i_cdev. | §2.2 |
| 28 | **b** | 4 = WARNING and more severe. | §2.5 |
| 29 | **c** | Format is offset/length: 0x4 into a function 0x10 bytes long. | §2.5 |
| 30 | **c** | You can't sleep in an ISR, so use a spinlock (irqsave on the non-ISR side). | §3.4 |
| 31 | **b** | -ERESTARTSYS when restarting is correct; -EINTR when changes can't be undone. | §3.3 |
| 32 | **b** | Opposite lock order means deadlock. | §3.4 |
| 33 | **c** | Semaphores before spinlocks (semaphores may sleep). Release in *reverse* order; take driver locks first. | §3.4 |
| 34 | **c** | _IOW = user writes into the kernel. | §4.1 |
| 35 | **b** | unlocked_ioctl (compat_ioctl alone isn't enough). | §4.1 |
| 36 | **b** | Wake writers when a read frees space; wake readers when a write adds data. | §4.2 |
| 37 | **c** | POLLIN \| POLLRDNORM for readable. | §4.3 |
| 38 | **b** | "Grass\n" covers offsets 0–5, so "Sentosa" starts at 6. | §4.4 |
| 39 | **b** | devtool modify (then devtool reset when done). | §1.7 |
| 40 | **c** | update-rc.d class + INITSCRIPT_NAME. | §1.7 |

## Part C — Select All That Apply

| # | Answers | Explanation | Review |
|---|---|---|---|
| 41 | **a, b, e** | Never trust user pointers. Drivers generally don't implement security policy. | §1.2 |
| 42 | **a, b, c** | No glibc malloc. The whole call chain shares the stack, so even "<4 KB" locals are risky. | §1.3 |
| 43 | **a, b, c** | `__user` points to *user* memory. | §2.3 |
| 44 | **a, b, c, d, e** | All are course points. | §1.6, §1.8 |
| 45 | **a, b, c, d, e** | All five are listed in LDD3 Ch. 5. | §3.1 |
| 46 | **a, b, c, e** | kmalloc(GFP_KERNEL) may sleep, so it can't be called under a spinlock. | §3.4 |
| 47 | **a, b, d** | Returning the bytes copied (and 0 at EOF) is correct behavior. | §3.5 |
| 48 | **a, b, c** | Avoid 'T' (the kernel decodes those). Don't number sequentially from 0. | §4.1 |
| 49 | **a, b, d** | Another process or event must wake you. Never sleep holding a spinlock. | §4.2 |
| 50 | **a, b, d** | An open fd keeps the module in use. module_exit is required to unload. | §3.5 |

**Scoring:** 54–60 exam-ready · 45–53 review the missed sections · below 45 re-study the modules.
