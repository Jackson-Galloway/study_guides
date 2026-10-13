# ECEA 5307 — Linux Embedded System Topics and Projects
## Final Exam Study Guide

*Built from the course lecture transcripts, slides, readings, and practice quizzes (instructor: Dan Walkes, CU Boulder MS-ECE). This is the capstone course after 5305 and 5306.*

**Final exam format** (from "About the Final Exam"): proctored via ProctorU, **45 minutes**, **1 attempt**, "similar in format to the homework quiz assignments". Worth **5%** of the grade; the final project sprints are 75%, peer review 10%, and the demo video 10%.
**Allowed:** one notes sheet (front and back, ≥11pt if typed, your own), or an electronic copy, or one URL to your notes. **Not allowed:** textbooks, lecture slides/content, AI.

> 45 minutes is short, so the exam is probably ~15–25 quiz-style questions. The course quizzes are ungraded practice, which makes them the best preview of the final. Work through `quiz-answer-key.md` and the practice exam.

---

## Table of Contents
1. [Module 1 — Final Project, Agile Scrum, Building for Hardware](#module-1)
2. [Module 2 — Linux Device Update, MQTT/IoT, Read-Only Rootfs, Containers](#module-2)
3. [Module 3 — Git Workflows, C++ Templates/STL, C Dependency Generation](#module-3)
4. [Module 4 — Embedded Software Testing, CI/CD, TDD/BDD](#module-4)
5. [Quick-Reference Tables](#quick-reference)

---

<a id="module-1"></a>
## Module 1 — Final Project, Agile Scrum, Building for Hardware

### 1.1 Agile Overview
- **The Agile Manifesto's values**:
  1. **Individuals and interactions** over processes and tools (keep it lightweight)
  2. **Working software** over comprehensive documentation
  3. **Customer collaboration** over contract negotiation (requirements change once people see things implemented)
  4. **Responding to change** over following a plan
  - So Agile does **NOT** call for a detailed, completely thought-out plan at the start.
- Agile isn't right for every project. It's covered because it's popular and comes up in interviews.
- **Scrum** is a **process framework for addressing problems**, run by a small, flexible team.
  - **Product Owner**: **manages the product backlog** (what's most important, what goes in the next sprint) and **provides the voice of the end user**.
  - **Development Team**: **delivers "Done"**. Usually **3–9 members**: more than 9 gets inflexible (split into separate teams), fewer than 3 is more overhead than you need.
  - **Scrum Master**: **enforces the rules and coaches** the team on the process (keeps stand-up reports short).
- **Pigs vs chickens** ("ham and eggs": the pig is *committed*, the chicken only *involved"):
  - **Product owner, dev team, and Scrum master are pigs**: their jobs ride on the project.
  - **Management is often a chicken**: involved, but spread across many projects.
- **Course mapping**: product owner = you + SAs + peer reviewers; dev team = you; Scrum master = the instructor or SAs.
- **Product backlog**:
  - An **ordered list of everything known to be needed in the product**: features, functions, requirements, enhancements, fixes, each with a description, order, estimate (time or cost), and value.
  - The product owner owns it. **It's never complete** while the product is alive; you always keep 2–3 sprints of items groomed ahead.
- **Sprint**: a time-box, usually **1–2 weeks** (not a month).
  - The **sprint backlog** is the subset of items for this increment. Tasks are broken down small enough to **show measurable progress in each daily scrum**.
- **Definition of Done (DoD)**: everyone (developer *and* product owner) must agree what "done" means for each item. It provides transparency and prevents miscommunication.
- **Every sprint increment should be *usable***: completing one item **must not break functionality from a previous sprint**. Ship a usable subset, not broken partial work.
- **Daily stand-up / Scrum questions**:
  1. **What did I do since the last report** (yesterday) toward the sprint goal?
  2. **What will I do before the next report** (today)?
  3. **What, if anything, is blocking me?** Blockers could be incomplete work, a bug, a late part order, or another team's hardware. Solve them as a team, or reorganize the work.

### 1.2 Final Project Structure
- **Requirements**:
  - **Real hardware**, not QEMU.
  - **Buildroot or Yocto** builds the image.
  - Some relationship to the course content.
  - Solo or in **groups of up to 3**, **graded individually**.
  - Total scope ≈ **3 assignments (30–45 hrs/person)**: **~10–15 hrs per sprint**.
- **Schedule**: Proposal → **Sprint 1 → Sprint 2 → Sprint 3** → **Demo video (≤5 min per student) and outline**, which covers challenges and lessons learned.
- **Project Overview wiki page**: goals and motivation with **at least one diagram** (identifying the hardware); the build system; the hardware platform (with support links if unsupported); open-source projects used (or "none"); previous assignment and course content reused; new content; source organization (which repos); team roles; and a link to the **Schedule** (a GitHub Projects page).
- **Issues**:
  - **Track work with GitHub issues**, each with a **complete DoD** understandable by an outsider; link blockers.
  - **At least one closed issue per member per sprint**, with a comment that the **DoD is verified** and how (screenshots help).
  - **If partially done**: redefine the issue to what was done, close it, and move the rest to a new linked issue.
  - **Use "Edit"** to keep the top description current (comments are for transient status).
  - Report partial status as "**As of commit <hash> …**".
  - **Put issues in the repo where the code lives**, so commits can reference them ("Issue #204"; **"Closes #204"/"Fixes" auto-closes**, so omit those words to keep the issue open).
  - The DoD test: "**Could a peer reviewer verify this from the code plus the DoD?**"
- **Sprint scoping**: don't plan a complete final component in one sprint. Split it up (e.g., limited features first, or run on the PC/VM before hardware). **Deliver *something* building each sprint**, never just empty `.c`/`.h` files. **Plan major components up front and prove the riskiest parts early.**
- **Sprint 1 suggestion**: Buildroot/Yocto build complete for the hardware, the board booting and communicating, and some software work done.
- **Retargeting Buildroot for hardware**:
  1. Edit **`shared.sh`** to use the hardware defconfig (`AESD_DEFAULT_DEFCONFIG`).
  2. **`./clean.sh`** (**clean builds when switching targets**, or use separate output directories).
  3. `./build.sh`, then `./save_config.sh` (to `AESD_MODIFIED_DEFCONFIG`).
  4. **Re-add your customizations** in menuconfig: dropbear, root password, assignment/ldd packages, rootfs overlay. Then save and build again.
  - **Flash the SD card** with `dd` (Linux) or Rufus (Windows).
- **Faster iteration than re-flashing**: test in QEMU, or **`scp` changed executables from `output/build` to the target**. Rebuild and retest the full image when done.
- **Supported boards**: **Raspberry Pi** (4B, 3B+, Zero W; Buildroot and Yocto) and **BeagleBone Black** (Buildroot). See the Supported-Hardware-Platforms wiki and past student projects.

### 1.3 Buildroot Additional Hints
- **Hidden dependencies**: if a package (e.g., **bash**) doesn't appear in menuconfig, search with `/` and read its **"Depends on"** line. Bash needs **`BR2_PACKAGE_BUSYBOX_SHOW_OTHERS`**.
- **Iterating on source**:
  - The **brute-force way** is to push to git and update the `FOO_VERSION` hash every time: slow, with lots of commits. **You do NOT have to do that.**
  - **Use `FOO_OVERRIDE_SRCDIR`**, run **from the buildroot directory**: `make AESD_ASSIGNMENTS_OVERRIDE_SRCDIR=~/aesd-assignments aesd-assignments-rebuild all`. Use the **ALL-CAPS package name** before `_OVERRIDE_SRCDIR`.
  - It **rsyncs** your directory into `output/build/<pkg>-custom`.
  - **`<pkg>-dirclean`** starts the package from an empty build directory.

### 1.4 Bootloaders and Device Trees (MELP Ch. 3)
- **Bootloader job**: **initialize the system at a basic level**, doing the **minimum needed to load the kernel**.
  - It starts with **one CPU core and on-chip memory**.
  - It **bootstraps** in phases: **set up DRAM, set up storage, load the kernel into RAM**.
  - *A bootloader does NOT run web servers or control devices through kernel drivers.*
- **Device trees**:
  - **Before kernel 3.8 (Feb 2013)**, platform info was **discovered at runtime or hard-coded into the kernel**. That suited PCs; embedded and ARM boards have known hardware.
  - Now replaced by **device trees** (especially on ARM): the **system as a hierarchy (tree) of components**, with properties as **`name = "value"`** pairs (memory, CPUs, interrupt controller, devices).
  - **Compile** with **`dtc`** (Device Tree Compiler): `.dts` source → **`.dtb`** (**Device Tree Binary/Blob**, a.k.a. **Flattened Device Tree, FDT**). `.dtsi` = device tree source include.
  - **dtc also works in reverse, which is useful for *examining* device trees**: `dtc -I fs -O dts /sys/firmware/devicetree/base` (input = the /sys filesystem tree, output = human-readable source). For example, checking that SPI went from "disabled" to "okay".
- **Simple bootloader**: the **reset vector boots directly to NOR flash** (linearly addressable), which sets up RAM, possibly copies code, and runs from RAM.
- **Why not always simple? NAND flash isn't linearly addressable** and needs **ECC and bad-block recovery**. General hardware complexity (multiple cores, DRAM setup) means more work before Linux can start. Other reasons: **the image may not fit in memory addressable by the initial boot code**.
- **Complex multi-phase boot**:
  1. **ROM code** (on the SoC): the reset vector boots into ROM, which loads the **SPL (Secondary Program Loader)** into a **small on-chip SRAM**, from a special non-ECC NAND area or **over USB** (in production).
  2. **SPL**: **initializes DRAM**, then loads the **TPL (Tertiary Program Loader)** from NAND (minimal ECC) or USB.
  3. **TPL** (a **full bootloader such as U-Boot**): full NAND read/write support. It reads and decompresses the **FDT, kernel, and initramfs** into DRAM and **jumps to the kernel**. (This is the point where QEMU's `-kernel` starts.)
  4. **Kernel boot**: the kernel initializes DRAM; **initramfs finishes hardware init and sets up the rootfs**, then starts **init**.
- **UEFI (Unified Extensible Firmware Interface)**: **the BIOS replacement**, common on **x86/x86_64** (and some ARM). Uses the 3-phase approach. **Recent x86 designs use UEFI.**
- **Choosing a bootloader**: does it **support your platform**? Is it **provided with your BSP** (often the best starting point)? Is **source available**? Can you **get support**? (Options include U-Boot, GRUB 2, Barebox, RedBoot, CFE…)
- **U-Boot**:
  - Started as an **open-source bootloader for PowerPC, later ported to ARM**; **very common on ARM**.
  - Uses **Kconfig/defconfig** (`make <board>_defconfig`). Builds `u-boot.bin`/`.img`, plus possibly an SPL named **MLO**.
  - **Installing it** usually needs **JTAG** or hardware-specific steps (e.g., load into RAM over JTAG, then write flash).
  - **Using it**: a **serial terminal** prompt (`help`). **Environment variables** (initially from platform header files) pass information and script commands: **`setenv`**, **`saveenv`**, load images from flash or **TFTP**, and **`bootm <kernel addr> <ramdisk addr> <dtb addr>`**. **`bootcmd`** is the variable that runs at power-up.
- **Raspberry Pi**: device-tree customization goes in **`config.txt`** on the boot partition (it acts like a BIOS).
  - Example: `dtparam=spi=on`, `dtparam=i2c_arm=on`, audio.
  - Edit on a running Pi: mount `/dev/mmcblk0p1` and edit `config.txt`. The **kernel command line is in `cmdline.txt`**.
- **BeagleBone Black**: enable UART1 etc. by **patching the `.dtsi`** (e.g., `am335x-bone-common.dtsi`) with pinmux definitions plus `status = "okay"`. Find the kernel source via Buildroot's config, and use the `Documentation/devicetree/bindings` docs.

### 1.5 Linux Networking Overview
- **`ifconfig` was the traditional tool; `ip` is the newer replacement.** So "ip is the traditional way" is **false**.
- **`ip addr`** lists the interfaces:
  - **`lo`**: **loopback** (127.0.0.1 / localhost; what sockettest uses natively).
  - **`eth0`**: wired Ethernet (**`enp…`** names on systemd systems).
  - **`sit0`**: IPv6-over-IPv4 tunnel.
  - **`wlanX`**: wireless.
- **Reading the output**:
  - **`UP` means the link is up**, which **does not mean you're connected to the Internet**.
  - **`link/ether`** = the **MAC address** (a globally unique hardware ID).
  - **`inet`** = the IPv4 address/mask; **`inet6`** = IPv6.
- **How the address gets assigned**: Buildroot and Yocto default to **DHCP** on eth0 (most routers support it), configured in **`/etc/network/interfaces`** (auto-generated).
- **More complex configurations**: replace that file with a **rootfs overlay (Buildroot)** or **`ROOTFS_POSTPROCESS_COMMAND` (Yocto)**.

> **Module 1 Cheat-Sheet Candidates:** 4 Agile values; product owner = backlog + voice of the user; Scrum master = rules/coach; dev team 3–9; backlog ordered and never complete; sprint 1–2 weeks; DoD; increments usable (don't break prior sprints); stand-up 3 questions; OVERRIDE_SRCDIR from the buildroot directory; bootloader minimum (DRAM, storage, load kernel); device tree (pre-3.8 hard-coded), dtc, .dtb/FDT, dtc -I fs -O dts; NAND needs ECC; ROM → SPL (SRAM) → TPL (U-Boot, DRAM) → kernel + initramfs; UEFI = BIOS replacement on x86; U-Boot setenv/saveenv/bootm/bootcmd; RPi config.txt; ip vs ifconfig; lo = loopback; UP = link up only; DHCP + /etc/network/interfaces → overlay.

---

<a id="module-2"></a>
## Module 2 — Linux Device Update, MQTT/IoT, Read-Only Rootfs, Containers

### 2.1 Device Update Design Goals (Part 1)
- **OTA ("Over The Air")** = updating software on an embedded device **after shipment**. The term is used even for wired updates.
- **The scenario**: a development environment produces a **device update file** (bootloader, rootfs components…), which goes through a **cloud server** and possibly a wireless router to the device.
- **Trade-off**: implementation effort vs ease of use.
  - **Turnkey platforms** (e.g., **Android, Particle**) cost more but have update built in.
  - **Wi-Fi, Bluetooth, or a phone app** make updates easier for users.
  - **Serial or USB cables** are cheap but not sustainable for field updates.
- **Three design goals**:
  1. **Security**: **device update can be a backdoor**. **Prevent unauthorized code replacement.** **Obfuscation ≠ cryptography**: don't rely on "hard to figure out", because motivated attackers are clever. Use **cryptographic** security.
  2. **Reliability**: **recover from power loss or connection outages mid-update**, and never "brick" the device.
  3. **Ease of use, for end users *and* developers**: developers update many times a day (30 s vs 5 min adds up). Needing a serial cable should set off alarm bells.
- **Great update solutions are invisible, valued for what they *don't* do**: they don't expose users or the company to hackers, don't brick devices, and don't burden customers or developers. It's the **football long snapper** analogy: nobody notices until it fails.

### 2.2 Cryptographic Signing (Part 2)
- **Keys are huge numbers** (e.g., **256 bits ≈ 77 decimal digits**), roughly enough to number every atom in the universe.
  - **Brute force is impossible**: the fastest supercomputer would need **multiple universe lifetimes**, far more than a year.
  - A **random number generator + key generation algorithm (RSA or ECC)** produce a **public/private key pair**.
- **Signature generation**: a **fixed-length input number + the *private* key → signature/encryption algorithm (RSA/ECC) → signature** (another large number).
- **Signature validation**: **signature + *public* key → the same algorithm → the original input number**.
- **Key properties**:
  - **Knowing the input number, the signature, *and* the public key reveals nothing about the private key.** You could publish all three, and a hacker still couldn't recreate the private key.
  - **Anyone with the private key can create signatures that validate with the public key**, so **guard it**.
- **Firmware is any length, but the algorithm needs a fixed-size input** → run the image through a **hash function (SHA)** to get a fixed-length digest. *Don't truncate the image or use only some of its bytes.*
  - **A tiny change in the input produces a completely different hash**, so **any small unauthorized change makes signature validation fail**.
- **Where the keys live**: the **private key never leaves the development environment**. The **public key is stored on every device** (it's fine if attackers read it).
- **Signing**: device update file + private key → **hash + public-key crypto → signature**; the **update package** = update file + signature.
- **Verification on the device**:
  - signature + **public key** → RSA/ECC → **signed hash**
  - **SHA of the received update file → computed hash**
  - **Accept only if signed hash == computed hash.**
  - An attacker who modifies the file on the cloud server can't produce a matching signature without the private key.
- **Quantum computing**: RSA and ECC assume today's hardware can't factor huge numbers. **Quantum computers will change that** (it needs thousands of entangled qubits; ~20 had been achieved by 2019; maybe 5–15 years away for billion-dollar machines).
  - So **today's asymmetric schemes are NOT quantum-resistant**. **NIST is working on post-quantum standards (2022–2024).**
  - **Keep up with standards and use existing solutions; don't roll your own.**

### 2.3 Update Strategies and Frameworks (Part 3)
- **A/B (dual redundant)**:
  - The bootloader chooses between **rootfs partition A and B**. The update is written to the **inactive** partition, and an interruption doesn't matter because the active partition is untouched. On success, tell the bootloader to boot the other partition next time.
  - **If boot fails, the bootloader falls back** to the previous partition, based on a "boot succeeded" state set by the image.
  - **Configuration data (Wi-Fi credentials…) must live in a separate data partition**, since each rootfs is replaced wholesale.
- **Single copy**:
  - One rootfs plus an **initrd or dedicated update/recovery partition** with minimal logic to **detect whether the rootfs is bootable** and **perform updates in the non-bootable case**. That recovery path may be limited (e.g., Bluetooth only).
  - It usually still has a data partition.
- **Application update**:
  - Update individual applications with **packages (.deb/.rpm) via apt or yum**: binaries, start scripts, config files.
  - **Yocto builds these packages** as part of its image build. This is **how standard distributions (Debian, Ubuntu) update**.
- **Trade-offs**:
  - **A/B benefits**: **full rollback to a previous known-good release (even automatically)**; no partial or mixed application states; a **clean separation of config data and image**, which makes **factory reset easy**.
  - **A/B drawbacks**: **2× rootfs storage**; the **full (compressed) image every update** (delta schemes are possible but harder), which matters on cellular/LTE.
  - **Application update benefits**: smaller updates, no extra storage.
  - **Application update drawbacks**: **rollback is complicated**; possible **inconsistent states after a power cycle** (e.g., 25 of 50 packages updated).

| Criteria | A/B Dual Redundant | Single Copy | Application Update |
|---|---|---|---|
| Storage | **Most** (2× root partition) | 1× root + upgrade components | No/minimal extra |
| Redundancy/rollback | **Best** (everything, including the updater, can roll back) | Good (the update components themselves are hard to update redundantly) | Most limited |
| System complexity | Higher (data partition, bootloader redundancy) | Higher (data partition, initrd/update setup) | **Lowest** |

- **SWUpdate**:
  - A **"building block"** that **doesn't enforce a strategy**. Supports **A/B, single copy, or application** updates on **Buildroot or Yocto**.
  - Supports **signing**, integrates with **Eclipse hawkBit** for cloud deployment, and has an **integrated web server** for pushing images.
- **Mender**:
  - **Built around Yocto** (with layers providing **U-Boot modifications**); Buildroot is also supported.
  - **Built around A/B dual redundant updates** (application updates on some platforms since 2.0).
  - Supports **signing** and **delta updates**, with its **own open-source management server**, also available **hosted** (free starter tier).

### 2.4 MQTT and IoT Platforms
- **MQTT (Message Queuing Telemetry Transport), from 1999**:
  - A **lightweight publish/subscribe** messaging transport: **low bandwidth** and **efficient distribution to one or many receivers**. So "MQTT needs higher bandwidth than most protocols" is **false**.
  - **Popular on IoT platforms.** Secured with **SSL/TLS** (the same technology as https). Open-source **Mosquitto** client/server.
- **Basic security scheme**:
  - A secure pipe with the **server's identity verified by a CA-signed certificate**. The **client identifies itself by sending a private key or password over the encrypted channel**.
  - **Risk**: a **man-in-the-middle who obtains that key can emulate the device**.
- **MQTT SSL (mutual) security scheme**:
  - A secure bidirectional pipe where **both server *and* device identity are verified by Certificate Authority–*signed* certificates**. **No private key or password is transmitted.**
  - **A hacker who compromises the channel can view/modify data but *cannot emulate the device***, since emulating it needs the certificate's private key, which stays on the device.
- **JSON (JavaScript Object Notation)**, from ~1999 as part of the JavaScript standard:
  - **Language independent and text based** (more human-readable than binary).
  - **Objects** are ordered name/value pairs in braces (`"name": value`, comma-separated).
  - **Easy for embedded software to parse.** *It's not obfuscated and not binary-packed.*
- **Device shadows / digital twins** (AWS IoT and Azure "Device Shadows", Google "Device Registry", IBM Watson "Digital Twins"):
  - The cloud **keeps the device's state**, so it can be **read** (e.g., the last reported state with a timestamp) and **configured** (desired state) **even when the device or the phone can't reach the Internet**.
  - The server computes the **delta** between desired and reported state.
  - Example: `{"state":{"reported":{"frontdoor":"open"}}}` → the phone sets `{"state":{"desired":{"frontdoor":"close"}}}` → the server sends `{"state":{"delta":{"frontdoor":"close"}}}` → the device reports "closed".
  - **Shadows still need a server; they don't eliminate Internet access.**

### 2.5 Read-Only Root Filesystems
- **OverlayFS** (mainline kernel since ~3.18; the course says 3.8) **virtually merges an *upper* and a *lower* directory** into one overlay view. The two can be on different volumes, with different permissions and filesystem types.
  - **Reads come from upper if the file exists there, otherwise from lower.**
  - **Modifying a lower-only file copies it up to the upper layer first ("copy-up")**, so **the lower layer is never changed**.
  - **Deleting a lower file** creates a **whiteout**: a **character device with major/minor 0/0** in the upper layer, which **hides the lower file without actually deleting it**.
  - A **workdir** (empty, **on the same filesystem as upperdir**) is required: `mount -t overlay overlay -o lowerdir=…,upperdir=…,workdir=… /merged`.
- **tmpfs**:
  - A **RAM-backed** filesystem for **short-lived content that can be lost at reboot** (it **uses RAM, not disk**).
  - `mount -t tmpfs tmpfs /mnt/x`. The **default max size is half of physical RAM**; limit it with `-o size=1m`.
- **Why a read-only rootfs?**
  - **Avoids unintentional writes.**
  - **Reduces filesystem corruption possibilities** (e.g., on power loss).
  - **Simplifies factory-reset strategies** (just wipe the data partition).
  - **Simplifies device update strategies.**
  - *It is NOT easier to set up or modify.*
- **With A/B updates**, writes to the rootfs would be **lost when you switch partitions**. A read-only rootfs **enforces that modified content goes to the data partition**, so it isn't lost.
- **Setting it up**: kernel parameter **`ro`** (on the Pi, in `cmdline.txt`). But **apps that write to the rootfs break**, e.g. **dropbear** generating SSH keys or **DHCP** writing lease/resolv data, so you must handle them. Split storage into **read-only rootfs**, **read/write persistent data**, and **volatile data (recreated each boot)**.
- **Implementation options**:
  1. **Symbolic links** from rootfs paths to the **data partition** (persistent) or **tmpfs** (volatile).
  2. **OverlayFS on specific directories**, with the **upper layer on tmpfs** (volatile) or the **data partition** (persistent).
  3. **Overlay the *entire* rootfs** with a writable partition, using **`chroot`** from the initramfs. You don't need to know which directories need writes, but it's harder to set up (there's a Raspbian script).
  - *"Ensure no application ever writes to the rootfs" isn't a practical option; you handle the writes with the methods above.*
- **Yocto support**: **`EXTRA_IMAGE_FEATURES += "read-only-rootfs"`** and **`VOLATILE_BINDS`** (maps rootfs paths onto volatile or data storage).
- **`chroot`** changes the apparent root directory for a process tree. It's used to switch roots and typically needs bind mounts of /proc, /sys, /dev (the FHS is a user-space convention).

### 2.6 Application Containers and Docker
- **Containers**: a **logical packaging mechanism that abstracts applications from the environment they run in**, deploying consistently across a datacenter, the cloud, a laptop, or an embedded device.
  - **Developers focus on app logic and dependencies** (which libraries and versions were tested).
  - **IT focuses on deployment and management** (start/stop, memory and storage limits, ports).
- **Container vs VM**: **a container is like a VM without a dedicated kernel: it shares the host kernel** (no hypervisor and no guest kernel). Each **VM has its own kernel** via a hypervisor. **They are not the same.** (Containers *do* have their own root filesystem view.)
- **Kernel features behind containers**:
  - **Namespaces**: **virtual, isolated instances of a global resource**, created with the **`clone()`** syscall (like fork), covering **networking, ports, mount points, process IDs**.
    - **PID namespaces (`CLONE_NEWPID`)** create **nested process trees**: **a child can't see or kill processes in the parent namespace** (the parent *can* see the children).
    - **Network namespaces (`CLONE_NEWNET`)** give a different set of interfaces (e.g., only loopback). **Parent ports may be inaccessible from the child**; virtual routing between namespaces is possible.
  - **Control groups (cgroups)**: **organize processes into hierarchical groups and enforce limits** (**CPU time, memory**) through the cgroupfs pseudo-filesystem. They can also **freeze and resume** groups, which lets the **host restrict a child container's total CPU**.
  - **Union mounts**: **combine multiple directories into one apparent rootfs**. History: started in the early '90s for writable CDs, then UnionFS and aufs; **OverlayFS is the first in the mainline kernel**.
- **Docker**: an open platform for containerized apps, built on a **client–server engine**.
  - **dockerd (daemon)** starts, runs, and controls containers. The **docker client (CLI)** sends it commands (`build`, `pull`, `run`). **A registry** (e.g., **Docker Hub**) **stores and shares images**, so you'd find a third party's image **in a registry**.
  - Ubuntu setup: add the Docker CE apt source, install docker-ce and the CLI, then `sudo docker run hello-world` (pulls from Docker Hub and does a multi-stage build).
- **Docker on Buildroot** (**yes, it's supported**):
  - Add **docker-engine** (needs the **glibc toolchain** instead of the RPi default) and **docker-cli**.
  - **Increase the rootfs size** (128 MB default → ~500 MB).
  - Run **`check-config.sh`** to find the required kernel options (cgroups, overlay), then rebuild the kernel.
- **When to use containers in embedded work**:
  - **On the development host**: a **consistent build environment**, **automated test frameworks**, and **CI/CD** deployment.
  - **On the target**: **pull in dependencies quickly for tests** (e.g., run Ubuntu on Buildroot and `apt-get install`), and **support optional or conflicting configurations** (a component needing specific Python versions, or a second web server on another port).

> **Module 2 Cheat-Sheet Candidates:** OTA goals: security (no obfuscation), reliability (power loss), ease of use; private key signs, public key verifies; public key + signature + input reveal nothing about the private key; SHA hash for fixed size and change sensitivity; private key never leaves dev; signed hash == computed hash; not quantum-proof (NIST PQC); A/B vs single copy vs application table; data partition; SWUpdate (building block, hawkBit, web server) vs Mender (Yocto/A-B, delta, hosted server); MQTT 1999, pub/sub, low bandwidth, TLS; mutual TLS → can't emulate the device; CA-signed certificate; JSON text-based; shadows = state when offline; OverlayFS upper/lower/workdir, copy-up, whiteout = char 0/0; tmpfs = RAM, lost at reboot, default half of RAM; read-only rootfs benefits; symlinks / overlay dirs / overlay everything + chroot; container shares the kernel; namespaces (clone, PID/net), cgroups (CPU/mem limits), union mounts; Docker daemon/client/registry; Docker works on Buildroot (glibc, bigger rootfs).

---

<a id="module-3"></a>
## Module 3 — Git Workflows, C++ Templates/STL, C Dependency Generation

### 3.1 Git Workflows
- **Feature branch workflow**:
  - **Feature development happens on dedicated feature branches, not on master.** **Multiple developers can work on features without disturbing the main code base.** **The master branch is never broken.**
  - **A feature ends with a pull/merge request, which is where code review happens**: *before* the merge, not after.
  - Steps: `git checkout master`, `git pull origin master`, `git checkout -b issue-25` (or `feature/foo`, `fix-foo`; the instructor prefers **issue-N**, which links to the tracker), commit, `git push origin issue-25`, code review (comments and changes), then merge.
  - **Features can be merged in any order relative to when they started.** If master moved on (another feature merged first), either **add a merge commit** (git aligns the histories, though that combination hasn't been tested) or **rebase the feature branch** onto the new master (it replays the commits, rewriting the *feature* branch's history, so collaborators must force-update). *Don't rewrite master's history.*
- **GitFlow**:
  - Based on the feature branch workflow, but **all development moves to a `develop` branch** (feature branches merge into develop).
  - **`master` only tracks release history** (updated only for releases to customers or QA); **release branches** are cut from develop.
  - **Hotfixes branch off master** (not develop), get released (e.g., v0.2), and are merged back into develop, without disrupting in-progress development.
  - **Difference from plain feature branching**: in GitFlow **master is reserved for releases and hotfixes**.

### 3.2 C++ Templates, STL, Boost
- **Strong typing** in C/C++ means separate functions for each type (e.g., `min` for int and for double), which is tedious and error-prone.
- **The C solution is a macro**, `#define min(a,b) ((a) < (b) ? (a) : (b))`. Drawbacks:
  - **Parameters aren't type-checked.**
  - **Parameters aren't evaluated before pasting**: the whole expression is pasted in, **evaluated at each use**, and **side effects repeat**.
  - **Example** (quiz): `i=6, j=20; k = min(++i, ++j);` expands to `((++i) < (++j) ? (++i) : (++j))`, so i becomes 7, j becomes 21, 7 < 21, then **++i → 8. k = 8.** (The lecture example `i=10, j=5` gives **m = 7**, because ++j runs twice.)
- **Function templates**: `template <typename T> T min(T a, T b) { return a < b ? a : b; }`.
  - Call with `min<int>(a, b)`, or let the compiler deduce T from the arguments.
  - **The compiler generates a separate function per type, with type checking, and evaluates arguments once** (like any function).
  - **Mixed types fail deduction**: `int i=10; double j=20.3; int k = min(i, j);` is a **compiler error** (T can't be both int and double). Fix it with `min<int>(i, j)`, which casts j to int.
  - **Reason to use templates**: avoid **rewriting functions with the same behavior for different parameter types**.
- **Class templates**: `template <class T> class mypair { T a, b; … };`, declared as `mypair<int>`, `mypair<double>`.
- **STL = Standard Template Library**: template-based classes supported by C++ compilers.
  - **`string`**: a dynamically sized `char*` with **dynamic allocation and append**, concatenation with `+`, and searching. That's the reason to use it over `const char*`.
  - **`vector`** (dynamic array), **`list`** (doubly linked), **`slist`** (singly linked).
  - **`map`**: an associative array with non-numeric or non-sequential keys, e.g. `device["SerialNumber12345"] = "found"`.
  - *(Not STL: "webserver", "mqttiot".)*
- **Streams**: the **insertion operator `<<` with `cout`** (and `cin`, `cerr`) for I/O. Classes can define their own `<<` operator.
- **`vector`**:
  - **Its size isn't specified; it grows dynamically** with **`push_back()`**.
  - Iterate with **`v.begin()` / `v.end()`** and `++`; use a `const_iterator` when you won't modify the elements.
- **Boost**:
  - **Peer-reviewed, portable C++ libraries** that work with the STL and are **candidates for future standardization**. **Permissively licensed**, so they're fine for closed source.
  - Includes **Thread** (portable across Windows/Mac/Linux; pthreads isn't native on Windows), **Regex**, and **Interprocess** (shared memory, semaphores, mutexes, message queues).
  - **Use it when you need peer-reviewed capabilities not in the STL.**

### 3.3 C Dependency Generation (GNU Make, Autotools, CMake)
- **Why**: **when a header changes, every object file whose source `#include`s it must be rebuilt**.
  - Listing headers as prerequisites by hand works for tiny projects but gets **cumbersome and error-prone** in larger ones (headers include other headers).
  - Automating it means **object files rebuild automatically when included headers change**. (It's object files that rebuild, not source files.)
- **GNU Make + GCC**:
  - GCC generates Make-compatible dependency files: **`-MMD`** generates them, **`-MT`** sets the target name, **`-MF`** sets the output file. (Older projects did this with `sed`.)
  - **A `.d` file contains GNU Make–formatted targets and dependencies** (text), e.g. `main.o: main.c file1.h file2.h`.
  - **Setup**: put the `.d` files in a hidden directory (e.g. `.d/`), compile with `DEPFLAGS`, and `include $(wildcard $(DEPFILES))`.
  - **Pitfalls**: dependency tracking still **isn't fully automatic** (e.g., the Makefile itself isn't a dependency); **platform-specific** (assumes **GCC** and that binutils are installed); **won't build on other platforms**.
- **Autoconf/Automake (Autotools)**:
  - **Autoconf dates from 1991.** It **produces `configure` scripts** for building, installing, and packaging, and is **implemented with Bash scripts on the host**.
  - Inputs: **`configure.ac`** (the project description) + **`Makefile.am`**, then **`autoreconf`** → **`configure`**.
  - **Running `./configure`** **generates a Makefile with correct dependencies** for this platform (it detects gcc, cross-compiling, etc.), possibly plus a `config.h` of `#define`s.
  - **The tell-tale install sequence: `./configure && make && sudo make install`.**
  - **Buildroot support: `$(eval $(autotools-package))`** auto-configures cross-compilation and installs into `output/target`.
  - **Pitfalls**: intimidating, complex, varied use cases; **historical baggage**; **tailored to GNU software workflows** (a reason to pick it is **tighter integration with GNU workflows**); **poorly supported on non-Unix systems such as Windows**. So "**Autotools is a common choice on Windows**" is **false**.
- **CMake**:
  - **A cross-platform, open-source, compiler-independent build-process manager.** Configured with **`CMakeLists.txt`** (minimum version, project name, executable plus its sources; headers are handled automatically).
  - **Generates native build files**: **GNU Makefiles on Linux/Unix**, **Visual Studio projects/workspaces on Windows** (recent VS opens CMake folders directly). It works with GCC.
  - **Usage**: `mkdir build && cd build && cmake .. && make` (the build directory isn't committed).
  - **Buildroot: `$(eval $(cmake-package))`**. Yocto supports it natively too.
  - **Benefits over Make and Autotools**: **more output formats (VS)**, **generates GNU makefiles**, **fewer configuration lines for basic projects**, **better Windows support**, works with GCC.
- **Buildroot package infrastructures** can use **GNU Make (generic-package), Autotools (autotools-package), and CMake (cmake-package)**: all three.

| | GNU Make | Autoconf/Autotools | CMake |
|---|---|---|---|
| Cross-platform support | Limited | Better | **Best** |
| Configuration complexity | Complex | Easier (arguably) | **Easiest** (arguably) |

- **Instructor's advice**: **for new projects, try CMake first.**

> **Module 3 Cheat-Sheet Candidates:** feature branches, master never broken, code review in the PR before merge, out-of-order merges → merge commit or rebase the feature branch; GitFlow: develop for integration, master = releases, hotfixes off master; macro pitfalls (no type check, repeated evaluation) → min(++i,++j) with 6, 20 = 8; template mixed int/double = compile error; STL = Standard Template Library (string, vector, list, slist, map); cout <<; vector grows (push_back); boost = peer-reviewed, permissive, thread/regex/IPC; dependency gen = rebuild .o on header change; -MMD/-MT/-MF, .d = make rules (text); ./configure && make && sudo make install = Autotools (1991, GNU workflow, poor on Windows); CMake = CMakeLists.txt, generates Makefiles or VS projects, best cross-platform; Buildroot supports all three.

---

<a id="module-4"></a>
## Module 4 — Embedded Software Testing, CI/CD, TDD/BDD

### 4.1 Testing Concepts (Part 1)
- **Manual testing is almost always needed on embedded systems**, at least in some areas (hardware and human interaction are hard to automate). Start manual tests with the hardest-to-automate parts.
- **Automated testing** brings **repeatability and speed**: it runs any time (even at midnight) and more often.
- **ISTQB principles** (International Software Testing Qualifications Board):
  1. **Testing shows the presence of defects, not their absence.**
  2. **Exhaustive testing isn't possible**, so be strategic about what to test. ("Exhaustively test every scenario" is **false**.)
  3. **Early testing saves time and money.**
  4. **Defects cluster**: when you find one bug, look for similar ones.
  5. **The pesticide paradox**: **tests must be continually expanded and developed** to keep finding new defects as the software changes. Running the same tests forever stops finding bugs in new areas.
  6. **Testing is context dependent**: safety-critical airliner software isn't tested like a mobile app.
  7. **The absence of errors is a fallacy**: finding and fixing bugs doesn't mean the software meets its objectives.
- **Definitions**:
  - **Test case**: **a set of preconditions, inputs, actions, expected results, and postconditions**.
  - **Test execution**: **an activity that runs a test on a component or system, producing results**. It's linked to a specific **software version**, (for embedded) a **hardware version**, and possibly external dependency versions.
  - **Test suite**: **a set of test scripts or procedures executed in a specific test run** (grouped by area, e.g. the web UI suite).
- **Test levels**:
  - **Component testing**: **individual components in isolation**, often by the developer who wrote the code.
  - **Integration testing**: **interactions between components or systems**.
  - **System testing**: **behavior of the entire system**; end-to-end tasks.
  - **Acceptance testing**: system testing **to gauge readiness for customer deployment**, e.g. **alpha and beta testing at customer sites**.
- **Manual test strategy**:
  - **Record the plan as a test case** with **step-by-step detail** that doesn't need deep software knowledge, so **someone other than the developer can run it**.
  - **Record each execution**: what was done, when, and with which software, dependencies, and devices.
- **Automated test strategy**:
  1. **Automate the build** first.
  2. **Merge developers' code into a shared mainline often** (**continuous integration**).
  3. **Integrate tests into the development environment** and **run all automated tests whenever code reaches mainline**.

### 4.2 Automated Testing (Part 2)
- **Test framework**: **creates executables on the run platform** (often the dev host, not the target), **groups tests by feature**, **selects tests to run**, and **reports status**.
- **Test mocks** simulate real objects in controlled ways. They're needed for:
  - **non-deterministic results** (current time, temperature)
  - **states that are difficult to reproduce** (device or network errors)
  - **slow operations or ones that need lots of setup** (databases)
  - **components that don't exist yet or may change**
  - **isolating components for component testing**
  - *They don't convert automated tests into manual ones.*
  - Example: **if a coworker is waiting on *your* component, they can write a mock of *your* component** and test now.
- **Unity** (ThrowTheSwitch, pure C):
  - Write `test_` functions with assertions (`TEST_ASSERT_…`) in `test/Test_xxx.c`.
  - A **Ruby script generates the test runner** (`main()` calling every `test_` function).
  - Add Unity as a submodule in CMake, then build and run the `unity-test` executable. (A1–A9 used Unity.)
- **CMock**: auto-generates **mocks and stubs for C functions** from header files using Ruby.
  - Calls to, e.g., `ParseStuff()` are redirected to the mock.
  - **`_ExpectAndReturn`** checks the arguments and returns a value. **`_IgnoreAndReturn`** ignores the arguments. **`_StubAndCallback`** calls a custom function.
- **Ceedling**: a **Ruby/Rake wrapper around Unity + CMock + CException** (C++-style exceptions in C).
  - `ceedling new <proj>`, put code in `src/` and tests in `test/test_*.c`, then run `ceedling test:all`.
- **GoogleTest**: **C++-centric** (can test C too), with verbose failure messages via streams and **googlemock**. **Used by OpenCV and ROS.**

### 4.3 CI/CD (Part 3)
- **Continuous Integration / Continuous Deployment**:
  - With multiple developers, **merge to a shared branch often (ideally daily)**.
  - Each merge triggers a **CI pipeline** that verifies the **build** still succeeds, **automated tests** pass, the code **deploys to hardware**, and **automated tests on hardware** pass.
  - **Pipeline stages**: **build → unit test → automated test → deploy** (plus hardware tests). Code review, manual testing, feature work, and bug fixing aren't automated pipeline stages.
- **Jenkins**:
  - **Open source, self-deployed**, **created in 2004** (so not "recent").
  - Has a web UI and a text **Jenkinsfile** pipeline, and **can deploy to devices** (a good fit for device update).
  - **Newer options integrate more tightly with git tools**, so "Jenkins is a recent platform with the best git integration" is **false**.
- **YAML ("YAML Ain't Markup Language")**:
  - Like JSON, but **prioritizes human readability** and serializing native data structures. **More readable than JSON, but more complex to generate and parse.**
  - It's positioned as a **replacement for XML** ("lessons learned from XML"). The statement "XML is a replacement for YAML" is **backwards/false**.
- **Travis CI**:
  - A **hosted** CI service that integrates with GitHub, **free for open-source** (public) repos. **Can't handle Yocto or Buildroot builds** (too big).
  - A `.travis.yml` with `language:` and `script:` (**non-zero exit = failure**). It re-runs on every commit and provides a **status badge** for the README.
- **GitLab CI**:
  - **GitLab is an open-source GitHub alternative** (self-hosted, or hosted at gitlab.com) with built-in CI using a `.gitlab-ci.yml` file.
  - **Fully integrated**: builds start on each commit and report through the same dashboards and email.
  - **Runners can be self-hosted on-premise, attached to physical hardware**, while the server is cloud-hosted. (The course's yocto-assignments-base has a `.gitlab-ci.yml`; GitHub Actions self-hosted runners played the same role in the assignments.)

### 4.4 TDD and BDD (Part 4)
- **Test-Driven Development (TDD)**:
  - **Write the requirements as test cases first**; they **fail** because there's no code yet. **Write the software until the tests pass.**
  - **Pitfall**: the developer usually writes both the tests and the code. **If they misread the requirements, both are wrong**, giving false success.
- **Behavior-Driven Development (BDD)**:
  - Test cases are written as **"scenarios" in natural language that non-programmers can read**, describing **purpose and benefit rather than technical details**. That **lets someone other than the developer add and review the requirements** that drive the automated tests.
  - Uses **Gherkin** syntax: `Feature:` / `Scenario:` / **`Given` … `When` … `Then`**.
  - Frameworks: **behave (Python)** and **Cucumber (Ruby)**. Step implementations are written in Python or Ruby; **the scenarios themselves are written in Gherkin**.

> **Module 4 Cheat-Sheet Candidates:** manual is still needed on embedded; 7 ISTQB principles (presence not absence, no exhaustive testing, early, clustering, pesticide paradox, context, absence-of-errors fallacy); test case (preconditions/inputs/actions/expected/postconditions) vs execution (runs → results, tied to SW/HW versions) vs suite (set run together); component/integration/system/acceptance (alpha/beta); mocks (non-deterministic, hard-to-reproduce, slow, nonexistent → mock the other person's component); Unity/CMock/Ceedling (C, Ruby), GoogleTest (C++, OpenCV/ROS); CI pipeline (build, unit test, automated test, deploy, HW test); Jenkins 2004 self-hosted; YAML = readable, replaces XML; Travis hosted, can't do Yocto/Buildroot; GitLab CI self-hosted runner on hardware; TDD tests first (pitfall: same person misreads the requirements); BDD = Gherkin Given/When/Then, behave/Cucumber, non-developers can review.

---

<a id="quick-reference"></a>
## Quick-Reference Tables

### Boot sequence (complex SoC)
| Phase | Runs from | Does |
|---|---|---|
| 1. ROM code | On-chip ROM (reset vector) | Loads the SPL into on-chip SRAM (from NAND special area or USB) |
| 2. SPL (MLO) | SRAM | Initializes DRAM; loads the TPL |
| 3. TPL (U-Boot) | DRAM | Full NAND/ECC; loads the FDT, kernel, initramfs; `bootm` |
| 4. Kernel | DRAM | Initializes memory; initramfs sets up the rootfs → init |

### Update schemes
| | A/B | Single Copy | Application |
|---|---|---|---|
| Storage | 2× rootfs | 1× + update component | ~0 extra |
| Rollback | Best | Good | Limited |
| Complexity | Higher | Higher | Lowest |
| Example framework | Mender (A/B), SWUpdate | SWUpdate | apt/yum packages, SWUpdate |

### Signing
| Item | Where it lives | Purpose |
|---|---|---|
| Private key | **Dev environment only** | Sign (hash → signature) |
| Public key | Every device | Verify (signature → signed hash) |
| SHA hash | Computed on both sides | Fixed size; any change → different hash |
| Accept update iff | signed hash == computed hash | |

### Container kernel features
| Feature | Purpose |
|---|---|
| Namespaces (clone: NEWPID, NEWNET, mounts) | Isolated views: processes, network, ports, mounts |
| Control groups | Limit CPU and memory; freeze/resume |
| Union mounts (OverlayFS) | Merge directories into one rootfs |

### Build tools
| Tool | Config file | Buildroot infrastructure | Signature command |
|---|---|---|---|
| GNU Make | Makefile (+ gcc -MMD .d files) | generic-package | `make` |
| Autotools | configure.ac, Makefile.am | autotools-package | `./configure && make && sudo make install` |
| CMake | CMakeLists.txt | cmake-package | `mkdir build; cd build; cmake ..; make` |

### Testing vocabulary
| Term | Meaning |
|---|---|
| Test case | Preconditions + inputs + actions + expected results + postconditions |
| Test execution | Running a test → results (tied to SW/HW versions) |
| Test suite | Set of tests run together |
| Component / Integration / System / Acceptance | Isolation / interactions / whole system / customer readiness (alpha/beta) |
| Mock | Controlled stand-in for a real component |
| TDD | Tests first, then code |
| BDD | Gherkin Given/When/Then scenarios (behave, Cucumber) |
