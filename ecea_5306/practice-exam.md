# ECEA 5306 — Practice Final Exam
### Linux Kernel Programming and Introduction to Yocto Project

**Format:** like the real final: True/False, multiple choice, and select-all-that-apply.
**Suggested time:** 2 hours, closed-book except your one-page notes sheet.
**Answers:** in `practice-exam-answers.md`.

These are new questions written from the lecture material, not copies of the graded quizzes. They cover all four modules plus the Yocto material.

---

### Part A — True / False (1 point each)

1. Linux is classified as a monolithic (modular) kernel. **T / F**

2. A network interface such as `eth0` appears as a node in `/dev` like character and block devices do. **T / F**

3. Memory allocated by a kernel module and never freed is automatically reclaimed when the module is unloaded with rmmod. **T / F**

4. `EXPORT_SYMBOL_GPL()` makes a symbol available only to modules that declare a GPL-compatible license. **T / F**

5. The `__init` marker allows the kernel to discard the init function's memory after the module has loaded. **T / F**

6. When building an out-of-tree module with recursive make, the module's Makefile is read twice. **T / F**

7. The Yocto Project build system is licensed under GPLv2, so you must share your layer changes. **T / F**

8. Running `./oe-init-build-env` (instead of `source oe-init-build-env`) correctly sets up the bitbake environment in your current shell. **T / F**

9. In Yocto, the target hardware is typically selected with the `MACHINE` variable in `conf/local.conf`. **T / F**

10. `copy_to_user()` returns the number of bytes that were successfully copied. **T / F**

11. The `struct file` passed to a driver's read method is the same object as the C library `FILE *`. **T / F**

12. printk() is safe to call from an interrupt handler. **T / F**

13. Loading a module that is out-of-tree may mark the kernel as "tainted." **T / F**

14. A completion's `complete_all()` call will also release threads that begin waiting after it was called. **T / F**

15. `down_trylock()` may put the calling process to sleep. **T / F**

16. A kmalloc call using `GFP_KERNEL` may sleep. **T / F**

17. Holding a mutex while calling `copy_from_user()` is acceptable because the caller may sleep. **T / F**

18. `wake_up_interruptible()` wakes only processes sleeping in an interruptible state. **T / F**

19. If a driver's write method returns 0, a typical user-space program will consider the write complete and stop. **T / F**

20. Calling `nonseekable_open()` is appropriate for a driver representing a stream of data such as a serial port. **T / F**

---

### Part B — Multiple Choice (1 point each)

21. Which statement best captures "mechanism, not policy"?
   - a) Drivers should implement authentication for each user
   - b) Drivers should expose hardware capabilities and leave decisions about how they're used to user space
   - c) Drivers should hide all hardware details behind simplified APIs
   - d) Drivers should only support blocking I/O

22. Which is NOT a kernel role according to LDD3?
   - a) Scheduling processes
   - b) Building process virtual address spaces
   - c) Routing network packets
   - d) Running an FTP server

23. Your module loads fine on your laptop but fails on the target with "Invalid module format". The most likely cause is:
   - a) A missing MODULE_AUTHOR line
   - b) The module was built against a different kernel (vermagic mismatch)
   - c) You used insmod instead of modprobe
   - d) The module has a memory leak

24. Which macro builds a `dev_t` from a major and minor number?
   - a) `MAJOR()`
   - b) `MINOR()`
   - c) `MKDEV()`
   - d) `register_chrdev()`

25. A driver has registered with a dynamically allocated major number. Where do load scripts find it?
   - a) `/proc/modules`
   - b) `/proc/devices`
   - c) `/sys/kernel/debug`
   - d) `/etc/devices`

26. Which function informs the kernel that an initialized `struct cdev` is ready to receive operations?
   - a) `cdev_alloc()`
   - b) `cdev_init()`
   - c) `cdev_add()`
   - d) `cdev_del()`

27. Inside an open method, what is the conventional way to locate your device structure that embeds a `struct cdev`?
   - a) `filp->f_op`
   - b) `container_of(inode->i_cdev, struct my_dev, cdev)`
   - c) `MINOR(inode->i_rdev)`
   - d) A global variable

28. Which `/proc/sys/kernel/printk` "current" value causes WARNING and more severe messages (levels 0–4) to print to the console?
   - a) 1
   - b) 4
   - c) 7
   - d) 0

29. An oops message reports `PC is at faulty_write+0x4/0x10 [faulty]`. What does `0x10` represent?
   - a) The faulting address
   - b) The offset into the function where the fault occurred
   - c) The total length of the function `faulty_write`
   - d) The line number in the source file

30. Which locking primitive is appropriate in an interrupt handler?
   - a) mutex
   - b) semaphore with down_interruptible
   - c) spinlock
   - d) completion's wait_for_completion

31. Which return value should an interruptible lock failure typically produce when the operation can be safely restarted?
   - a) -EINTR
   - b) -ERESTARTSYS
   - c) -EAGAIN
   - d) -EBUSY

32. Two threads each acquire two mutexes but in opposite orders. This is a classic recipe for:
   - a) Priority inversion
   - b) Deadlock
   - c) A memory leak
   - d) A buffer overrun

33. Which rule of thumb from LDD3 about multiple locks is correct?
   - a) Acquire kernel-wide locks before your driver's locks
   - b) Acquire spinlocks before semaphores
   - c) Acquire semaphores before spinlocks
   - d) Release locks in the same order you acquired them

34. Which ioctl macro defines a command that transfers data from user space INTO the driver?
   - a) `_IO`
   - b) `_IOR`
   - c) `_IOW`
   - d) None, ioctl cannot transfer data

35. Which fops member must you implement for modern ioctl support?
   - a) `.ioctl`
   - b) `.unlocked_ioctl`
   - c) `.compat_ioctl` only
   - d) `.llseek`

36. In a scullpipe-style driver, when should the writer's wait queue be woken?
   - a) When new data is written
   - b) When a read frees space in the buffer
   - c) When the device is opened
   - d) Never; writers poll

37. What should a driver's poll method return when data is available to read?
   - a) `POLLHUP`
   - b) `POLLOUT | POLLWRNORM`
   - c) `POLLIN | POLLRDNORM`
   - d) 0

38. In Assignment 9, writes "Grass\n", "Sentosa\n", "Singapore\n" are stored. What file offset corresponds to the 'S' at the start of "Sentosa"?
   - a) 5
   - b) 6
   - c) 7
   - d) 14

39. Which Yocto tool lets you edit a recipe's source in a workspace (similar to Buildroot's OVERRIDE_SRCDIR)?
   - a) `bitbake -c cleanall`
   - b) `devtool modify`
   - c) `bitbake-layers add-layer`
   - d) `runqemu`

40. Which Yocto class installs SysV init scripts for a package?
   - a) `inherit core-image`
   - b) `inherit module`
   - c) `inherit update-rc.d`
   - d) `inherit systemd`

---

### Part C — Select All That Apply (2 points each)

41. Which are security best practices for driver authors from LDD3 Chapter 1? (select all)
   - a) Zero kernel memory before passing it to user space
   - b) Length-check user data before copying it
   - c) Trust user-supplied pointers because the system call validated them
   - d) Implement per-user authentication in every driver
   - e) Restrict module loading to authorized users

42. Which statements about the kernel stack and memory in modules are true? (select all)
   - a) The kernel stack may be as small as 4 KB
   - b) Large buffers should be allocated with kmalloc
   - c) Floating point is generally not supported in kernel code
   - d) malloc() and free() are available through glibc
   - e) Large local arrays are fine as long as they are under 4 KB

43. Which are reasons why `buf` in a driver's read method (a `__user` pointer) must not be dereferenced directly? (select all)
   - a) It may not be valid or accessible from kernel space on some architectures
   - b) The page may not currently be present in RAM
   - c) It comes from an untrusted program
   - d) It always points to kernel memory

44. Which are TRUE about Yocto compared with Buildroot per the course? (select all)
   - a) Yocto typically has a longer clean build time
   - b) Yocto generally has better incremental build support
   - c) Buildroot has the lower initial learning curve
   - d) Yocto is a better choice when a BSP is only available for Yocto
   - e) Yocto builds most tools from source to support binary reproducibility

45. Which of the following are sources of concurrency that a driver must handle? (select all)
   - a) Multiple user-space processes using the driver
   - b) SMP systems executing the driver on several CPUs
   - c) Kernel preemption
   - d) Asynchronous hardware interrupts
   - e) The device being removed while in use

46. Which are true about spinlocks? (select all)
   - a) Waiting CPUs execute a tight loop
   - b) They must be held for as short a time as possible
   - c) Code holding a spinlock must not sleep
   - d) They are a good choice when you need to call kmalloc(GFP_KERNEL) while holding the lock
   - e) spin_lock_irqsave is safe in both interrupt and non-interrupt context

47. A driver's read method is being called repeatedly forever by `cat`. Which could be the cause? (select all)
   - a) The method never returns 0 at end of data
   - b) The method does not update `*f_pos`
   - c) The method returns the number of bytes copied
   - d) The method always returns the full count even when no data remains

48. Which correctly describe ioctl command numbers? (select all)
   - a) They encode a magic number unique to the driver
   - b) They encode the transfer direction from the user's perspective
   - c) They encode the size of the argument
   - d) Magic number 'T' is recommended for custom drivers
   - e) They should simply be 0, 1, 2, … for each driver

49. Which are correct statements about sleeping in the kernel? (select all)
   - a) A sleeping process is removed from the scheduler run queue
   - b) The condition in `wait_event_interruptible(q, cond)` is re-evaluated each time the task wakes
   - c) A sleeping process can wake itself up from the same thread
   - d) You should re-check state after waking because anything may have changed
   - e) It is safe to sleep while holding a spinlock if the sleep is short

50. Your aesdchar driver's `rmmod` fails with "resource temporarily unavailable" during testing. Which actions address this? (select all)
   - a) Have aesdsocket open /dev/aesdchar only when it accesses it, and close it afterward
   - b) Make sure each open() is paired with close() in user space
   - c) Remove the module_exit function
   - d) Ensure no process holds the device open when unloading

---
*End of practice exam: 50 questions. Scoring: Part A 20 + Part B 20 + Part C 20 = 60 points.*
