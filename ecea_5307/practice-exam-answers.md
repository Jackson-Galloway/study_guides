# ECEA 5307 — Practice Final Exam: Answer Key

## Part A — True / False

| # | Ans | Explanation | Review |
|---|---|---|---|
| 1 | **F** | Management is usually a **chicken** (involved); the PO, dev team, and SM are pigs (committed). | §1.1 |
| 2 | **T** | More than 9 gets unwieldy; fewer than 3 is too much overhead. | §1.1 |
| 3 | **T** | 1–2 weeks, not a month or longer. | §1.1 |
| 4 | **T** | Platform info was discovered at runtime or hard-coded; device trees replaced that. | §1.4 |
| 5 | **T** | .dtb = Device Tree Binary/Blob = FDT. | §1.4 |
| 6 | **F** | The SPL runs from small **on-chip SRAM** and *initializes* DRAM. | §1.4 |
| 7 | **T** | bootcmd is scripted at power-up. | §1.4 |
| 8 | **T** | The MAC address is a globally unique hardware ID. | §1.5 |
| 9 | **F** | The **private** key never leaves the dev environment; the public key goes on every device. | §2.2 |
| 10 | **T** | Logical separation of configuration from the image. | §2.3 |
| 11 | **T** | Mender is built around A/B (plus app updates on some platforms). | §2.3 |
| 12 | **T** | The workdir must be empty and on the same filesystem as upperdir. | §2.5 |
| 13 | **T** | Limit it with `-o size=`. | §2.5 |
| 14 | **F** | Containers share the host kernel; VMs use a hypervisor and guest kernel. | §2.6 |
| 15 | **F** | Hotfixes branch from **master**. | §3.1 |
| 16 | **T** | Rebase rewrites the branch's history. | §3.1 |
| 17 | **T** | Templates are real functions (and type checked). | §3.2 |
| 18 | **T** | It generates makefiles or Visual Studio projects. | §3.3 |
| 19 | **F** | Travis can't handle Yocto or Buildroot builds. | §4.3 |
| 20 | **T** | Tests first, then code until they pass. | §4.4 |

## Part B — Multiple Choice

| # | Ans | Explanation | Review |
|---|---|---|---|
| 21 | **b** | The Scrum Master enforces the rules and coaches. | §1.1 |
| 22 | **b** | Done since last time / plan until next time / blockers. | §1.1 |
| 23 | **b** | NAND isn't linearly addressable and needs ECC and bad-block recovery. | §1.4 |
| 24 | **b** | Input fs (/sys tree) → output dts. | §1.4 |
| 25 | **b** | config.txt (cmdline.txt holds kernel arguments). | §1.4 |
| 26 | **b** | Fixed size and change sensitivity. | §2.2 |
| 27 | **b** | signed hash == computed hash. | §2.2 |
| 28 | **c** | Rollback is complicated and states can be inconsistent. | §2.3 |
| 29 | **b** | SWUpdate (Mender is the A/B, Yocto-centric one). | §2.3 |
| 30 | **c** | No secret is sent, so the device can't be impersonated (traffic can still be seen or modified if the channel is compromised). | §2.4 |
| 31 | **c** | The server computes the delta. | §2.4 |
| 32 | **b** | cgroups. | §2.6 |
| 33 | **c** | The registry (e.g., Docker Hub). | §2.6 |
| 34 | **b** | The macro pastes the expression in twice (repeated side effects; actually undefined behavior here). | §3.2 |
| 35 | **c** | map. | §3.2 |
| 36 | **b** | -MMD (with -MT, -MF). | §3.3 |
| 37 | **b** | Integration testing. | §4.1 |
| 38 | **b** | Absence-of-errors fallacy. | §4.1 |
| 39 | **b** | _IgnoreAndReturn. | §4.2 |
| 40 | **b** | Given / When / Then. | §4.4 |

## Part C — Select All That Apply

| # | Answers | Explanation | Review |
|---|---|---|---|
| 41 | **a, b, c** | Peer-review standard: could an outsider verify it? | §1.2 |
| 42 | **a, b, c** | Read-only makes runtime installs *harder*. | §2.5 |
| 43 | **a, b, c** | ROM code isn't affected by the rootfs mount. | §2.5 |
| 44 | **a, b, c** | Containers share the host kernel; they don't replace it. | §2.6 |
| 45 | **a, b, d** | Development happens on feature branches, not master. | §3.1 |
| 46 | **a, b, d** | Autotools is poorly supported on Windows. | §3.3 |
| 47 | **a, b, d** | Exhaustive testing isn't possible. | §4.1 |
| 48 | **a, b, c** | Real customers = acceptance testing, not mocks. | §4.2 |
| 49 | **a, b, c, d** | All four from the CI/CD slide. | §4.3 |
| 50 | **a, b, c** | YAML is text. | §4.3 |

**Scoring:** 54–60 exam-ready · 45–53 review the missed sections · below 45 re-study the modules.
