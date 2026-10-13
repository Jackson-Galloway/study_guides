# ECEA 5307 — Practice Final Exam
### Linux Embedded System Topics and Projects

**Format:** like the real final: True/False, multiple choice, select-all.
**Suggested time:** 45 minutes (the real exam's limit). Use only your one-page notes sheet.
**Answers:** in `practice-exam-answers.md`.

These are new questions written from the lecture material, covering all four modules.

---

### Part A — True / False (1 point each)

1. In Scrum, management is typically considered a "pig" because the project's outcome directly affects their job. **T / F**

2. A Scrum development team typically has between 3 and 9 members. **T / F**

3. A sprint is typically one to two weeks long. **T / F**

4. Before Linux kernel 3.8, ARM platform information was commonly hard-coded into the kernel as platform data. **T / F**

5. A Device Tree Blob (.dtb) is also called a Flattened Device Tree (FDT). **T / F**

6. In a multi-phase boot, the Secondary Program Loader typically runs from DRAM before DRAM has been initialized. **T / F**

7. The U-Boot `bootcmd` variable defines commands run automatically at power-up. **T / F**

8. The MAC address shown by `ip addr` under `link/ether` is intended to be globally unique. **T / F**

9. In a signed update scheme, the public key should never leave the development environment. **T / F**

10. A/B dual redundant update schemes make factory reset easier because configuration data is kept on a separate data partition. **T / F**

11. Mender is a device update framework built primarily around A/B dual redundant updates. **T / F**

12. In OverlayFS, the work directory must be on the same filesystem as the upper directory. **T / F**

13. By default, a tmpfs mount can grow up to half of physical RAM. **T / F**

14. Containers require a hypervisor and a separate guest kernel. **T / F**

15. In GitFlow, hotfix branches are created from the develop branch. **T / F**

16. Rebasing a shared feature branch may require collaborators to force-update their copies. **T / F**

17. C++ template functions evaluate their arguments only once, unlike function-like macros. **T / F**

18. CMake can generate Visual Studio project files as well as GNU makefiles. **T / F**

19. Travis CI is well suited for running full Yocto builds. **T / F**

20. In Test-Driven Development, test cases are written before the software that satisfies them. **T / F**

---

### Part B — Multiple Choice (1 point each)

21. Which Scrum role enforces the process rules and coaches team members?
   - a) Product Owner
   - b) Scrum Master
   - c) Development Team
   - d) Management

22. Which is one of the three daily stand-up questions?
   - a) What is our budget for this sprint?
   - b) What is blocking my progress toward the sprint goal?
   - c) What will we ship next quarter?
   - d) Who should be promoted?

23. Why can't a simple NOR-flash style bootloader be used with NAND flash?
   - a) NAND flash is read-only
   - b) NAND isn't linearly addressable and needs ECC and bad-block handling
   - c) NAND flash can only be used with UEFI
   - d) NAND flash requires a hypervisor

24. Which command converts the running system's device tree back into readable source?
   - a) `dtc -I dts -O dtb`
   - b) `dtc -I fs -O dts /sys/firmware/devicetree/base`
   - c) `make dtbs`
   - d) `bootm`

25. Which file do you edit on a Raspberry Pi to enable SPI via `dtparam`?
   - a) cmdline.txt
   - b) config.txt
   - c) /etc/network/interfaces
   - d) start.elf

26. Why is a hash such as SHA used when signing firmware?
   - a) It encrypts the firmware so it cannot be read
   - b) It produces a fixed-size value that changes completely with any modification
   - c) It replaces the need for a private key
   - d) It compresses the firmware for faster transfer

27. On the device, an update is accepted only if:
   - a) The public key matches the private key
   - b) The signed hash recovered with the public key equals the hash computed from the update file
   - c) The update file is smaller than the previous version
   - d) The update was downloaded over HTTPS

28. Which update scheme generally has the most limited rollback capability?
   - a) A/B dual redundant
   - b) Single copy with recovery initrd
   - c) Application (package) update
   - d) All are equal

29. Which device update tool is described as a flexible "building block" that supports A/B, single copy, or application updates and integrates with Eclipse hawkBit?
   - a) Mender
   - b) SWUpdate
   - c) apt
   - d) Jenkins

30. In MQTT's mutual TLS scheme, what can an attacker who compromises the channel NOT do?
   - a) Observe traffic
   - b) Modify traffic
   - c) Emulate the device
   - d) Delay traffic

31. In an AWS IoT device shadow, which state section contains the difference between desired and reported state?
   - a) reported
   - b) desired
   - c) delta
   - d) metadata

32. Which kernel feature limits the CPU and memory available to a group of container processes?
   - a) Namespaces
   - b) Control groups
   - c) Union mounts
   - d) chroot

33. Which component of Docker stores and shares images?
   - a) Docker daemon
   - b) Docker client
   - c) Docker registry
   - d) Dockerfile

34. Given `#define SQ(x) ((x)*(x))` and `int i = 3; int k = SQ(i++);`, what is the core problem?
   - a) The macro is type checked
   - b) The argument expression is evaluated more than once
   - c) Macros cannot take arguments
   - d) The result is always 9 with no side effects

35. Which STL container is an associative array with non-sequential keys?
   - a) vector
   - b) list
   - c) map
   - d) string

36. Which GCC flag generates make-compatible dependency files?
   - a) -O2
   - b) -MMD
   - c) -Wall
   - d) -g

37. Which test level evaluates interactions between components?
   - a) Component testing
   - b) Integration testing
   - c) System testing
   - d) Acceptance testing

38. Which statement reflects the "absence of errors fallacy"?
   - a) Testing can prove there are no bugs
   - b) Finding and fixing bugs doesn't guarantee the software meets its objectives
   - c) Bugs never cluster
   - d) Manual testing is never needed

39. Which CMock helper ignores the passed arguments and always returns a configured value?
   - a) `_ExpectAndReturn`
   - b) `_IgnoreAndReturn`
   - c) `_StubAndCallback`
   - d) `_Verify`

40. Which keywords form the core structure of a Gherkin scenario?
   - a) Begin / Process / End
   - b) Given / When / Then
   - c) Setup / Run / Teardown
   - d) Try / Catch / Finally

---

### Part C — Select All That Apply (2 points each)

41. Which describe a good Definition of Done (DoD) for a sprint issue? (select all)
   - a) Understandable by someone outside the project team
   - b) Updated to match what was actually completed
   - c) Includes how the work was verified
   - d) Kept vague so it can be closed easily

42. Which are reasons to use a read-only root filesystem? (select all)
   - a) Avoids unintentional writes
   - b) Reduces filesystem corruption possibilities
   - c) Simplifies factory reset
   - d) Makes it easier to install new packages at runtime

43. Which applications commonly need special handling when the rootfs is mounted read-only? (select all)
   - a) dropbear (SSH host key generation)
   - b) DHCP client (lease/resolver files)
   - c) Applications that save user configuration
   - d) The bootloader's ROM code

44. Which are valid uses of containers in embedded development? (select all)
   - a) Consistent build environments across developers
   - b) Automated test frameworks and CI/CD
   - c) Quickly pulling in dependencies on the target for testing
   - d) Replacing the Linux kernel on the target

45. Which are characteristics of the git feature branch workflow? (select all)
   - a) Master is never broken
   - b) Code review happens on pull/merge requests
   - c) All development happens directly on master
   - d) Feature branches are often named after tracker issues (e.g., issue-25)

46. Which are true about Autoconf/Automake? (select all)
   - a) Produces a configure script
   - b) Uses configure.ac and Makefile.am inputs
   - c) Is the best-supported option on Windows
   - d) Is supported in Buildroot via autotools-package

47. Which are ISTQB testing principles from lecture? (select all)
   - a) Early testing saves time and money
   - b) Defects are often clustered
   - c) Exhaustive testing is possible with enough automation
   - d) Testing is context dependent

48. Which situations call for a test mock? (select all)
   - a) Testing behavior that depends on the current time
   - b) Simulating a network error that is hard to reproduce
   - c) A dependent component isn't written yet
   - d) Verifying the final product with real customers

49. Which belong in a CI pipeline for an embedded Linux product per lecture? (select all)
   - a) Verify the build succeeds
   - b) Run automated tests
   - c) Deploy to hardware
   - d) Run automated tests on hardware

50. Which statements about YAML are correct per lecture? (select all)
   - a) It prioritizes human readability
   - b) It is generally more complex to parse than JSON
   - c) It was designed with lessons learned from XML
   - d) It is a binary format

---
*End of practice exam: 50 questions, longer than the real 45-minute exam so you get extra practice. Scoring: A 20 + B 20 + C 20 = 60 points.*
