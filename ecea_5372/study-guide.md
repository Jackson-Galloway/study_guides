# ECEA 5372 — Network Principles in Practice: Cloud Networking
## Final Exam Study Guide

*Built from the course lecture transcripts, slides, assigned paper slides, and the abstracts and key results of every assigned paper (instructor: Eric Keller, CU Boulder MS-ECE). All hands-on work uses Google Cloud Platform (GCP) and Terraform.*

**Final exam format** (from "About the Final Exam"): **unproctored**, **90 minutes**, **34 questions**, **2 attempts** (1 submission each). Multiple-choice questions plus **one free-response question**; no programming questions. Worth **15%** of the grade (three Terraform labs 15% each; five quizzes 8% each).
**Not allowed:** notes or external websites. There's no notes sheet, so the "Memorize" boxes below are things to know cold.

> **AI policy for this course: Conditional AI.** AI may be used to summarize material and generate practice questions (which is what this guide and the practice exam are). It may **not** be used to answer graded quizzes or labs, or to enter their questions into an AI tool. That's why there's no quiz answer key here. The [quiz coverage map](#quiz-map) at the end points to the sections each graded quiz draws on, without answering anything.

> The course mixes three kinds of material, and the exam will too: **(1) GCP concepts and how-to** (regions/zones, VPCs, firewall rules, peering, Cloud NAT, load balancers, VPN/Interconnect), **(2) Terraform**, and **(3) the research papers** behind the cloud (VL2, 4D/Ethane/OpenFlow/RMT/P4, Andromeda, AccelNet, SWAN/B4, Espresso, Maglev, Akamai). For each paper, know **the problem, the key idea, and the headline result**.

---

## Table of Contents
1. [Module 1 — Cloud Background (cloud, virtualization, VL2, GCP, Terraform)](#module-1)
2. [Module 2 — VPCs (GCP VPCs, SDN history, Andromeda, AccelNet)](#module-2)
3. [Module 3 — WAN (private IPs, VPC peering, Cloud NAT, SWAN, Espresso)](#module-3)
4. [Module 4 — Network Services (load balancing, tiers, autoscaling, Maglev, CDN, Akamai)](#module-4)
5. [Module 5 — Hybrid Networks (VPN, Direct Connect / Interconnect, cross-cloud)](#module-5)
6. [Paper Cheat Table and Quick Reference](#quick-reference)
7. [Graded Quiz Coverage Map](#quiz-map)

---

<a id="module-1"></a>
## Module 1 — Cloud Background

### 1.1 How to read an academic paper
- **Why papers:** the cloud required big innovations, and papers are how those innovations get communicated. SDN came out of academic work over many years. Google and Microsoft publish a lot.
- **Top venues:** networking: **SIGCOMM, NSDI, CoNEXT**; systems: SOSP, OSDI, ASPLOS, USENIX ATC; security: USENIX Security, IEEE S&P.
- **Parts of a paper:**
  - **Abstract:** a 1–2 paragraph overview.
  - **Introduction:** summarizes the **entire paper**: why it matters, the alternatives, the key insight, the system, the findings.
  - **Motivation / background / related work.**
  - **Architecture:** a high-level overview of the system.
  - **Implementation:** how it was built.
  - **Evaluation:** proves the claims and addresses concerns. **Microbenchmarks** look at key details; **macrobenchmarks** look at the whole system.
  - **Discussion**, then **conclusions / future work**.
- **Reading strategies:** intro + architecture figures (fast survey); focus on related work (entering a new area); a deep read with a focus on the evaluation.
- **Industry papers** may introduce less that's new, but they show what worked and what was hard **at scale**, and they reveal open problems.

### 1.2 What cloud computing is
- **Utility computing** (renting compute by the hour) goes back to **1966**.
- **AWS origin:**
  - Amazon was founded in 1994 as a bookstore and grew into general e-commerce.
  - Around 2000, building merchant.com (a platform for third-party sellers) forced it to untangle its systems into well-documented APIs.
  - Projects expected to take 3 months spent 3 months just building database/compute/storage pieces, so Amazon built **shared internal infrastructure services**.
  - It realized it was good at running reliable, scalable, cheap data centers, and sold that capability as **AWS**.
- **Berkeley "Above the Clouds" (2009): three things that are new about cloud computing:**
  1. **Elimination of an up-front commitment.** No colocation lease, buying servers/routers/firewalls, racking and stacking, configuring. No capital expense.
  2. **Pay for use on a short-term basis.** Fixed capacity is always either **over-provisioned** (wasted in the valleys) or **under-provisioned** (fails at peaks). **Elasticity** provisions what's needed right now.
  3. **The illusion of infinite computing resources.** Growth is hard to predict, and you don't want to stunt it.
- **Why it works as a business:**
  - **Economies of scale.** Electricity, network, storage and admin are about **5–7× cheaper** in very large data centers.
  - **Expertise** at running infrastructure.
- **Service models:**

| Model | Provider manages | You manage | Example |
|---|---|---|---|
| **IaaS** | Data center: compute, storage, networking, virtualization | **OS, middleware, VMs, apps, data** | VMs on GCP/AWS |
| **PaaS** | All the hardware and software to develop/run apps | Your app | Managed databases, app platforms |
| **SaaS** | The **entire application stack** | Just use it | Salesforce (CRM) |

- Reference pictures: IBM's XaaS ecosystem; the **NIST** reference architecture's five actors (**consumer, provider, carrier, auditor, broker**).
- **Summary of benefits:**
  - Trade capital expense for variable expense.
  - Massive economies of scale.
  - Stop guessing about capacity.
  - Speed and agility.
  - Focus on what matters (your business).
  - Go global in minutes.

### 1.3 Virtualization
- **The problem:** enterprise IT runs many services (Exchange, Oracle, SAP, web servers, Citrix…), each wanting its own OS version, hardware and configuration.
  - Co-locating them is hard: conflicting demands, incompatible loads.
  - Upgrades and commissioning are painful: shadow servers, complicated changeovers.
- **Virtualization:**
  - **Decouples the (OS, service) pair from hardware.**
  - **Multiplexes lightly used services** onto shared hosts.
  - **Migrates** services between hosts.
  - Makes it easy to add new (OS, service) pairs for commissioning, testing upgrades, or experiments.
- **Virtual machine:** presents an interface **identical to bare hardware** (devices, interrupts, memory, page tables).
  - The **hypervisor** (virtual machine monitor) creates the illusion of multiple independent processors.
  - VMs share nothing except via network protocols.
- **Bare metal → VMs → containers:**
  - **Bare metal** wastes resources when one app per server.
  - **VMs** solve the different-OS, variable-workload and conflicting-dependency problems, but are **heavyweight** (extra OS and binaries) and have overhead.
  - **Containers** put isolation in the OS (**namespaces + cgroups**). History: chroot 1982 → jails 2000 → LXC 2008.
- **In the cloud:** a developer requests a VM through the portal, it lands on some server in a data center, and users reach it over the Internet.

### 1.4 VL2: Azure's data center fabric (SIGCOMM 2009)
- **Cost is huge:** "upwards of **$12 million per month** for a 100,000-server data center" (2009). AWS later spent $12B on 5 new data centers. Efficiency matters.
- **Tenets of a cloud-scale data center:**
  - **Agility:** assign **any server to any service**, which boosts utilization (statistical multiplexing).
  - **Scale out:** use large pools of **commodity** gear for reliability, performance and low cost (economies of scale).
- **Problems with the conventional (Cisco reference) design** (core routers → access routers → L2 switches → racks; ~1,000 servers per pod = one IP subnet/VLAN):
  - Dependence on **expensive proprietary routers**.
  - **Very limited server-to-server capacity** (oversubscription grows up the tree: ~5:1, ~40:1, ~200:1).
  - **Resource fragmentation:** services are stuck inside IP subnets/VLANs, so moving them needs complicated manual L2/L3 reconfiguration → lost revenue and wasted expense.
- **Measured traffic:** highly **volatile** (many distinct patterns per day) and **unpredictable** (weak correlation), so traditional traffic engineering would have to re-optimize constantly.
- **Goal: "a huge L2 switch, or an abstraction of one":**
  1. **L2 semantics** (any VM anywhere, flat addresses).
  2. **Uniform high capacity** between servers.
  3. **Performance isolation** between services.
- **How VL2 does it:**

| Objective | Approach | Solution |
|---|---|---|
| 1. L2 semantics | **Flat addressing** | **Name-location separation** + a **directory service** (replaces ARP broadcast) |
| 2. Uniform high capacity | Guarantee bandwidth for hose-model traffic | **Flow-based random traffic indirection (Valiant Load Balancing)** on a **Clos** topology |
| 3. Performance isolation | Enforce the hose model with **existing mechanisms only** | **TCP** |

- **Name-location separation:**
  - Servers keep flat names. Switches run **link-state routing** and only track the **switch-level topology**.
  - A sender looks up "where is z?" in the **directory service** (z → ToR3) and **encapsulates** the packet to that top-of-rack switch, which decapsulates it.
  - If z moves to ToR4, only the directory entry changes (z keeps its identity).
  - Benefits: low-cost switches; protects network and hosts from host churn; no host or switch reconfiguration.
- **Clos topology:** huge aggregate capacity and **many paths at modest cost**. With D-port aggregation switches it supports 20·(D²/4) servers: D = 48 → 11,520 servers; 96 → 46,080; 144 → 103,680.
- **Valiant Load Balancing (random indirection):** bounce each flow off a **randomly chosen intermediate switch**, then down to the destination.
  - **Randomness** spreads load so **arbitrary traffic matrices** can be handled without clever traffic engineering.
  - Implemented with standard **ECMP + IP anycast**: harnesses bisection bandwidth, avoids esoteric optimization, robust to failures, works with today's switches.
- **Lessons:**
  - **Randomization can tame volatility.**
  - **Add functionality where you have control** (the hypervisor/host layer intercepts and encapsulates).
  - **There's no need to wait** (commodity parts work now).

### 1.5 Google Cloud Platform basics
- New users get free credits ($50 with just a Gmail address; $300 with a card).
- **Projects** (each has a name and an **ID**; the ID is what tools use).
- **Compute Engine → VM instances → Create:**
  - Name; **region** (e.g., us-central1, typically cheap); **zone** (us-central1-a; for fault isolation).
  - Machine family/series (**E2, "low cost"**); type **e2-micro** (about 1¢/hour); boot disk image.
- **A VM gets an internal (private) IP** (only reachable from your VPC network) **and optionally an external (public) IP.**
  - `ip addr` inside the VM shows **only the internal IP**. The external one is translated by Google's infrastructure (Andromeda).
- **SSH options:**
  - The console's SSH button (browser terminal; Google handles keys and authentication).
  - **gcloud CLI:** `gcloud init` (create/select a configuration and log in with an auth URL/token); `gcloud config configurations list/activate/delete`; then `gcloud compute ssh …` (the console's "View gcloud command" shows the exact command).
  - **Cloud Shell:** a free, browser-based Linux VM with gcloud preinstalled and persistent files.
- Stop (keeps the disk) vs **delete** the VM. Delete things so you aren't billed.
- WSL (Windows Subsystem for Linux) is recommended for running Linux tools on Windows.

### 1.6 Infrastructure as Code and Terraform
- **Infrastructure as Code (IaC):** apply software practices (**versioning, code review, pull requests**) to infrastructure. It's repeatable, manageable and reviewable.
- **Provisioning** infrastructure (launch VMs, define networks) vs **deploying** applications (configure the OS and packages; tools like Ansible). **Terraform shines at provisioning.**
- **IaC provisioning tools:**
  - Cloud-specific: **AWS CloudFormation, Google Deployment Manager, Azure Resource Manager**.
  - Cloud-agnostic: **Terraform, Pulumi**.
- **Architecture:** Terraform core plus **providers** (one per cloud/service, from the registry) plus local state files.
- **Declarative vs imperative:**
  - **Imperative:** you say **how** (step 1 create VM1, step 2 …).
  - **Declarative:** you say **what** (the desired state).
  - **Terraform is declarative.** It compares desired state with current state and makes changes in a **reconciliation loop**. It creates only what's missing (a third VM, or a VM someone deleted in the console).
- **Syntax:**
  - **Arguments** assign a value to a name.
  - **Blocks** are containers with a **type** (resource, provider, variable, …) and **0+ labels** (a resource has 2: the resource type and a local name). Blocks can nest.
  - Also: variables, conditions, loop-like constructs, explicit `depends_on`, functions.
- **Provider block** (GCP): `credentials = file("<service-account-key>.json")`, `project = "<project-id>"`, default `region`/`zone`.
- **Resource block:**
  ```hcl
  resource "google_compute_instance" "vm1" {
    name         = "mod1-tf-vm1"
    machine_type = "e2-micro"
    zone         = "us-west1-c"
    boot_disk { initialize_params { image = "debian-cloud/debian-11" } }
    network_interface { network = "default"  access_config {} }   # access_config = external IP
    metadata = { startup-script = "apt update; apt install -y …" }
  }
  ```
- **Service account:** an account for an **application or workload rather than a person**. Create it in IAM & Admin → Service Accounts, give it a role (e.g., **Compute Engine Service Agent**; least privilege is best practice), then **Keys → Create key (JSON)**. Terraform authenticates with that file; deleting the account makes the key useless.
- **Workflow:**
  - **`terraform init`** (download providers).
  - **`terraform plan`** (show what will change).
  - **`terraform apply`** (confirm with `yes`; creates resources).
  - `terraform show` (`--json | jq` to query attributes like nat IP and network IP).
  - **`terraform destroy`** (tear down everything, so you stop paying).
- **Lab "Starting Terraform with GCP":** create a project, install Terraform and gcloud, clone the repo, fill in `provider.tf` (credentials, project), deploy, and SSH in both ways.

> **Memorize — Module 1**
> - Berkeley: **no up-front commitment, pay-per-use (elasticity), illusion of infinite resources**. Works because of **economies of scale + expertise**.
> - IaaS (you manage OS and up) / PaaS (platform) / SaaS (whole app).
> - Virtualization: decouple OS+service from hardware; consolidate, migrate. VMs heavyweight → containers.
> - **VL2:** agility (any server, any service); goals = **L2 semantics, uniform high capacity, performance isolation**; **flat addressing + directory service**, **Clos**, **Valiant LB (randomness) via ECMP + anycast**, TCP for isolation. "Randomization tames volatility."
> - VM has internal IP (+ optional external, not visible inside the VM). Zones = fault isolation.
> - Terraform = **declarative** IaC; needs a **service account key**; **init → plan → apply → destroy**.

---

<a id="module-2"></a>
## Module 2 — Understanding and Creating VPCs

### 2.1 How the cloud is organized
- **Data centers:** huge buildings near good energy sources, all over the world.
- **Region:** a location of Google data centers, named by location (**us-west1, asia-east1**; `gcloud compute regions list`).
- **Zone:** within a region; **provides fault isolation**. Each zone has its own critical shared resources (**power, cooling, networking**). Zones are interconnected with fiber.
- **VPC (Virtual Private Cloud):** **your own private cloud running on the provider's physical infrastructure**. It's the core networking abstraction (AWS also calls it a VPC; Azure calls it a **VNet**).
  - **In GCP a VPC is global** (not tied to one region).
- **Subnet:** a partition of the VPC's IP space (e.g., 10.0.1.0/24 and 10.0.2.0/24). **Subnets are regional.** VMs get addresses from their subnet.
- **Routes:** direct traffic between subnets and to external gateways.
- **Firewall rules:** the policy for what to block or allow.

### 2.2 Creating a VPC in GCP (console and Terraform)
- **Console:** VPC networks → Create.
  - Subnet mode **custom** (define your own subnets, each tied to a region) vs **automatic** (one subnet in every region).
  - The **default** VPC is what VMs use if you don't specify one.
- **Create VM:** under **Advanced options → Networking**, choose the VPC and subnet; the internal IP can be ephemeral or reserved; the external IP can be ephemeral, static, or **none**; network tier premium/standard. **Management → startup script / metadata.**
- **Firewall concepts:**
  - Match-action tables.
  - **Good practice = default drop**, then allow specific traffic. **GCP blocks inbound traffic by default; you add allow rules.**
  - A rule has: **network**, **targets** (all instances, or instances with a **network tag** like `frontend`), **source filter** (IP ranges like `0.0.0.0/0`), **protocols/ports** (tcp:22, tcp:1234, icmp).
  - Rules are reachable two ways: the **Firewall** page (all VPCs) or **VPC → Firewall tab**.
  - **Debugging tip from the lecture:** a mistyped source range (0.0.0.0/24 instead of 0.0.0.0/0) blocked everything.
- **Terraform resources:**
  - **VPC:** `google_compute_network` (`auto_create_subnetworks = false` for custom mode).
  - **Subnet:** `google_compute_subnetwork` (`ip_cidr_range`, `region`, `network = google_compute_network.<name>.id`, where `id` is an attribute Terraform fills in after creation).
  - **Firewall:** `google_compute_firewall` (`allow { protocol = "tcp" ports = ["22","1234"] }`, `allow { protocol = "icmp" }`, `source_ranges = ["0.0.0.0/0"]`, sometimes an explicit `depends_on` the network).
  - **VM:** `google_compute_instance` (`network_interface { network = …, subnetwork = …, access_config { network_tier = "STANDARD" } }`).
  - The registry docs list **arguments** (you set them) and **attributes** (Terraform fills them in after creation). The console's "equivalent code" view also shows Terraform.
- **Testing connectivity:**
  - **netcat:** reads and writes data across TCP/UDP connections. Server `nc -l -p 1234`; client `echo "blah" | nc <ip> 1234`.
    - In the demo, port 1234 (allowed) connected; 1235 (not allowed) silently failed.
  - **socat** (socket cat): `socat <src>,opts <dst>,opts`. `-` = stdin/stdout; `tcp4-listen:8080`; `tcp4:1.2.3.4:8080`; `openssl-listen`.
    - It can also chain as a **proxy** or act as an **echo server** (`socat -v tcp-l:1234,fork exec:'/bin/cat'`).
  - Also ssh and ping.
- **Lab "Multiple VPCs with Terraform":**
  - vpc1 (sub1 10.0.1.0/24, vm1 with external IP) and vpc2 (sub2 10.0.2.0/24 with vm2 [external] and vm3; sub3 10.0.3.0/24 with vm4). Firewalls allow icmp, tcp:22, tcp:1234. Names are prefixed `tf-mod2-lab1-`.
  - **Expected reachability:**
    - VMs in the **same VPC** reach each other's **internal IPs**, even across subnets.
    - **Different VPCs can't use internal IPs** (VPCs are isolated) but can use **external IPs**.
    - VMs without an external IP have **no Internet access** (until Module 3's Cloud NAT), but you can still SSH via gcloud/IAP.
  - Submit `terraform show --json | jq > terraform_show_out.json`, run the jq queries and connectivity tests, and answer questions.

### 2.3 Software-Defined Networking: background
- **Mood when SDN emerged:**
  - Innovation was **glacial**, slowed by vendor-driven standards processes (consumers locked out, lowest-common-denominator features). The Internet felt **ossified**.
  - Networking had built **"an artifact, not a discipline"**: a bag of protocols, hard to manage, slow to evolve. New requirements (VLANs, ACLs, MPLS, ECMP, middleboxes) were added one by one, so it worked only by **mastering complexity**, not by **extracting simplicity** (abstractions).
  - Example: **OSPF's RFC is ~245 pages**. The distributed system that builds a consistent map takes **101 pages**; **Dijkstra's algorithm takes 4**.
  - Analogy: programming went from machine code to languages, OSes and abstractions.
- **Legacy switches:** a **fixed-function ASIC pipeline** plus vendor software (routing processes) plus a config interface (CLI, NETCONF).
- **4D (CCR 2005), "A clean slate 4D approach to network control and management":**
  - **Problems:** routers make **uncoordinated changes**, causing instability and thrashing. Operators must **predict and undo the effects of the control plane**, and **translate their goals into protocol knobs**. Switch software gets in the way of automation.
  - **Principles:** **network-level objectives**, **network-wide views**, and **direct control** (the ability to directly express the desired solution).
  - **Four planes:** **decision** (network-wide decisions), **dissemination**, **discovery**, **data**. Routers just forward at the behest of the decision plane.
- **Ethane (SIGCOMM 2007)** **put 4D into practice** for enterprise networks, with a **centralized controller** plus simple flow switches:
  - **Discovery:** e.g., noticing a newly plugged-in server.
  - **A policy interface (decision):** "Nick can talk to Martin over IM".
  - **Enforcement (data plane + dissemination):** **match/action tables** plus a protocol to write them.
  - Deployed at Stanford (300+ hosts). **It became OpenFlow.**
- **OpenFlow:** the key innovation was an **abstraction**, the **match-action table** plus an API/protocol for a controller to program switches.
  - **One abstraction unifies devices:**
    - **Router:** match the longest destination IP prefix → forward out a link.
    - **Switch:** match the destination MAC → forward or flood.
    - **Firewall:** match IPs and ports → permit or deny.
    - **NAT:** match IP and port → rewrite them.
  - **Its limitation:** a **fixed set of match fields** (ingress port, Ethernet src/dst/type, VLAN, MPLS, IPv4 src/dst/proto/ToS, TCP/UDP ports, ICMP type/code) and a **fixed set of actions** (forward, send to controller, drop, normal pipeline, modify fields), all **dictated by fixed ASICs**.
  - Header fields grew from 12 (OF 1.0, 2009) to 41 (OF 1.4, 2013) and still weren't enough (VXLAN, NVGRE, …).
- **RMT, "Forwarding Metamorphosis" (SIGCOMM 2013):** **Reconfigurable Match Tables**, a RISC-inspired switch-chip pipeline where tables can match **arbitrary fields** and the forwarding plane can be **changed in the field without new hardware**. They showed it's feasible at almost no extra cost or power (a 64×10G chip).
- **P4 (2014):** a language for this model.
  - **Goals:** **reconfigurability** (redefine parsing and processing in the field), **protocol independence** (not tied to specific headers), **target independence** (a compiler maps to the hardware).
  - **Abstract model:** a **programmable parser** + **match-action stages in series or parallel** + protocol-independent action primitives. **Configure** (parser, tables, control flow) vs **populate** (table entries).
  - **Programmable switch:** ingress parser → match-action stages with memory and **stateful memory** + metadata → queues/buffer → egress → deparser.
- **The mainframe analogy:** vertically integrated, closed and proprietary → slow innovation and a small industry. Open interfaces (x86, OSes, apps) → rapid innovation and a huge industry. SDN tried to do the same for routers: **merchant silicon + an open interface (OpenFlow) + many controllers + apps.**

### 2.4 Google's Andromeda (NSDI 2018): GCP's network virtualization
- **The question it answers:** how does Google implement VPCs (isolated tenant networks, subnets, policies, internal/external IPs)?
- **Goals:**
  - **Performance and isolation:** high throughput, low latency, regardless of what other tenants do.
  - **Velocity:** ship features and improvements quickly.
  - **Scalability:** big networks, many tenants, **rapid provisioning**.
- **Architecture: SDN.** Logically centralized **VM controllers** (partitioned) talk through an OpenFlow front end (extended OpenFlow) to a **virtual switch on every host**. When a VM is added, the controllers install flows from other VMs to it and from it to other VMs.
- **Problem:** programming time grows as **O(n × H)** (VMs × hosts), up to minutes for big networks (tested on 10,000 hosts, 30 controller partitions).
- **Hoverboards:**
  - **Default: every VM's traffic goes through a few Hoverboard gateways** (a slow path that holds all the rules).
  - When the controller sees a **high-bandwidth flow** (via stats), it **offloads** that flow by installing a direct rule in the host virtual switch.
  - Result: **37× faster programming** for a 40,000-VM network; scales to 100k VMs per network.
  - **Why it works:** **83% of VM pairs never communicate**, **98% have peak throughput < 20 kbps**, and today **>99.5% of traffic is offloaded**.
- **Data plane:** an **OS-bypass, busy-polling, dedicated-CPU Fast Path** (encap/decap and per-flow actions) with a flow cache.
  - Cache misses go to a slower but more complex path (Open vSwitch-based) and **coprocessor threads** (per-VM, for CPU-heavy work like DoS protection), so features grow without slowing the fast path.
  - CPU cost per packet fell across versions 1.0 → 2.2 (kernel datapath → optimized pipeline → OS bypass → removing thread hops → memory copy offload). It achieves >30 Gb/s and >3M pps with one core.
- **Velocity:**
  - **Weekly non-disruptive rollouts** via **hitless upgrades**: the old data plane transfers state to the new one in the background, then there's a brief blackout (**median 270 ms**) to transfer the remaining delta, then the new one takes over.
  - **Live migration** moves VMs between hosts transparently (for host maintenance).
- **Andromeda also does NAT for external IPs, Cloud NAT** (no NAT box in the path), and **internal network load balancing** (Module 4).

### 2.5 Azure AccelNet: SmartNICs in the public cloud (NSDI 2018)
- **VFP (Virtual Filtering Platform):** Azure's SDN data plane, a **virtual switch in Hyper-V** doing VNet address virtualization, **VIP → DIP translation** for the software load balancer, ACLs, metering and security. It's programmed by multiple SDN controllers.
  - It uses **match-action tables**, one **"layer" per policy** (VNet routing, NAT, ACLs). Inspired by OpenFlow but **multi-controller, stateful**.
- **Unified Flow Tables (the fast path):** walking every layer's tables is expensive, but it only has to happen **for the first packet of a flow**. The resulting combined action is cached (via a "header transposition engine"), so later packets need **one hash lookup**.
- **The problem:** NIC speeds went 1G (2009) → 10G → 40G → 50G (2017), with 100G coming. That's a **50× throughput increase without 50× more CPU**. Running the network stack on host cores takes CPU from VMs (cost) and adds latency and jitter.
- **Options considered:**
  - **SR-IOV** (the NIC delivers packets straight into the VM, native performance) has **no SDN policy**.
  - Silicon spectrum: **CPUs** (flexible) … GPUs … **FPGAs** … **ASICs** (efficient).
    - ASICs: a 12–18 month design cycle is too slow and inflexible.
    - Embedded CPU cores don't scale, especially on single flows.
  - **Choice: FPGA.** Programmable **gates**, highly parallel (**pipeline parallelism** = stages working on different packets; **thread-level** = multiple pipelines). Large FPGAs can even run control-plane software.
- **The Azure SmartNIC** is an FPGA as a **"bump in the wire"** between the NIC ASIC and the top-of-rack switch.
  - **The first packet of a flow is still processed by VFP in host software.** The resulting unified flow entry is then **offloaded into the FPGA**, which processes later packets at line rate.
  - Packets go to the VM via **SR-IOV**, bypassing the host OS and hypervisor.
- **Results:** higher VM-to-VM throughput, lower latency, and **much lower tail latency** than other clouds (in their comparison).
  - **Tail latency** = the slow outliers (e.g., 99th percentile), which hurt application behavior.
  - **Conclusion: "Software or hardware? Each has its place."** Software for complexity and agility, hardware for speed.

> **Memorize — Module 2**
> - Region (location) ⊃ zones (**fault isolation**; separate power, cooling, network). **VPC = your private cloud on shared infrastructure** (global in GCP), subnets regional, routes + firewall rules.
> - GCP firewall: **default deny inbound**, add allow rules (targets/tags, source ranges, protocols/ports). Test with **nc, socat, ssh, ping**.
> - Terraform: `google_compute_network`, `google_compute_subnetwork`, `google_compute_firewall`, **`google_compute_instance`** (VM).
> - **4D principles:** network-level objectives, network-wide view, direct control; planes decision/dissemination/discovery/data. **Ethane** → **OpenFlow** (match-action abstraction unifies router/switch/firewall/NAT). OpenFlow's limit = fixed fields → **RMT/P4** (reconfigurable parser + match-action).
> - **Andromeda** = Google's **network virtualization** stack: SDN controllers + host vswitch; **Hoverboards** handle the idle long tail, active flows offloaded (37× faster programming); OS-bypass fast path; hitless upgrades (270 ms), live migration.
> - **AccelNet** = Azure **FPGA SmartNIC**: VFP (vswitch) handles a flow's **first packet in software**, then the unified flow is **offloaded to the FPGA**; SR-IOV to the VM; better tail latency.

---

<a id="module-3"></a>
## Module 3 — Wide Area Network (WAN)

### 3.1 LAN vs WAN; private addresses
- **Communication beyond the VPC:** (1) **VPC to VPC**, (2) **VPC to end user** (Internet).
- **LAN** here = inside the data center (where your VPC lives; VL2-style fabrics, private IPs). **WAN** = beyond a region/data center: the public Internet, or the provider's **private backbone** (fiber between its data centers).
- **RFC 1918 private ranges** (never routed on the public Internet, not globally unique, anyone can use them internally):

| Range | Prefix |
|---|---|
| 10.0.0.0 – 10.255.255.255 | **10.0.0.0/8** |
| 172.16.0.0 – 172.31.255.255 | **172.16.0.0/12** |
| 192.168.0.0 – 192.168.255.255 | **192.168.0.0/16** |

  - Watch the 172 range: only 172.16.x.x through 172.31.x.x are private; the rest of 172.x.x.x is public.
- A VM's internal IP comes from its subnet; the external IP is mapped by Andromeda/AccelNet (public ↔ private translation on the host).
- **Public-IP traffic:** traffic to other **Google** IPs stays on **Google's backbone**. Other destinations leave through **peering with ISPs**.
- **Two mechanisms for private-IP communication:** **VPC Network Peering** (VPC↔VPC) and **Cloud NAT** (private VM → Internet).

### 3.2 VPC Network Peering
- **Without peering:** two VPCs (even yours, in different regions) **can't reach each other's private IPs**. Pings fail. A VM in another subnet of the *same* VPC works.
- **VPC peering** creates a **private connection between two VPCs** so their VMs talk over **private (internal) IPs**.
  - **You must create one peering in each direction** (vpc1→vpc2 and vpc2→vpc1). It's inactive until both exist.
  - It **exchanges routes**: each VPC's subnet routes appear in the other's route table (optionally custom routes and public-IP subnet routes too). Subnets added later get advertised automatically.
  - **Subnets can't overlap** (e.g., both using 10.0.0.0/24).
  - **Not transitive:** with A↔B and B↔C, **A doesn't get C's routes**. B can't act as a transit hub, so peer A↔C directly if needed.
  - **Firewall rules still apply** to peered traffic (it still crosses VPC boundaries).
- **Terraform** (two resources, one per direction):
  ```hcl
  resource "google_compute_network_peering" "peering1" {
    name         = "peering1"
    network      = google_compute_network.vpc1.self_link
    peer_network = google_compute_network.vpc2.self_link
  }
  resource "google_compute_network_peering" "peering2" {
    name         = "peering2"
    network      = google_compute_network.vpc2.self_link
    peer_network = google_compute_network.vpc1.self_link
  }
  ```
  - A subnetwork can also have a `secondary_ip_range`.

### 3.3 Cloud NAT
- **Problem:** a VM with **no external IP can't reach the Internet** (e.g., `ping google.com` fails), and that's not a firewall issue.
- **NAT review:**
  - **Use case 1 (static 1:1):** a VM with both internal and external IPs. Google SNATs outbound and DNATs inbound.
  - **Use case 2 (many-to-one with port pools):** like a home router, many private hosts share a public IP using different source ports. Linux equivalent: `iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE` plus a FORWARD rule allowing ESTABLISHED,RELATED traffic.
- **Cloud NAT** = Google's service giving **VMs without external IPs outbound access** to external networks (use case 2, for a whole VPC/region).
  - **There's no NAT proxy in the data path.** Andromeda on each host performs the translation using ports assigned to that VM.
  - It's **shared by all VMs/subnets** in that VPC region (as configured).
- **Setup:**
  - Cloud NAT → gateway name, network, region, type **public**.
  - **It requires a Cloud Router** (create one inline).
  - Source = VM instances / subnet ranges; NAT IPs automatic or manual.
  - Terraform: `google_compute_router` + `google_compute_router_nat`.

### 3.4 SWAN: Microsoft's Software-driven WAN (SIGCOMM 2013)
- **Context:** VPC1 and VPC2 in different regions look like they're joined by a line, but in between is Microsoft's **global backbone**.
  - The inter-DC WAN is **critical and expensive** (leased/bought fiber, subsea cables, equipment), yet with standard routers it runs **highly inefficiently**.
- **Two causes of inefficiency:**
  1. **Lack of coordination** among services.
     - Peak-to-mean traffic was **2.17**. Much of it is **background** traffic (e.g., database backups) that could be **scheduled** into the valleys, keeping the peak flat while interactive traffic goes first.
  2. **Local, greedy resource allocation** (e.g., MPLS-TE style).
     - Each router/flow grabs the shortest available path as it arrives, so later flows get bad paths. A **globally optimal** allocation (e.g., moving the blue flow to an equal-cost alternative) fits everything better.
- **Goals:**
  - A **highly efficient WAN**.
  - **Flexible sharing policies:** strict priority classes, max-min fairness within a class.
- **Key design elements:**
  - **Coordinate the sending rates of services:** hosts report demand to a **service broker** and get **rate-limited** to their allocation (on the hosts, via the vswitch).
  - **Centralized resource allocation:** an SDN controller collects topology and traffic and directly configures switches. **SDN** = streamlined switches, a centralized off-board control plane, direct data-plane configuration (vs legacy indirect configuration through routing protocols).
- **Design challenges and solutions:**
  - **Scalably computing allocations:** a path-constrained **multi-commodity flow** problem. Allocate higher priority first; be fair within a class; solve at **data-center granularity**, then split among services.
  - **Avoiding congestion during network updates:** switches don't update simultaneously, so a naive "one-shot" change can briefly overload links. SWAN uses a little **scratch capacity** on links to make updates in **provably congestion-free steps** (e.g., move half of flow A first).
  - **Limited switch memory:** use **tunnel-based forwarding** and install only the **working set** of tunnels, updated using scratch table space. (Compare Andromeda's Hoverboards.)
- **Results:** near-optimal throughput, **~60% more traffic than current practice (MPLS-TE)**, and congestion-controlled updates.
- **Lessons:** high efficiency is key to cost-effective cloud. Owning the infrastructure is a chance to be **clean slate** (no legacy protocol constraints inside the WAN).
- **Google's equivalent: B4 (SIGCOMM 2013).** A private SDN WAN connecting Google's data centers with **OpenFlow-controlled merchant-silicon switches** and centralized traffic engineering that drives links to **near 100% utilization**, splitting flows across paths by priority.

### 3.5 Espresso: Google's SDN peering edge (SIGCOMM 2017)
- **Context:** traffic leaving Google for the **public Internet** (VPC → end users), terabits per second to Internet peers (video, cloud).
  - Google's layers: **Jupiter** (data center fabric, SIGCOMM 2015) → **B4** (inter-DC WAN) → **metro points of presence (PoPs), >100 worldwide** → **Espresso** at the peering edge → ISPs → users.
- **Problems with traditional peering routers:**
  1. **Optimizing per customer/application** (best video quality, differentiated cloud service) is constrained by **BGP's shortest path and lack of application awareness**. A better path may exist.
  2. **Feature velocity:** the vendor cycle (request → commit → implement → vendor test → Google integration test → deploy) takes **years**.
  3. The belief that "SDN only suits walled gardens": **peering requires interoperating with many heterogeneous external peers**.
- **Design principles:**
  1. **Hierarchical control plane:** global optimization plus local control for fast reaction.
  2. **Fail static:** the local control plane keeps working if the global controller fails.
  3. **Software programmability:** externalize features into software on **commodity servers** for scale.
  4. Testability. 5. Manageability.
- **Architecture:**
  - The big, expensive traditional peering router (Internet-size RIB/FIB, big TCAM/ACLs) is replaced by a **simple label-switched peering fabric**.
  - **BGP speakers run as software on servers**, peering with the external peers.
  - **Host-based packet processors** on servers do ACLs, DoS handling and routing decisions. Labels on packets specify the egress port.
  - A **global controller** plus local and peering-fabric controllers.
  - **BGP is the only legacy protocol needed**, and only at the external boundary. Everything inside is software/SDN.
- **Application-aware routing:**
  - Application signals feed the global controller, which can send traffic out a **different PoP/ISP than BGP's best path** (e.g., if an ISP is congested).
  - Serves **13% more traffic** than BGP best path and **improves user experience** (longer time between rebuffers, higher goodput).
- **Velocity:** releases every **~1–2 weeks**, **>50× more often** than traditional routers; a new L2 VPN feature shipped 6× faster.
- **Cost:** about **75% cheaper**. Serves over **22%** of Google's Internet traffic.
- **Conclusion:** peering can evolve to exploit SDN, and **SDN's value is flexibility and feature velocity**.
- **Related: EdgePlex (AT&T, SOSR 2015)** decomposes the **provider edge router** with SDN/NFV, using **one VM sandbox per customer** for isolation and mobility.

### 3.6 WAN lab
- **Build in Terraform** (prefix `tf-mod3-lab1-`): two VPCs with one VM each (172.16.0.0/24 and 172.16.1.0/24, **no external IPs**), **VPC peering** both ways, and **Cloud NAT + Cloud Router** on vpc2.
- **Goal:** send vm1's Internet traffic through vm2.
- **Why `ip route add <dst> via <vm2-ip>` doesn't work:** VMs aren't L2-adjacent. The VPC's gateway is a Google router, so ARPing for vm2 gets no answer.
- **Fix: a VXLAN tunnel (an L2 frame inside UDP/IP, port 50000).**
  ```
  vm1: ip link add vxlan0 type vxlan id 5001 local 172.16.0.2 remote 172.16.1.2 dev ens4 dstport 50000
       ip addr add 192.168.100.2/24 dev vxlan0 ; ip link set up dev vxlan0
  vm2: (same with local/remote swapped) ip addr add 192.168.100.3/24 dev vxlan0
  vm2: sysctl -w net.ipv4.ip_forward=1
       iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -o ens4 -j MASQUERADE
  vm1: ip route add 34.223.124.45/32 via 192.168.100.3    # neverssl.com
       wget 34.223.124.45
  ```
  - **The firewall must also allow the VXLAN traffic (UDP 50000) between the VPCs.** Check with `tcpdump -i ens4 -n port not ssh` and `tcpdump -i vxlan0`.
- **Assessment:** `terraform show --json`, then jq queries on the peering resources (`.values.state`, `.address`, `.values.network`, `.values.peer`), plus connectivity outputs.

> **Memorize — Module 3**
> - RFC 1918: **10/8, 172.16/12 (172.16–172.31), 192.168/16**.
> - **VPC peering:** private-IP connectivity between VPCs; **create both directions**; **exchanges routes**; **no overlapping subnets**; **not transitive**; firewall still applies. Terraform `google_compute_network_peering` ×2.
> - **Cloud NAT:** outbound Internet for **VMs without external IPs**; **needs a Cloud Router**; implemented **in Andromeda on each host** (no proxy box).
> - **SWAN:** inefficiency from **lack of coordination + local greedy allocation** → coordinate service rates + centralized allocation (SDN); congestion-free updates with scratch capacity; working-set tunnels; ~60% more traffic. **B4** = Google's SDN WAN (~100% utilization).
> - **Espresso:** Google's SDN **peering edge**; **BGP** speakers in software + label-switched fabric + host packet processing; hierarchical control, **fail static**; application-aware routing (+13% traffic); >50× faster releases; 75% cheaper.

---

<a id="module-4"></a>
## Module 4 — Network Services

GCP's network services covered so far: VPC, firewall rules, Cloud NAT. This module adds **load balancing, Cloud CDN, and premium/standard tiers**; Module 5 adds connectivity services.

### 4.1 Load balancing concepts
- A **load balancer** is a service's **entry point** and directs each request to one of N servers, for **scaling** (add servers) and **resilience** (stop sending to failed ones).
- **Key property 1, the algorithm:** e.g., round robin, least connections.
- **Key property 2, the server set:** which servers are eligible, plus **liveness/health checks** (heartbeats; no reply → mark down).
- **Application LB (L7) vs Network LB (L4):**

| | **Network LB (Layer 4)** | **Application LB (Layer 7)** |
|---|---|---|
| Looks at | IP + TCP/UDP headers | Also the HTTP request (URL, headers) |
| Traffic | Can **pass through** (no connection termination) | A **proxy** that **terminates the TCP connection** |
| Routing | Balance load, keep **flow affinity** (same connection → same server) | Can also route **URLs to specific servers** (`/signup` → server 1, `/purchase` → servers 2–3) |
| GCP system | **Maglev** (external); **Andromeda** (internal) | **Envoy** proxies |

- **Internal vs external:**
  - **External** = has a public IP, used for Internet clients (e.g., the web tier).
  - **Internal** = between services inside Google. **An internal network LB needs no middlebox: forwarding rules are handled in Andromeda on each host** (like Cloud NAT).
- **Global vs regional:** are the backends in one region or several?

### 4.2 Premium vs Standard tier, and anycast
- **Standard tier egress:** traffic **leaves Google's network close to the source** (near the region), then crosses the public Internet ("hot potato": get it off my network fast).
- **Premium tier egress: "cold potato."** Traffic rides **Google's backbone** and **exits close to the destination** (the user), keeping the public-Internet portion short. Better performance because Google controls and optimizes its own backbone (Espresso).
- **Premium tier ingress / global load balancing:** the **same global external IP address is advertised from many PoPs (anycast via BGP)**. Each client's ISP routes to the **nearest Google Front End (GFE)**.
  - BGP review: AS4 announces prefix D; neighbors prepend themselves and re-announce; each AS picks (roughly) the shortest AS path. The same prefix announced from several places gives each network a "closest" site.
  - **This is how global load balancing works with a single IP.**

### 4.3 GCP load balancer walkthrough (application LB)
- **Setup:** vpc1/subnet1 (10.0.1.0/24) with vm1 and vm2 running a simple **Flask** app on port 5000 (`/health` and `/hello` → returns the hostname). The firewall allows tcp 22, **5000**, icmp.
- **Steps:**
  1. **Instance group:** **unmanaged** (static list of your existing VMs) vs **managed** (from a template, supports **autoscaling**). Add a **named port** (`app-http: 5000`).
  2. Create the load balancer: **Application LB**, **public-facing**, **regional** (or global).
  3. A **proxy-only subnet is required** (e.g., 10.0.3.0/24); Google launches the Envoy proxies in it.
  4. **Frontend:** protocol HTTP, port 80, ephemeral (or static) IP.
  5. **Backend service:** type instance group, named port, capacity settings.
  6. **Health check:** HTTP, port 5000, path `/health`, interval and thresholds. Backends show unhealthy until the app is running.
  7. Optional **routing rules** (URL → backend). Review and create (about 30 s).
- **Test:** `curl http://<LB-IP>/hello` alternates between vm1 and vm2. Backends don't need external IPs.
- Terraform needs several resources (instance group, backend service, health check, URL map, proxy, forwarding rule); start from Google's examples.

### 4.4 Autoscaling and the metadata server
- **Autoscaling** monitors usage and **creates or kills instances** as needed, and updates the load balancer's server set automatically.
  - **Threshold-based:** e.g., add a VM above 50% CPU, remove one below 25%.
  - **Predictive:** ML on past load (time of day, recent trend) to scale ahead of demand. See **Autopilot (EuroSys 2020)**, Google's ML-based horizontal and vertical autoscaler: it cut resource slack from 46% to **23%** and cut out-of-memory incidents by 10×.
- **GCP steps:**
  - Build a VM with the software installed → **stop it** → **create an image** from its disk.
  - Create an **instance template** (machine type, image, configuration; a VM spec that isn't launched).
  - Create a **managed instance group** with autoscaling: on/off or scale-out only, min/max instances (cap your bill), metric (CPU, others, or predictive), initialization period (boot time), health checks.
- **Metadata server: `169.254.169.254`**, an address from the **IPv4 link-local range 169.254.0.0/16**.
  - Link-local is **never routed**; it's local to the current network only (compare the private 10/8 ranges, which can be routed internally).
  - It serves **instance metadata** (key-value pairs like the **startup script**) over HTTP (`curl`/`wget`), plus **DNS and NTP**.
  - Startup scripts read it at boot, letting each autoscaled instance configure itself.

### 4.5 Google's Maglev (NSDI 2016): software network load balancer
- **Consistent hashing background** (from Akamai's founders, for content distribution):
  - `server = hash(x) mod k` spreads load, but **when k changes (failure or scaling), almost every key remaps**.
  - **Consistent hashing:** servers (buckets) and keys hash onto a **ring**; a key goes to the **next bucket clockwise**.
    - **Balanced:** no bucket gets a disproportionate share.
    - **Smooth:** adding or removing a bucket only moves keys from its neighbors.
  - Problem: when a node fails, its successor gets double the load. Fix: **virtual nodes** (each server owns v random points).
- **What Google wanted from a load balancer:** **even** balancing, **reliability** (TCP connections don't reset), **flexibility** (iterate fast), **scalability**, **efficiency**.
  - **Hardware appliances** (e.g., F5) had poor flexibility (slow vendor cycles), were hard and expensive to scale, and had poor failure handling.
- **Maglev:**
  - A **distributed software system on commodity Linux servers**. Capacity changes by adding or removing servers.
  - **Scale out:** routers spread packets across Maglevs with **ECMP**.
  - **Scale up:** **kernel bypass**, so **one machine saturates 10 Gbps with small packets**.
  - Return traffic **bypasses Maglev** (direct server return). In service since **2008**; also provides GCP's network load balancing.
- **The reliability challenge:** a connection must always reach the same backend, even when **(a) the set of Maglevs changes** (failure or scaling makes ECMP re-hash flows to a *different* Maglev) **and/or (b) the set of backends changes**.
  - **Connection tracking alone fails** when ECMP moves a flow to a Maglev that has no state for it.
  - **Plain hashing alone fails** when the backend set changes.
  - **Both at once** breaks both.
- **Solution: connection tracking + consistent hashing.**
  - Each Maglev first checks its **connection table** (a fast hit).
  - On a miss, it uses **Maglev hashing** (Google's own consistent hashing algorithm, with better **evenness** than earlier ones). **Every Maglev picks the same backend for the same 5-tuple without sharing state**, and backend changes disturb few existing connections.
  - Then it **encapsulates** the packet to the backend.
  - **Operational wins:** rolling restarts/upgrades without draining; little impact from flapping backends.

### 4.6 Content Distribution Networks (CDN)
- **Serving everything from a single origin server** gives a single point of failure, easy overload, long latency, and lots of peering traffic.
- **A CDN provider** runs **servers at the network edge** with many peering agreements.
  - The **content provider** signs up and names its **origin server**. Clients are directed to a **nearby CDN edge server**.
  - **First access:** the edge server fetches from the origin and **caches** the content. **Later accesses** are served from the cache: little network traffic and little origin load.
- **Google Cloud CDN:**
  - Create an origin pointing at a **backend bucket** (Cloud Storage), a **backend service** (behind a load balancer), or a **custom origin** (any host:port).
  - Attach it to a load balancer and configure caching. Google's global network does the rest.

### 4.7 Akamai's network (Operating Systems Review 2010)
- **Performance = business:**
  - In an A/B test, users who went through Akamai were **15% more likely to complete a purchase** and **9% less likely to abandon after one page**.
  - Customers reported **$200K to $3M+** in annual revenue gains from better performance and reliability.
- **"The problem with the Internet":**
  - Any single network's reach is limited.
  - Peering point congestion.
  - Inefficient routing protocols.
  - Unreliable networks.
  - Inefficient communication protocols.
  - Backend scalability.
  - Application limitations / slow adoption of change.
  - Distance hurts: longer RTT → more loss → lower throughput.
- **Scale (2010):** **61,000 servers, ~1,000 networks, 70 countries.** Edge servers plus a **hierarchy of parent servers**, plus monitoring and management.
- **Use cases:**
  - **Content delivery:** stayed lossless through a multi-day **cable cut** that crippled the origin.
  - **Live video streaming:** fan-out through parents to the edges, like multicast.
  - **Application delivery, even for non-cacheable content:**
    - **Path optimization** via **Resilient Overlay Networks (RON, SOSP 2001):** route around broken Internet paths through other overlay nodes. RON detects and recovers in **<20 seconds** vs minutes for BGP.
    - **Packet loss reduction** with **forward error correction**.
    - **Transport optimizations** (pooled persistent connections, smarter retransmission).
    - **Application optimization** (parse and **prefetch** embedded content).
- **How clients find a nearby server: DNS-based load balancing.**
  - Content is named under akamai.net. Akamai's DNS servers do a **geo-IP lookup** on the requester (its resolver) and return the IP of a **nearby** edge server.
  - Contrast with Google premium tier's **anycast**.

### 4.8 Suggested lab (ungraded)
- Build a web service: backend Flask servers behind an **internal network LB** (Andromeda), frontend Flask servers behind an **external application LB**.
- Start **unmanaged/regional**, then make the frontend **autoscaled (managed)**, then go **global across two regions**. Load-test it.

> **Memorize — Module 4**
> - LB = algorithm + server set (health checks). **ALB = L7 proxy (terminates TCP, URL routing, Envoy)**; **NLB = L4 pass-through (Maglev external, Andromeda internal)**.
> - ALB in GCP needs a **proxy-only subnet** (for Envoy), frontend, backend service, **health check**, instance group (+ named port).
> - **Standard tier** exits near the source (hot potato); **Premium** rides Google's backbone and exits near the destination (cold potato); premium ingress = **same IP via anycast from many PoPs**.
> - Autoscaling: threshold or predictive (**Autopilot**); image → instance template → managed instance group. **Metadata server 169.254.169.254** (link-local).
> - **Maglev:** software NLB on commodity servers; ECMP across Maglevs; **consistent hashing + connection tracking** keep existing connections on the same backend through Maglev/backend changes; kernel bypass, 10G/machine.
> - **CDN:** edge caches; first access pulls from origin. **Akamai:** 15% more purchases; hierarchy; RON path optimization, FEC, prefetch; **DNS-based** server selection.

---

<a id="module-5"></a>
## Module 5 — Hybrid Networks

### 5.1 Why hybrid?
- **Branch offices** need cloud data (employees query GCP from the office).
- **Colocation facilities** hold existing infrastructure that wants to expand into the cloud.
- **Multi-cloud** (e.g., use OpenAI's infrastructure in **Azure** alongside GCP).
- Choices trade off **security, reliability, performance (latency/throughput), cost, development effort**.
- **Three approaches:**
  - **VPN** (from branch offices, or anywhere).
  - **Direct connections / Interconnect** (from colocation).
  - **Cross-cloud interconnect.**

### 5.2 VPN (IPsec)
- **Why:** company sites (or an office and the cloud) share data across the Internet, and eavesdroppers could read it.
  - Ways to eavesdrop: **Wi-Fi is broadcast**; a provider compelled by a government, a rogue employee, or a compromised router; a routing misconfiguration; deliberate **route hijacking** (BGP has little security).
- **IPsec** creates an **encrypted tunnel** between endpoints (a whole site or one machine).
  - **ISAKMP/IKE** negotiates parameters (SA-1), does the **Diffie-Hellman key exchange**, authenticates, sets SA-2, then traffic flows encrypted.
- **GCP setup (HA VPN):**
  1. **Create a VPN gateway** (choose **HA = two tunnels**, or Classic = one tunnel). Pick network and region; Google allocates its interface IPs.
  2. **Add VPN tunnels:** create a **peer VPN gateway** object with the **public IP(s) of your on-prem device** (often the firewall).
  3. **Create a Cloud Router** (pick a private **ASN**, e.g., 64xxx; **advertise all subnets** visible to it). It exchanges routes **dynamically** with on-prem.
  4. Set IKE/IPsec parameters.
  5. **Configure the BGP session(s):** the peer router's IP and **ASN**.
  6. **Configure the on-prem side:** router/firewall with IPsec + **BGP** (Google provides downloadable configs for common platforms).
  - Then on-prem servers and VMs reach each other over private IPs through the tunnel.
- **Not limited to offices:** you can VPN to **AWS or Azure** VPN gateways (use the peer IPs they assign). Downsides vs private links: **lower bandwidth, higher latency, dependent on the Internet (no SLA)**.

### 5.3 Direct Connect / Cloud Interconnect (from colocation)
- **Colocation facility** (e.g., **Equinix, Digital Realty, CoreSite**):
  - A provider-owned building in a popular area where customers put their own equipment.
  - It provides **space, power, network connectivity, physical security**.
  - **Cage:** each company's **own secured cage**.
  - **Meet-me room:** where **network providers' equipment** lives.
  - **Cross connect:** a (paid) cable from your cage to a provider in the meet-me room.
  - **Virtual cross connects / fabric:** colo providers are moving to an SDN-style **network fabric** your cage connects to once, with connections then set up in software.
- **Cloud direct connect:**
  - **In some facilities, cloud providers have network equipment**, so you can get a **port on Google's physical switch** that connects you **directly to Google Cloud** (not over the public Internet).
  - **Partners** with global fabrics (**Megaport, Console Connect, Equinix**) automate connections and reach beyond facilities where Google has a switch.
  - GCP calls these **Dedicated Interconnect** (straight to Google) and **Partner Interconnect** (via a partner).
- **GCP setup:** after ordering the cross connect and interconnect port, create a **Cloud Router** and a **VLAN attachment** (Interconnect wizard), then run **BGP** with your router in the cage.

### 5.4 Inter-cloud (cross-cloud interconnect)
- **A VPN between clouds works**, but has lower bandwidth, higher latency, and no SLA.
- **A private alternative:** a direct connect from each cloud to a provider's fabric, plus **a router in between**.
- **Cross-Cloud Interconnect** = effectively a **direct connect to each cloud plus a virtual router** that the provider (a **partner like Megaport or Console Connect, or Google itself**) runs. It can also span regions over the fabric.
- You still configure the other cloud's side (its router equivalent, VLAN attachment, BGP).
- Not cheap (hundreds of dollars to try), but the setup is the same pattern: **VLAN attachments + Cloud Router + BGP**.

> **Memorize — Module 5**
> - **VPN** (IPsec over the Internet): **VPN gateway (HA = 2 tunnels) + peer gateway (on-prem IPs) + Cloud Router (ASN, advertise subnets) + BGP sessions + on-prem router/firewall config**. Works to AWS/Azure too; lower bandwidth, no SLA.
> - **Colo** = space, power, connectivity, security; **cage**, **meet-me room**, **cross connect**, fabric.
> - **Direct connect / Interconnect** = a port on Google's switch inside a colo (Dedicated) or a connection via a partner fabric (Megaport, Console Connect, Equinix: Partner). Needs a **Cloud Router + VLAN attachment + BGP**.
> - **Cross-cloud interconnect** = direct connect to both clouds + a **provider-run virtual router** (Google or partners like Megaport).
> - **Cloud Router shows up everywhere:** Cloud NAT, VPN, Interconnect.

---

<a id="quick-reference"></a>
## Paper Cheat Table and Quick Reference

### Papers

| Paper (venue) | Company | Problem | Key idea | Headline result |
|---|---|---|---|---|
| **Above the Clouds** (Berkeley TR 2009) | — | What is cloud computing? | No up-front commitment, pay-per-use, illusion of infinite resources | 5–7× economies of scale |
| **VL2** (SIGCOMM 2009) | Microsoft (Azure) | Fragmented, oversubscribed, expensive DC networks | Flat addressing + directory; Clos; Valiant LB (random indirection via ECMP + anycast) | Agility at scale with commodity switches |
| **4D** (CCR 2005) | Academia | Control plane complexity, indirect control | Network-level objectives, network-wide views, direct control; decision/dissemination/discovery/data planes | Blueprint for SDN |
| **Ethane** (SIGCOMM 2007) | Stanford | Enterprise management/security | Central controller + flow switches enforcing network-wide policy | Became OpenFlow |
| **RMT** (SIGCOMM 2013) | Academia/industry | OpenFlow's fixed fields/actions | Reconfigurable match tables in switch ASICs | Flexible at ~no extra cost → P4 |
| **Andromeda** (NSDI 2018) | Google | Network virtualization (VPCs) at scale | SDN controllers + host vswitch; **Hoverboards**; OS-bypass fast path; hitless upgrades | 37× faster programming; >99.5% offloaded; 270 ms blackout |
| **AccelNet** (NSDI 2018) | Microsoft | NIC speeds outpacing CPUs | **FPGA SmartNIC** offloads VFP's unified flows; first packet in software; SR-IOV | Better throughput, latency, tail latency |
| **SWAN** (SIGCOMM 2013) | Microsoft | Inefficient inter-DC WAN | Coordinate service rates + centralized allocation; congestion-free updates; working-set tunnels | ~60% more traffic, near optimal |
| **B4** (SIGCOMM 2013) | Google | Inter-DC WAN | OpenFlow + merchant silicon + central TE | Near 100% link utilization |
| **Espresso** (SIGCOMM 2017) | Google | Peering edge: BGP limits, vendor velocity | Software BGP speakers + label-switched fabric + host packet processing; hierarchical control; fail static | +13% traffic, >50× release velocity, 75% cheaper |
| **EdgePlex** (SOSR 2015) | AT&T | Monolithic provider edge routers | Decompose PE with SDN/NFV; VM per customer | Isolation, mobility, line-rate |
| **Maglev** (NSDI 2016) | Google | Hardware LBs inflexible and hard to scale | Software LB on servers; ECMP; consistent hashing + connection tracking; kernel bypass | 10 Gbps/machine; in service since 2008 |
| **Akamai** (OSR 2010) | Akamai | Internet not built for app performance | Edge + parent hierarchy, DNS-based mapping, RON, FEC, prefetch | 61K servers; +15% purchases |
| **Autopilot** (EuroSys 2020) | Google | Over-provisioned resource limits | ML-based horizontal + vertical autoscaling | Slack 46% → 23%; 10× fewer OOMs |
| **RON** (SOSP 2001) | MIT | Slow BGP recovery from outages | Application-layer overlay routes around failures | Recovers in <20 s |

### GCP building blocks

| Need | Service / resource | Terraform type |
|---|---|---|
| Private network | VPC (+ subnets) | `google_compute_network`, `google_compute_subnetwork` |
| Allow traffic | Firewall rule | `google_compute_firewall` |
| VM | Compute Engine instance | `google_compute_instance` |
| VPC ↔ VPC private | VPC Network Peering (×2) | `google_compute_network_peering` |
| Private VM → Internet | Cloud NAT + Cloud Router | `google_compute_router_nat`, `google_compute_router` |
| Spread load | Application LB (Envoy) / Network LB (Maglev/Andromeda) | several resources |
| Scale automatically | Managed instance group + template + autoscaler | — |
| Cache content | Cloud CDN | — |
| Office ↔ cloud | HA VPN + Cloud Router (BGP) | — |
| Colo ↔ cloud | Dedicated / Partner Interconnect + VLAN attachment + Cloud Router | — |
| Cloud ↔ cloud | Cross-Cloud Interconnect (or VPN) | — |

### Addresses to know

| Address | Meaning |
|---|---|
| 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | RFC 1918 private |
| 169.254.0.0/16 (169.254.169.254) | Link-local (GCP metadata server) |
| 0.0.0.0/0 | Any address (firewall source "anywhere") |

---

<a id="quiz-map"></a>
## Graded Quiz Coverage Map

These quizzes are graded (8% each), so under the course's Conditional AI policy this guide does **not** answer them. Use this map to find which sections to review before you take each one.

| Quiz | Topics it draws on | Review |
|---|---|---|
| **Cloud Background Quiz** | Berkeley characteristics, what virtualization solves, VL2 properties and the role of randomness in VLB, Terraform's style, Terraform-on-GCP steps | §1.2–1.6 |
| **VPC Quiz** | Zones, VPC components, Linux connection-testing tools, 4D principles, the match-action abstraction, Andromeda, AccelNet's packet path, the Terraform VM resource | §2.1–2.5 |
| **WAN Quiz** | RFC 1918 ranges, internal-IP reachability, VPC peering properties, Cloud NAT, SWAN's inefficiency causes, Espresso's protocol | §3.1–3.5 |
| **Network Services Quiz** | ALB vs NLB, premium vs standard tier, the proxy-only subnet, Maglev consistent hashing, Akamai's server selection | §4.1–4.7 |
| **Hybrid Quiz** | Which hybrid technology fits which scenario, VPN setup steps, where direct connects happen, cross-cloud providers | §5.1–5.4 |

Once a quiz has been graded, Coursera shows you the feedback. Comparing it with these sections is a good way to find weak spots before the final. You also get **two attempts** at this final, so after the first attempt, review the sections behind any questions you weren't sure of.
