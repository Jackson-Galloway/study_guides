# ECEA 5371 — Network Principles in Practice: Linux Networking
## Final Exam Study Guide

*Built from the course lecture transcripts, slides and readings (instructor: Eric Keller, CU Boulder MS-ECE). This course puts the 5370 concepts into practice with Linux.*

**Final exam format** (from "About the Final Exam"): **unproctored**, **2 hours**, **22 questions**, **1 attempt / 1 submission**. Multiple-choice questions plus **one free-response question**; no programming questions. Worth **20%** of the grade (labs 18–18.5% each; the five summary quizzes 5% each).
**Not allowed:** notes or external websites. There's no notes sheet, so the "Memorize" boxes below are things to know cold.

> **AI policy for this course: Conditional AI.** AI may be used to summarize material and generate practice questions (which is what this guide and the practice exam are). It may **not** be used to answer graded quizzes or labs, or to enter their questions into an AI tool. That's why there's no quiz answer key here. The [quiz coverage map](#quiz-map) at the end points to the sections each graded summary quiz draws on, without answering anything.

> The exam is only 22 questions, so expect them to lean on **command syntax and what a command does** (`ip link`, `ip addr`, `ip route … via`, `iptables -t nat …`, `ipvsadm`, `tc qdisc …`, `ip netns`, `kubectl`), plus the concepts behind them.

---

## Table of Contents
1. [Module 1 — Introduction to Linux Networking (L2, devices, tools)](#module-1)
2. [Module 2 — IP Layer with Linux (addresses, routes, BIRD/BGP)](#module-2)
3. [Module 3 — Creating a Gateway (iptables, IPVS, tc)](#module-3)
4. [Module 4 — Virtual Networking (namespaces, veth, Docker)](#module-4)
5. [Module 5 — Kubernetes Networking](#module-5)
6. [Command Cheat Tables](#quick-reference)
7. [Graded Quiz Coverage Map](#quiz-map)

---

<a id="module-1"></a>
## Module 1 — Introduction to Linux Networking

### 1.1 Why Linux?
- **Network appliances** come in many forms (office boxes to core routers, plus virtual versions), from many vendors (Cisco, Palo Alto, Juniper, Arista), and for many uses (data centers, core routers, wireless gateways, perimeter firewalls, cloud routers).
- **The three planes:**

| Plane | What it does | Examples |
|---|---|---|
| **Data plane** | Processes packets; must be **fast** | NAT, forwarding, filtering |
| **Control plane** | Runs protocols that decide how the data plane processes traffic | BGP, OSPF, DHCP |
| **Management plane** | How a user or automation **configures and monitors** the appliance | CLI, Terraform (infrastructure as code), SNMP |

- **Motivation 1: Linux covers the whole stack.** Data plane = **Linux kernel**; control plane = the **Linux ecosystem** (routing daemons); management plane = **Linux utilities**. It's open source with a big community.
- **Motivation 2: Linux as the NOS.**
  - **Legacy:** tight coupling; the vendor sells data, control and management planes as one **vertically integrated** box.
  - **Trend: disaggregation.** Separate the hardware (**white box switches**, which mostly provide the data plane), an open **network operating system (NOS)**, apps, and **open APIs**.
  - **Benefits of disaggregation: more choice and extensibility.**
  - **Example NOSes, all with Linux at the core:** **SONiC** (Microsoft-championed), **Cumulus Linux** (now NVIDIA, via Mellanox), **DENT OS** (championed by Amazon).

### 1.2 The Linux data plane
- Linux provides forwarding, NAT, filtering, bridging, load balancing and traffic shaping.
- **Generic router architecture:** a **pipeline of functions** with **tables driving the processing** (lookup → update header → queue → transmit). Protocol **type fields** guide parsing: EtherType → IPv4/IPv6/ARP/VLAN; IP protocol → ICMP/TCP/UDP/GRE/ESP.
- **Simplified Linux data plane:** driver receive → ARP / VLAN / bridge / IPv4 → local? (ICMP, TCP, IPsec) or forward → **queuing discipline** → ARP (egress) → driver transmit. Along the way are **hook points** for NAT and filtering.
- **Utilities:**
  - **iproute2:** interfaces, bridges, forwarding (`ip link`, `ip addr`, `ip route`, `ip neigh`, `ip netns`).
  - **iptables:** classification, NAT, filtering.
  - **tc:** traffic control / queuing disciplines.
  - **ipvsadm:** load balancing.
- **Netlink:** how user-space utilities **tell the kernel how to process traffic** (i.e., fill in its tables).
  - It's based on Berkeley sockets: each application is a **client**, the **servers run in the kernel**, and it supports **multicast**. Libraries (libnl) hide the details.
  - `socket(AF_NETLINK, SOCK_RAW, family)`. Families include **NETLINK_ROUTE**, **NETLINK_NETFILTER**. Multicast groups: RTMGRP_LINK, RTMGRP_IPV4_IFADDR.
  - Messages start with `struct nlmsghdr` (length, **type** such as `RTM_NEWLINK`, flags like `NLM_F_REQUEST | NLM_F_ACK`, seq, pid), followed by a payload (e.g., `struct rtmsg`).
- **Summary:** the data plane is functions plus tables in a pipeline, each with an API exposed through **Netlink sockets**; utilities provide user-friendly abstractions over those APIs.

### 1.3 Lab environment
- **Goal:** build a topology of Linux network devices and hosts, configure them, inject traffic, and troubleshoot, without buying hardware.
- **Vagrant:** creates lightweight, reproducible, portable dev environments from a **Vagrantfile** (box/Linux distribution, CPU/RAM limits, provisioning script installing net-tools, Docker, containerlab). Assumes **VirtualBox**.
  - `vagrant up`; `vagrant suspend` (save state and stop; resume later); `vagrant halt` (shut down); `vagrant status`; `vagrant ssh-config` (host 127.0.0.1, **port 2222**, user `vagrant`, private key path).
  - Connect over **SSH** (the course uses MobaXterm with X11 forwarding). Key-based: the **public key on the server, private key on the client**.
- **Bare metal → VMs → containers:**
  - **Bare metal** wastes resources (one app per server), but running many apps on one server causes conflicts (different OS needs, variable workloads, conflicting dependencies like python2 vs python3).
  - **VMs** (hypervisor: VMware, Xen, KVM, Hyper-V) isolate whole OSes, but are **heavyweight** (an extra OS and binaries) and have overhead.
  - **Containers** put isolation **inside one OS**: **namespaces** (what resources, and resource names, a process sees: file descriptors, IP addresses) and **cgroups** (group processes and allocate CPU/memory the kernel enforces).
  - History: chroot 1982 → FreeBSD Jails 2000 → Virtuozzo/OpenVZ 2000 → Linux VServer 2001 → **LXC 2008**. **Docker didn't invent containers; it made them usable** (nice packaging).
- **Docker architecture:**
  - **Client** (`docker` CLI) → **daemon** on the host (manages images and starts processes with cgroups/namespaces) → **registry** (stores images for push/pull; e.g., **Docker Hub**).
  - **Image** = an **immutable** filesystem snapshot used as a container's starting files (copy-on-write).
  - **Container** = a temporary, writable (copy-on-write) filesystem layered over an image that disappears at end of life, **plus its own network stack**, plus a process group (when the main process exits, the container exits).
- **Containerlab:** a **CLI that uses Docker containers to orchestrate container-based networking labs**. It starts containers, **builds the virtual wiring** between them from a YAML topology, and manages the lab lifecycle. It can also use vendor images (SONiC, Nokia SR Linux).
  - The YAML has a `name` (e.g., `lab1-part1`), then `topology:` with `nodes:` (kind: linux, image, `binds` to mount a host directory into the container) and `links:` (`endpoints: ["host1:eth1", "switch:eth1"]`).
  - **eth0 is reserved for containerlab's management network.**
  - `sudo containerlab deploy` / `sudo containerlab destroy`. Container names are `clab-<labname>-<node>`.
  - Run commands with `docker exec -it clab-lab1-part1-host1 <cmd>`; shells use aliases (e.g., `alias host1="docker exec -it clab-…-host1"`).

### 1.4 Troubleshooting tools
- **ping:** is a host up, and what's the RTT?
- **tshark:** command-line **network protocol analyzer** (Wireshark without the GUI, similar to tcpdump). **Captures and displays packets.** Use it to **check whether packets are arriving on an interface.**
  - `-i <iface>` (default eth0).
  - `-f "<capture filter>"` (filter **before** capture): `-f "arp"`, `-f "port 80"`, `-f "host 1.2.3.4 and port 80"`.
  - `-Y "<display filter>"` (capture everything, show only matches): `-Y "tcp.port == 80"`, `-Y "ip.src == …"`, `-Y "ip.ttl > 10"`.
  - `-r <file>` reads a pcap; `-w`/`-o` writes one. `-T fields -e eth -e vlan -e vxlan` prints selected fields.
- **pcap** = a **file format storing a set of captured packets**.
- **Wireshark:** graphical; usually run outside the container (on a veth) or open a pcap that tshark wrote via a bind-mounted directory.
- **Scapy:** a Python library/interactive tool for **crafting and modifying packets**, reading/writing pcaps, and sending/capturing traffic.
  - Build a header: `Ether(dst="…", src="…", type=0x0800)`, or set fields one at a time (`eth.dst = …`).
  - Stack headers with `/`: `pkt = Ether()/IP()/TCP()`. Access a layer with `pkt[IP].src = "1.2.3.4"`.
  - `ls()` lists protocols; `ls(IP)` lists fields. `rdpcap()`/`wrpcap()` read and write pcaps.
  - `sniff(count=3)` captures; **`sendp()` sends at layer 2**, **`send()` at layer 3**; `sr()`/`sr1()` send and receive.
  - In the bridge demo, Scapy (`onepkt.py`) **crafted a single packet with a given source and destination MAC** (plus IP/TCP and a message) and sent it out an interface.

### 1.5 Network device configuration: `ip link`
- **Device types:**
  - **Physical:** Ethernet.
  - **Attached to another device:** **VLAN**.
  - **Join several devices together:** **bridge**, **bond**.
  - **Tunnels:** **VXLAN**, **GENEVE**, **GRE**.
  - **Virtual:** **veth**.
  - `ip link add` can create all of these.
- **iproute2 vs net-tools:** iproute2 (`ip …`, `tc`) is the modern suite; net-tools is the older one (`ifconfig`, `arp`, `route`). The course uses **iproute2**.
- `ip link` subcommands: **show, set, add, delete**.
  - `ip link show [dev] [up] [type TYPE] [master DEV]`. Options go before `link`: `ip -d -j -p link show eth0` (details, JSON, pretty).
  - `ip link set dev eth0 address 22:33:22:44:55:44`; `ip link set dev eth0 up`.
  - `ip link delete dev eth0.2`.
- **Bridge** = a device that implements a **learning switch** (layer 2, MAC learning, flooding unknowns):
  ```
  ip link add name mybridge type bridge
  ip link set mybridge up                # new devices start DOWN
  ip link set eth1 master mybridge       # enslave ports to the bridge
  ip link set eth2 master mybridge
  ```
  - Before the bridge exists, frames reach the switch container's eth1 but go nowhere. After, they're flooded to eth2.
- **VLAN** = adds a **VLAN tag** (an extra header field with the VLAN ID). Gives **isolation** in a shared L2 network (hosts 1–2 on one VLAN, 3–4 on another):
  ```
  ip link add link eth0 name eth0.2 type vlan id 2   # convention: <dev>.<vlan id>
  ```
- **Tunnels** **encapsulate** traffic with extra headers so it can cross an IP network (beyond the LAN), then strip them at the far end:
  ```
  ip link add name gen0 type geneve id 55 remote 1.2.3.4
  ```

> **Memorize — Module 1**
> - Data plane = fast packet processing (NAT, forwarding, filtering); control plane = BGP/OSPF/DHCP; management = CLI/Terraform/SNMP.
> - Disaggregation = **white box + NOS + open APIs → more choice, extensibility**. SONiC, Cumulus, DENT all run Linux.
> - **Netlink** = user space ↔ kernel interface for configuring the data plane (sockets, kernel is the server).
> - Containers = **namespaces + cgroups**; Docker = client/daemon/registry; images are immutable.
> - **Containerlab** = Docker-based network labs from YAML; eth0 = management.
> - **tshark** sees whether packets arrive (`-f` capture filter, `-Y` display filter); **pcap** = packet file; **Scapy** crafts packets (`/` stacks layers; sendp L2, send L3).
> - `ip link` can create **bridge, VLAN, VXLAN, GENEVE, GRE, veth**. VLAN = **isolation**.

---

<a id="module-2"></a>
## Module 2 — IP Layer with Linux Networking

### 2.1 Where the IP layer shows up
- Linux as an **end host** (needs an IP and to know where to send traffic); as an end host with a **VPN** (which traffic goes through the tunnel); as a **firewall/load balancer** at the edge (one external NIC, one internal); as a **virtual router** (e.g., a vendor router VM in the cloud).
- **IP** enables **communication between multiple heterogeneous networks** (Cerf & Kahn 1974 "gateway"). Each **interface** gets an address. IPv4 is 32 bits; IPv6 is 128 bits.

### 2.2 `ip addr`
- Each device needs **at least one address** for a protocol, and **can have several**.
- `ip addr add 10.0.1.2/24 dev eth1`; also `del`, `show`, `change`.
- **Adding an address with a prefix length (e.g., /24 or /30) also installs a connected route** for that subnet on the device, as if you ran `ip route add <subnet> dev <dev>`. The route only appears once the device is **up**.
- With multiple addresses on an interface, the host answers to all of them. `ping -I <addr>` picks the source address (and `traceroute -s <addr>`).

### 2.3 Prefixes and LPM (review)
- `a.b.c.d/z`: **z = the number of upper-order bits that uniquely specify the group** (subnet). `2.1.1.0/24` covers 2.1.1.0–2.1.1.255.
- Prefixes give scalability (routers only need the prefix, not every host).
- **Longest prefix match:** 11.11.11.11 with entries /16 → port 1 and /24 → port 2 uses **port 2**; 11.11.12.12 matches only the /16 → port 1.

### 2.4 `ip route`: the forwarding table
- `ip route { add | del | change | append | replace } ROUTE`. ROUTE = NODE_SPEC (the **prefix**, plus table/metric/…) + INFO_SPEC (next hop `via ADDRESS`, `dev NAME`, weight).
- `ip route add 11.11.0.0/16 dev eth3`: the destination is **directly connected** on eth3.
- `ip route add 11.11.0.0/16 via 192.168.2.1 dev eth3`: there's an **IP-layer hop in between** (a router).
- **What `via` changes is who Linux ARPs for:**
  - **Without via**, Linux ARPs for the **packet's destination IP** (11.11.1.1) and puts **that host's MAC** in the frame. This only works if the destination is on the same L2 segment.
  - **With via**, Linux ARPs for the **next hop** (192.168.2.1) and puts the **router's MAC** in the frame; the IP destination is unchanged.
- **Lecture walkthroughs** (lan1, lan2, lan3 each connected to rtr):
  - **Ex 1 (no via):** hosts have `10.0.0.0/16 dev eth1`. Pinging another LAN fails with "Destination host unreachable". tshark on the router shows **ARP "who has 10.0.3.2"** going unanswered, because the hosts aren't on the same L2 segment.
  - **Ex 2 (via):** hosts get a /32 route to the router's interface (`10.0.1.1/32 dev eth1`) plus `10.0.0.0/16 via 10.0.1.1 dev eth1`. The router has `10.0.X.0/24 dev ethX`. Now pings work: the host ARPs for the router, and the router does LPM and forwards.
  - **Ex 3:** **multiple addresses per interface**; choose the source with `ping -I`. (`ip route` can also set a preferred source.)
  - **Ex 4: GRE tunnel.**
    ```
    ip tunnel add gre1 mode gre local 10.0.1.2 remote 10.0.3.2 ttl 255
    ip addr add 192.168.0.1/30 dev gre1
    ip link set gre1 up
    ```
    Pinging 192.168.0.2 uses the route out gre1 → inner packet [IP 192.168.0.1→192.168.0.2][ICMP]. gre1 wraps it: **[outer IP 10.0.1.2→10.0.3.2][GRE][inner IP][ICMP]**. The outer destination 10.0.3.2 then routes out eth1 via the router. On eth1 you see the outer header; on gre1 you see only the inner packet.
- Scripts in the lab use `create`/`delete` functions; deleting an address automatically removes its routes.

### 2.5 `ip neigh`: the ARP / neighbor table
- **ARP** maps IP → MAC (broadcast "who has X?", the owner replies). The table is called the ARP table, neighbor table, or MAC address table.
- `ip neigh show`; `ip neigh add <IP> lladdr <MAC> dev <dev>` (a **static** entry, e.g., to avoid the ARP delay for consistent performance); `ip neigh del …`; `ip neigh flush all` (e.g., after an IP is reassigned to a new machine).
- Mostly for troubleshooting; normally ARP just happens.

### 2.6 Routing in Linux (BGP refresher + BIRD)
- **Forwarding = data plane** (forwarding table, **FIB**); **routing = control plane** (routing table, **RIB**, built by talking to neighbors).
- **BGP refresher:**
  - The Internet = interconnected **autonomous systems**; BGP coordinates them.
  - **Peering:** routers open a **TCP connection**, exchange properties (**AS number**, capabilities), and a state machine reaches **Established**.
  - **Announcements** advertise a path to a prefix (e.g., "D via AS2 AS4"); **withdrawals** retract it (e.g., after a link failure).
  - Each router independently picks **one** best route (for its FIB and to announce): **local preference → AS path length → MED → eBGP over iBGP → shortest IGP cost → router ID**.
- In a Linux router, the "switch fabric" is basically PCIe and forwarding happens in kernel software (unless offloaded to NICs).
- **Linux routing software (the "ecosystem"):** **Quagga, BIRD, FRR, GoBGP**, … They aren't part of Linux itself; NOSes like SONiC and Cumulus use them.
- **BIRD config** (`/etc/bird/bird.conf`, or `bird -c <file>`):
  - `protocol device { }`: gets **interface information from the kernel**.
  - `protocol kernel { ipv4 { export all; }; }`: **synchronizes BIRD's tables with the OS kernel's** forwarding table (one per table; the default table is `master4`).
  - **BGP peering block** (one per neighbor). **Required: your local address + AS, and the neighbor's IP address + AS number.** Optional: hold time (default 240), MD5 password.
    ```
    protocol bgp uplink1 {
      local 198.51.100.1 as 65000;
      neighbor 198.51.100.10 as 64496;
      ipv4 { import filter rt_import; export where source ~ [RTS_STATIC, RTS_BGP]; };
    }
    ```
  - **Filters** (import/export) are BIRD-language functions that accept/reject routes (e.g., `if bgp_path.first != 64496 then accept; … reject;`). Use them for business policy (don't export routes you'd be paying to carry).
  - Also: **templates** (a shared protocol block that only differs by neighbor IP), custom **tables**, **logging**, **debug**.
  - Static routes in the config (picked up through the kernel protocol, or declared) get announced to peers.
- **Running it:** `bird -c /etc/bird-alt/rtrA-bird.conf`. Interact with **`birdc`**:
  - **`birdc show protocols`**: peering sessions and their state (want **Established**).
  - **`birdc show route all`**: **BIRD's routing table** (prefixes, AS paths, which peer it came from; `*` = chosen).
  - `ip route` shows what got installed in the kernel FIB.
- **Diamond lab** (lan1 – rtrA – {rtrB, rtrC} – rtrD – lan2):
  - First assign IPs on all router interfaces (routers need connectivity to peer).
  - rtrA peers with B and C and announces lan1's prefixes; rtrD announces lan2's.
  - rtrA learns lan2's prefix two ways (via B–D and via C–D) and picks one.
  - Ping with `-I` (source = an announced address) and verify the path with tshark and **traceroute** (TTL trick).
  - **Lab ("Peering a BGP Router"):** add a B–C peering, observe longer alternate paths, then **fail A–B and C–D links** and watch BGP reroute A→C→B→D on its own.

### 2.7 Larger routing experiments
- **RIPE** (the RIR for Europe, the Middle East and parts of Central Asia) runs **RIS (Routing Information Service)**, a route collector that **publishes real Internet routing table dumps** you can download (e.g., `bview.20230803.0800.gz`, MRT format; read with **bgpdump**). Output lines look like `TABLE_DUMP2|…|33.0.0.0/8|8455 3356 749|IGP|…`.
- **ExaBGP:** a **programmable BGP agent for testing**. You can configure static routes **with arbitrary AS paths** (BIRD can't) and script announcements/withdrawals in Python or with `exabgpcli`.
  - **Why it's useful:** it **emulates a large set of network nodes as a single process**, so you don't need to build a full Internet topology to experiment with real Internet routes.
- **SEED Internet Emulator** (Syracuse; Kevin Du): Python primitives (**autonomous system, Internet exchange, network, router/BGP speaker, host, service**). It composes Docker containers (multi-node capable), configures peering relationships (peer, provider/customer), and has a **web map UI**. Used for BGP/DNS/botnet attack labs.
- An ExaBGP demo variant injects routing-table-dump routes into rtrA from lan1 and watches them propagate.

> **Memorize — Module 2**
> - `ip addr add <ip>/<len> dev <dev>` (also creates a connected route). `ip route add <prefix> [via <nexthop>] dev <dev>`.
> - **via ⇒ ARP for the next hop; no via ⇒ ARP for the destination.**
> - `ip neigh` = ARP table (show/add/flush). `ip tunnel add gre1 mode gre local A remote B` → outer IP(A→B) + GRE + inner IP.
> - Routing software: Quagga, **BIRD**, FRR, GoBGP. BIRD BGP block needs the **neighbor IP and neighbor AS** (plus local AS). `protocol kernel` syncs with the kernel.
> - **`birdc show protocols`** = sessions; **`birdc show route all`** = routing table.
> - **RIPE RIS** = real routing table dumps (MRT, bgpdump). **ExaBGP** = programmable BGP speaker that injects arbitrary AS paths (emulate many nodes in one process). **SEED** = Python Internet emulator.

---

<a id="module-3"></a>
## Module 3 — Creating a Gateway with Linux

### 3.1 What a gateway is
- In this course: something at the **edge of a network** (public Internet ↔ private network, or between two segments under one admin) that does **more than routing/forwarding**:
  - **Filtering / address translation** → **iptables**.
  - **Load balancing** → **IPVS (ipvsadm)**.
  - **Quality of service** → **tc**.

### 3.2 Filtering and address translation (concepts)
- **Filtering:** block or allow traffic by **policy**.
  - Examples: a web server blocks everything not for port 80; a home gateway blocks everything except connections **initiated from inside**.
  - **Why:** to **keep unwanted traffic from reaching end-host applications and overwhelming them** (and for security). It can run at the network edge or on the hosts themselves.
- **Match-action tables:** **match** (classification; e.g., dstIP = 1.2.3.0/24 and dstPort = 80) → **action** (drop, allow, …).
- **Good practice: default drop**, then add rules that allow specific traffic (whitelisting).
- **Address translation (NAT):** actions can rewrite **source/destination IP and port**, e.g., private addresses on the LAN behind one public IP.
  1. **Static mapping:** each internal server has a fixed public ↔ private pair. Outbound: rewrite the **source IP**. Inbound: rewrite the **destination IP**.
  2. **Dynamic mapping, single external IP (NAPT):** many hosts share one public IP. Outbound: rewrite the **source IP and source port** (pick a free port, record the mapping, e.g., 10.0.1.1:5000 ↔ 123.11.11.11:1000). Inbound replies: rewrite the **destination IP and port** back. This is the typical home network.
  3. **Port forwarding:** run a public server on a private IP (e.g., a Minecraft server on 10.0.1.1:25565). **Inbound: rewrite the destination IP (and the destination port, if the internal port differs)** to the internal server. Outbound replies: rewrite the source back.

### 3.3 iptables
- Part of the **netfilter** framework: user-space utilities (**iptables, nftables**) that configure the kernel's filtering framework.
- **Structure: tables contain chains; chains are lists of rules; rules = match + action.**
  - Command shape: `iptables [-t TABLE] -A|-D|-I CHAIN <match> -j <target>`.
- **The four tables:**
  - **filter** (the **default** when no `-t`): chains **INPUT, OUTPUT, FORWARD**.
  - **nat:** consulted for packets that **create a new connection**. Chains **PREROUTING, INPUT, OUTPUT, POSTROUTING**.
  - **mangle:** specialized packet alteration. All chains.
  - **raw:** mainly exemptions from connection tracking.
- **Netfilter hook points / packet path:**
  - Inbound: **PREROUTING** → routing decision → **INPUT** (local destination) **or FORWARD** (pass through) → **POSTROUTING** → out.
  - Locally generated: **OUTPUT** → POSTROUTING.
  - **Chains are named for their hook point.**
  - **A FORWARD rule never sees traffic addressed to the gateway itself** (that goes through INPUT). A common "why isn't my rule working?" mistake.
- **Chains:** rules are evaluated **sequentially**; they can be terminating (ACCEPT/DROP) or jump to a **user-defined chain**.
- **Matching:**
  - Basic: `-p` protocol, `-s` source, `-d` destination, `-i` in-interface, `-o` out-interface (each can be negated with `!`).
  - **Protocol extensions** (loaded by `-p tcp`): `--dport 8080` / `--destination-port`, port ranges, `--tcp-flags SYN,ACK,FIN,RST SYN`.
  - **Match modules** (`-m`): `-m connlimit --connlimit-above 16 --connlimit-mask 24`; `-m conntrack --ctstate ESTABLISHED,RELATED` (or `-m state --state …`).
- **Actions (`-j` target):** **ACCEPT**, **DROP** (also RETURN, QUEUE).
  - Extensions: **REJECT** (`--reject-with icmp-port-unreachable`, which tells the sender), **LOG** (`--log-prefix`, `--log-level`), SET, **SNAT**, **DNAT**, **MASQUERADE**.
- **The three lecture examples:**

| Command | Table | Chain | Match | Action |
|---|---|---|---|---|
| `iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT` | filter (default) | FORWARD | Connection already established or related | ACCEPT |
| `iptables -A INPUT -p tcp --dport 80 -j DROP` | filter | INPUT | TCP to port 80 | DROP |
| `iptables -A INPUT -i eth1 -s 10.0.0.0/8 -j LOG --log-prefix "IP_SPOOF A: "` | filter | INPUT | From eth1 with source 10/8 (spoofed private source) | LOG |

- **Port forwarding with DNAT:**
  `iptables -t nat -A PREROUTING -d 111.111.0.1 -p tcp --dport 1000 -j DNAT --to-destination 10.10.0.2`
  For inbound traffic to the gateway's **public** IP on TCP 1000, this rewrites the destination to the **internal host 10.10.0.2**. Pair it with an SNAT/MASQUERADE rule in POSTROUTING for the return direction.
- **Practice topology:** ext1, ext2 – gw – int1, int2 (e.g., block ext2 but allow ext1).

### 3.4 Load balancing (concepts)
- A **load balancer** is the **entry point for a service**. It sends each request to one of N servers, for **scaling and resilience**, and usually involves NAT.
- **Server set:** usually statically configured, possibly with **liveness checks (heartbeats)**.
- **Flow affinity:** TCP is stateful, so **every packet of a connection must go to the same server**. The first packet runs the algorithm; later packets are looked up in a flow table (src IP, port → backend). The address of the chosen backend is called the **VIP** in the lecture.
- **Algorithms:**

| Algorithm | How | State | Balanced when… |
|---|---|---|---|
| **Source hash** (`sh`) | hash(srcIP, srcPort) → server | **Stateless** (the hash gives affinity) | Many clients with a spread of source IPs/ports |
| **Round robin** (`rr`) | 1, 2, 3, 1, 2, … | Needs a flow table | **Each request is roughly the same load** |
| **Least connections** (`lc`) | Server with the fewest active connections | Flow table + connection counts | Each *connection* is similar load (handles long-lived connections better) |

- **Layer 4 vs layer 7:**
  - **L4** (network load balancer) looks at IP/TCP headers only.
  - **L7** (**web proxy**) also looks at HTTP (e.g., `/signup` → server 1, `/purchase` → servers 2–3).
  - They're often **layered**: Facebook uses DNS → L4 (IPVS) → L7 proxies → services.

### 3.5 IPVS / ipvsadm
- **IPVS (IP Virtual Server)** does **transport-layer (L4) load balancing in the kernel**. **ipvsadm** sets up and inspects the virtual server table. Needs `modprobe ip_vs`.
- **Two parts: (1) define the service, (2) add the real servers.**
- **Define the service:** `ipvsadm -A|-E|-D -t|-u <service-address[:port]> -s <rr|lc|sh>`
  - `ipvsadm -A -t 207.175.44.110:80 -s rr` = **add a TCP service at 207.175.44.110:80 that picks backends round robin**.
- **Add a server:** `ipvsadm -a|-e|-d -t <service-address> -r <real-server[:port]> [-g|-i|-m] [-w weight]`
  - `ipvsadm -a -t 207.175.44.110:80 -r 192.168.10.1:80 -m`
  - Uppercase `-A/-E/-D` = **service**; lowercase `-a/-e/-d` = **server**.
- **Packet forwarding methods:**
  - **`-g` gatewaying / direct routing** (the **default**): no address change; delivered at the MAC level (extra setup).
  - **`-i` ipip:** tunnel with an outer IP header (src = LB, dst = server).
  - **`-m` masquerading (NAT):** rewrite the destination to the server and keep translation state. **Only `-m` allows the server port to differ** from the service port.
- **`-w` weight** = server capacity, used by weighted schedulers (weighted round robin).
- **Direct Server Return (DSR):** the server replies **straight to the client, bypassing the LB** (less LB load; needs server-side support). Common in production.
- Practice: run `nc` listeners on int1/int2 and balance between them.

### 3.6 Quality of service (concepts)
- **QoS** = the overall performance of a service: **bandwidth, latency, jitter, loss**. Different apps need different things.
- **Three building blocks (egress):**
  - **Classification:** **inspect packets to decide their traffic class** (put them in a queue; e.g., low = email, medium = web, high = video conferencing).
    - By **L3/L4 headers** (port 80 = web; a paying customer's subnet).
    - By **deep packet inspection** (the payload; expensive, but catches what L3/L4 can't).
    - By **fingerprinting** (statistics over many packets: sizes, inter-packet gaps).
  - **Shaping:** **make a class conform to desired properties such as rate**.
  - **Scheduling:** **decide which packet to transmit next and which to drop**.
- **Token bucket** (shaping / rate limiting):
  - Tokens are added at **rate R → the traffic rate**. A packet needs a token to go.
  - **Bucket depth B → the allowed burst**: idle time fills the bucket, so a burst can use the saved tokens; overflow discards tokens, capping the burst.
- **Scheduling, part 1: queue size and drops.**
  - Too big → long delays under congestion; too small → lots of drops.
  - **Tail drop:** drop arrivals when full. The problem: TCP only learns about congestion at its **peak**, about an RTT or timeout later, by which point many more packets were sent and dropped.
  - **RED (Random Early Detection), CoDel, FQ-CoDel** drop **earlier/probabilistically** to signal TCP sooner.
- **Scheduling, part 2: which queue to send from.**
  - **Strict priority** (always the highest non-empty queue; can **starve** the low ones).
  - **Round robin** (one packet per queue; unfair with variable packet sizes).
  - **Fair queuing** (mimics bit-by-bit multiplexing by computing a theoretical departure time per packet).
- **Policing (ingress):** like shaping (enforce a rate), but its **only** corrective action is **dropping** received packets.

### 3.7 tc (traffic control)
- Key constructs: **qdisc, class, filter**.
- **qdisc (queuing discipline):** the main object for **shaping and scheduling**.
  - **Every interface has an ingress and an egress qdisc.**
  - The default is **pfifo_fast** (classifies by type-of-service into bands, prioritized FIFO).
  ```
  tc qdisc [add|delete|replace] dev DEV [parent ID | root] [handle ID] QDISC [params]
  tc qdisc add dev eth0 root handle 1: tbf rate 1mbit burst 32kbit latency 400ms
  ```
  - In that example, the qdisc type is **tbf (token bucket filter)**. `root` is its position; `1:` is its handle; `rate`/`burst`/`latency` are tbf parameters.
- **Classless qdiscs:** pfifo/bfifo (plain FIFO by packets/bytes), pfifo_fast, **sfq** (stochastic fair queuing), **tbf**, codel/fq_codel.
- **Classful qdiscs** contain **classes**, which contain further qdiscs. Example: **HTB (Hierarchical Token Bucket)**, where child classes can **share/borrow tokens**.
  ```
  tc qdisc add dev eth0 root handle 99: htb
  tc class add dev eth0 parent 99:  classid 99:1  htb rate 100mbit
  tc class add dev eth0 parent 99:1 classid 99:10 htb rate 30mbit
  tc class add dev eth0 parent 99:1 classid 99:11 htb rate 70mbit
  tc qdisc add dev eth0 parent 99:10 fq_codel
  tc qdisc add dev eth0 parent 99:11 fq_codel
  ```
  - Classes can't be root, use `classid` instead of `handle`, and must be the **same type as the parent qdisc**.
- **Filters** put traffic into classes:
  ```
  tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip src 1.2.3.4 classid 99:10
  ```
  - `protocol` is all, ip or ipv6. **Lower prio number = evaluated first.**
  - Filter types: **u32** (arbitrary packet bits), **fw** (firewall mark → class), **flow/flower** (flow keys).
  - Action: `classid X` (send to a class) or `action …`.
- **netem:** **network emulation**. Adds delay, jitter and loss to test apps under real-world (e.g., wireless) conditions:
  ```
  tc qdisc replace dev eth0 root netem delay 100ms 12ms 10%
  ```
  - Put a Linux node between two hosts, forward with ip route, and apply netem on both interfaces.
- Practice: measure with **iperf** (server on int1, client on ext1), then rate-limit with tc and measure again.

> **Memorize — Module 3**
> - Gateway functions → **iptables** (filter/NAT), **ipvsadm** (LB), **tc** (QoS).
> - Port forwarding rewrites the **destination IP/port inbound** (DNAT in **nat PREROUTING**). NAPT rewrites the **source IP/port outbound**.
> - iptables: **tables (filter default, nat, mangle, raw) → chains (hook points) → rules (match + -j target)**. FORWARD ≠ traffic to the box itself.
> - LB algorithms: **sh** (stateless hash), **rr** (equal-cost requests), **lc**. Flow affinity matters because TCP is stateful. L4 vs L7.
> - `ipvsadm -A -t VIP:port -s rr` (service) then `ipvsadm -a -t VIP:port -r server:port -m` (server). -g default, -i ipip, -m NAT.
> - QoS: **classification** (what class), **shaping** (conform to a rate), **scheduling** (what to send next/drop). **Policing** = ingress, drop only.
> - **Token bucket: token rate = traffic rate; bucket depth = burst.**
> - tc: **qdisc/class/filter**; `tbf` = token bucket filter; HTB classful; **netem** emulates delay/loss.

---

<a id="module-4"></a>
## Module 4 — Virtual Networking in Linux

### 4.1 Virtualization
- **Technology for abstracting the underlying hardware/software.**
- **Benefits:** **more efficient use of resources** and **simpler deployment** (by isolating configuration).
- **Examples:** cloud computing (VMs on AWS servers), **network function virtualization** (routers and firewalls as VMs in a provider like AT&T), and **virtualizing the Linux network stack for containers** (this module's focus).

### 4.2 Network namespaces
- **Namespace** = kernel isolation that gives processes their own view (and naming) of resources. **cgroups** allocate CPU/memory.
- **In the kernel:** each process's `task_struct` has an `nsproxy` pointer → a struct holding the IPC, mount, PID, … and **`struct net *net_ns`** (the network namespace).
  - `struct net` contains `netns_ipv4` (the **FIB** = forwarding table, rules) and `netns_nf` (the **netfilter** tables).
  - **So each network namespace gets its own interfaces, forwarding tables, ARP tables and iptables rules.**
- That's why, in containerlab, `ip route add` in host1 doesn't show up in the router's table: each container is a separate network namespace.
- Processes inherit their parent's namespaces. Outside any container you're in the **root (default) namespace**.

### 4.3 `ip netns` and veth pairs
**Three setup steps:**
1. **Create a namespace:** `ip netns add nsDemo` (`ip netns list` to see it).
2. **Create a veth pair:** `ip link add vethY type veth peer name vethZ`.
   - **veth** = virtual Ethernet. Always created as an **interconnected pair**; a packet sent on one end **immediately appears on the other** (like a virtual cable). Both ends start **down**.
3. **Move one end into the namespace:** `ip link set vethZ netns nsDemo`. It **disappears from `ip link` in the root namespace**.
- **Both ends can go into (different) non-root namespaces**, connecting two namespaces directly: `ip link set vethZ netns nsDemo1; ip link set vethY netns nsDemo2`.
- **Running commands inside:** `ip netns exec nsDemo <cmd>`, or the shortcut `ip -n nsDemo <ip-subcommand>`:
  ```
  ip netns exec nsDemo ip addr add 10.10.10.10/24 dev vethZ
  ip netns exec nsDemo ip link set vethZ up      # connected route appears once up
  ip addr add 10.10.10.11/24 dev vethY           # root side
  ip netns exec nsDemo ping 10.10.10.11
  ip -n nsDemo route add 11.11.0.0/16 dev vethZ
  ```
- **Finding a veth's peer:** `ethtool -S vethY` shows `peer_ifindex: 5`; then `ip link | grep '^5:'` (or `ip netns exec <ns> ip link` inside each namespace) finds interface index 5.
  - Root-side names show `vethZ@vethY` / `veth…@ifNN`.
- **Moving a veth out of a namespace:** it's invisible from root, so run the command **inside** the namespace:
  - `ip netns exec nsDemo ip link set vethZ netns 1` (1 = the root namespace's PID).
  - Or move it straight to another namespace: `ip netns exec nsDemo ip link set vethZ netns nsDemo-new`.
- veths are ordinary Ethernet devices, so all the Linux networking tools apply to them (bridges, routes, iptables, tunnels).

### 4.4 Networking between namespaces (lab cases)
- **Case 1, bridge:** ns1/ns2/ns3 with pairs A–B, C–D, E–F.
  1. `ip netns add` ×3.
  2. `ip link add vethA type veth peer name vethB` (etc.).
  3. Move B, D, F into the namespaces.
  4. Bring all veths up (root side directly; namespace side with `ip netns exec`).
  5. `ip link add mybridge type bridge; ip link set mybridge up`.
  6. `ip link set vethA master mybridge` (and C, E).
  7. Addresses 10.0.0.1/2/3/24 on B/D/F. No `via` needed: they're all on one L2 segment.
  - Pinging ns3 from ns1: **ARP is broadcast through the bridge**, then ICMP flows.
- **Case 2, external access / port forwarding** (two VMs, node1 192.168.1.100 and node2 192.168.1.101):
  - ns1 (10.10.10.2) and ns2 (11.11.11.2) connect to the root via veths A (10.10.10.1) and C (11.11.11.1).
  - Each namespace has a **default route via its root-side veth address**.
  - On node1: `iptables -t nat -A PREROUTING -i enp0s8 -d 192.168.1.100 -p tcp --dport 80 -j DNAT --to-destination 10.10.10.2` (and port 25565 → 11.11.11.2), plus **SNAT** on egress (src 10.10.10.2 port 80 → 192.168.1.100).
  - Test with `nc -l` in each namespace and `nc 192.168.1.100 80` from node2.
  - Run `clear-fw.sh` first so Docker's iptables rules don't interfere.
- **Case 3 (on your own):** extend a namespace network between machines with a **GRE tunnel**.
- **Lab ("Investigating Network Namespaces"):** an **obfuscated** script builds namespaces, veths and routes. You use `ip netns list`, `ip netns exec`, `ip link`, `ip route`, **ethtool**, ping and tshark/tcpdump to work out the namespaces, veth pairs, reachability and paths.
  - Troubleshooting method: for each namespace, list its veths and their peer ifindexes, its addresses, and its routes. A ping works only if there's a route **both ways**.

### 4.5 Docker networking
- `docker run -d --rm --name mynginx nginx` creates a veth in the root namespace, but **`ip netns list` shows nothing**.
  - **Why:** `ip netns` looks in **`/var/run/netns`**, and Docker **doesn't create a reference file there**.
  - **Fix** (the `run-container.sh` script): get the container's PID (`docker inspect` → `State.Pid`), `touch /var/run/netns/mynginx`, and `mount -o bind /proc/<pid>/ns/net /var/run/netns/mynginx`. Now `ip netns exec mynginx ip link` shows the container's **eth0**, which pairs with the root-side `veth…` (confirm with `ethtool -S`).
- **docker0** is a **bridge** and the **default network that every container's root-side veth attaches to** (`ip link` shows `master docker0`). It's 172.17.0.1/16; containers get 172.17.0.2, .3, …
- **Custom networks:**
  - `docker network create mynet1` (a new bridge `br-<id>`, 172.18.0.0/16) or `--subnet=11.11.11.0/24`.
  - `docker run --net=mynet1 …` attaches a container to it. `docker network connect <net> <container>` adds a second interface (eth1).
- **Port forwarding:** `docker run -p 8080:80 …` (host port 8080 → container port 80). **Docker implements this with iptables NAT rules** (compare `iptables -t nat -L` before and after).
- **docker compose:** a **YAML wrapper** for Docker commands (build, pull, run, networks) covering **multiple containers**. Define `services:` (build or image, `networks:`) and `networks:` (drivers). Run with `docker compose up`.

> **Memorize — Module 4**
> - Virtualization benefits: **efficient resource use, simpler deployment**.
> - Each network namespace has its own **interfaces, FIB, netfilter tables** (`struct net`).
> - **Three steps:** `ip netns add` → `ip link add A type veth peer name B` → `ip link set B netns NS`.
> - `ip netns exec NS cmd` = `ip -n NS …`. Moving out: run inside the namespace, `… netns 1`.
> - Find a peer: **`ethtool -S vethX` (peer_ifindex) + `ip link`**. A veth pair can connect two non-root namespaces directly.
> - **docker0 = default bridge**; Docker namespaces are invisible to `ip netns` (no /var/run/netns entry); `-p` port forwarding uses **iptables NAT**.

---

<a id="module-5"></a>
## Module 5 — Kubernetes Networking with Linux

### 5.1 Why Kubernetes
- Docker runs individual containers. **Distributed apps** (presentation, logic and data tiers) on a **cluster** raise four challenges, and Kubernetes answers each:
  - **Where to run:** a **scheduler** picks nodes.
  - **How to communicate:** networking constructs keep communication working through changes.
  - **Scaling:** constructs like Deployments and Services.
  - **Failure:** it **monitors and restarts** containers.
- **Kubernetes (k8s)** = an open-source **container orchestration** system for automating deployment, scaling and management.
  - **You declare the desired state; k8s works out how to reach it and keeps it there.**
  - `kubectl` talks to the **control plane**, which assigns work to **workers**.

### 5.2 Using Kubernetes
- **Making a cluster:** (1) VMs + **kubeadm**, or (2) **KinD (Kubernetes in Docker)**, where **each node (control plane or worker) is a Docker container** (the course uses kind).
  - `kind create cluster --config cluster-configs/1master2worker.yaml`; `docker ps` shows the node containers; `kind delete cluster`.
- **kubectl:** the management CLI. `~/.kube/config` tells it where the API server is.
  - Commands used: **apply, delete, get, describe, logs** (+ `exec`, `label`, `port-forward`).
- **YAML resources** have the base fields `apiVersion`, `kind`, `metadata`, `spec`.
- **Pod** = **the unit of work**. Often one container, but can be several. **The Pod gets a unique IP address in the cluster** (all of its containers share that one IP).
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata: { name: mynginx }
  spec:
    containers:
    - name: mynginx
      image: nginx
  ```
  - `kubectl apply -f simple-nginx.yaml`; `kubectl get pods -o wide` (shows the IP and node); `kubectl logs mynginx`; **`kubectl describe pod mynginx`** (details and errors; good for troubleshooting); `kubectl delete --force pod mynginx`.
- **Pod IPs are local to the cluster network.** For quick tests: `kubectl port-forward pod/mynginx 8000:80` then `curl 127.0.0.1:8000`.
- **Controllers:** don't create Pods directly; create a resource that **creates and manages** them.
  - **Deployment:** you describe the desired state and the Deployment controller moves the actual state there **at a controlled rate** (via a **ReplicaSet**).
  - Uses: scale Pods up/down, controlled upgrades/downgrades, controlled Pod spec updates.
  - Fields: `replicas: 3`, `selector.matchLabels: {app: nginx}`, `template:` (the Pod spec with `labels: {app: nginx}`). Change `replicas` and re-apply to scale.
- **Labels/selectors:** arbitrary key-value pairs on resources (`kubectl label pods mynginx app=nginx`) used to pick sets of resources.
- **Service:** "an abstraction which defines a **logical set of Pods** and a policy to access them". It **load balances** across them, using a **selector** so membership follows scaling and failures.
  - It gets its own (virtual) IP. `kubectl describe service mynginx-service` lists the **Endpoints** (the Pod IPs).
  - `kubectl port-forward service/mynginx-service 8000:80`.

### 5.3 Kubernetes architecture
- **Control plane:** **API server**, **etcd**, **scheduler**, **controller manager**. **Workers:** **kubelet**, **container runtime** (e.g., containerd), **kube-proxy**. Add-ons: DNS, dashboard, ingress controller / external LB, **CNI plugin**.
- **kubelet is the only component that always runs as a regular system service.** It runs everything else as Pods, including the control plane via **static Pods** (manifests in `/etc/kubernetes/manifests/`).
  - Explore with `docker exec` into a node, then `ps aux`, `crictl ps`, `kubectl get all -A`.
- **etcd:** a distributed **key-value store** holding **all cluster state** (Pods, Deployments, …). Clients use port **2379**; peer sync uses **2380**.
  - It uses the **Raft** consensus algorithm: each node's state is either what the **majority agrees** is current, or a previously agreed state.
- **API server:** a **CRUD interface over a REST API** for cluster state, which it **stores in etcd**.
  - Request path: **authentication plugins → admission control plugins** (can modify or reject, e.g., fill missing fields, apply the default ServiceAccount, NamespaceLifecycle blocks Pods in a namespace being deleted) **→ resource validation → etcd**.
  - Components **watch** the API server (`GET …?watch=true`; also `kubectl get pods --watch`). After an update, etcd notifies the API server, which pushes the updated object to all watchers.
- **Scheduler:** watches for **new unscheduled Pods** and **assigns each a node** (it doesn't tell the node to run it).
  - **Filter** (feasible nodes) → **score** (rank them). Broken into 11 extension points.
- **Controller manager:** runs controllers (ReplicaSet, DaemonSet, Deployment, Node, Service, Endpoints, …).
  - Each **watches** the resources it cares about and runs a **reconciliation loop** (desired vs current state). E.g., ReplicaSet creates/deletes Pods; Endpoints builds Endpoints from Services + Pods.
- **kubelet:** registers its Node, **watches for Pods scheduled to its node**, tells the **container runtime** to start the containers, monitors them and reports events, runs **liveness probes** (restarts on failure), deletes containers when Pods are deleted, and runs static Pods.
- **Chain of events for `kubectl apply` of a Deployment:**
  1. kubectl creates the **Deployment** → API server/etcd.
  2. Deployment controller (notified via watch) creates a **ReplicaSet**.
  3. ReplicaSet controller creates the **Pods**.
  4. **Scheduler** assigns each Pod to a node.
  5. That node's **kubelet** (watching) tells the container runtime to run the containers.
  - Every hand-off goes through the API server and etcd (create → watch notification).
- **Namespaces (Kubernetes):** groupings/partitions of resources (`default`, `kube-system`, …). Not the same thing as Linux network namespaces.

### 5.4 Kubernetes networking
- **Needed communication:** Pod→Pod, Pod→set of Pods, Pod→external service, external→Pod(s).
- **Pod to Pod:** every Pod gets an IP (kind uses **10.244.0.0/16**, with a /24 per node), even across nodes.
  - **CNI (Container Network Interface):** the **container runtime creates the Pod, then calls a network plugin** to set up its networking.
  - The plugin's configuration and API follow the **CNI spec**, with commands **ADD, DEL, CHECK**, ….
  - **Flannel** (a popular plugin) creates a veth pair per Pod attached to a **cni0 bridge**, plus a **VXLAN** tunnel between nodes (GRE would also work).
- **Pod to set of Pods = Service**, implemented by **kube-proxy**:
  - Runs on **every node**.
  - It **programs iptables or IPVS** instead of sitting in the traffic path (iptables is the default) when the API server's Service/Endpoints change.
  - The Service IP is **virtual** (no interface is created). iptables in each node rewrites the destination from the Service IP (e.g., 10.99.89.41) to one of the endpoint Pod IPs, and normal Pod-to-Pod networking takes over.
  - **Load balancing with iptables** uses the `statistic` module to spread connections evenly across 3 endpoints:
    ```
    iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 -m statistic --mode random --probability 0.33 -j DNAT --to-destination 10.0.0.2:1234
    iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 -m statistic --mode random --probability 0.5  -j DNAT --to-destination 10.0.0.3:1234
    iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 -j DNAT --to-destination 10.0.0.4:1234
    ```
    (1/3 match the first rule; half of the remaining 2/3 match the second; everything left matches the last. Each gets 1/3.)
- **Pod to external:** a Service with **`type: ExternalName`** (`externalName: blah.com`). Pods use the same Service mechanism, so you can switch between internal and external backends.
- **External to Pods:** **Ingress + Ingress Controller.**
  - The **Ingress Controller** is a deployed **load balancer** (nginx, traefik, haproxy). Deploy it first; cloud providers already have.
  - An **Ingress** resource **tells the Ingress Controller how to reach a Service** (host/path rules → backend Service and port). The controller configures itself from it.
  - It's **layer 7**, so different paths can go to different Services.
  - (`kubectl port-forward` is only for testing.)

### 5.5 Creating a CNI network plugin
- **How CNI plugins are found:**
  1. **Config** in **`/etc/cni/net.d/`**; the file with the **lowest numeric prefix** wins (e.g., `09-nppnet.conflist` beats kind's `10-kindnet.conflist`).
  2. The config's **`"type"`** names an **executable in `/opt/cni/bin/`** (kind uses `ptp`) that implements the CNI commands.
  3. Both are installed **on every node**; the config differs per node (e.g., subnet 10.244.1.0/24 vs 10.244.2.0/24).
- **The two files of a network plugin:** a **configuration file describing the plugin** and **an executable that implements the CNI spec**.
- **Skeleton test:**
  - `kind create cluster --config 1worker2Mount.yaml` (mounts host directories).
  - Copy the conf and executable into `/etc/cni/net.d` and `/opt/cni/bin` on the worker.
  - `kubectl label node kind-worker node=node1`, then apply a Pod with a node selector.
- **Debugging a plugin:**
  - `kubectl describe pod <pod>` (shows errors).
  - `kubectl describe node <node>` (config problems).
  - **`journalctl -u kubelet`** inside the node.
  - **The plugin's own log** (`cat /var/log/nppnet.log`).
- **What the full plugin does (ADD):**
  1. **Allocate an IP** from the node's subnet.
  2. Create an entry in **/var/run/netns/** (so `ip netns` works).
  3. **Create a veth pair.**
  4. **Move one end into the Pod's namespace.**
  5. **Set the address and route** inside (e.g., `10.244.0.0/16 via 10.244.1.1`).
  - The root end attaches to a per-node **cni0** bridge (10.244.1.1).
  - **Inter-node routes** live on the host (`10.244.2.0/24 via 172.18.0.3`; kind sets these up by default).
  - The install script (`nppnet-install.sh`) puts the config and binary in place and creates cni0 on each node.
  - Test: `kubectl exec -it forexec3 -- ping 10.244.2.2`.
- **Course recap and next step:** M1 bridges/devices → M2 routers (forwarding + routing daemons) → M3 L4 (iptables, ipvs, tc) → M4 namespaces/Docker → M5 Kubernetes and a CNI plugin. **Suggested next topic: eBPF/XDP.**

> **Memorize — Module 5**
> - k8s = **declare desired state**; scheduler, scaling, restarts. **kind** = k8s nodes as Docker containers.
> - **Pod** = unit of work with **one IP per Pod** (shared by its containers). Deployment → ReplicaSet → Pods. Service = selector + load-balanced virtual IP.
> - Control plane: **API server (REST CRUD) + etcd (Raft KV) + scheduler (filter/score) + controller manager (reconcile loops)**. Node: **kubelet + runtime + kube-proxy**. kubelet runs the rest as (static) Pods.
> - **CNI:** runtime calls the plugin (ADD/DEL/CHECK); config in **/etc/cni/net.d** (lowest number wins) + executable in **/opt/cni/bin**.
> - **kube-proxy programs iptables/IPVS** to implement Services. iptables LB uses `-m statistic --probability`.
> - **Ingress** = rules telling the **Ingress Controller** (nginx etc., an L7 LB) how to reach Services. ExternalName = Pod→external.
> - Debug a plugin: `kubectl describe pod/node`, `journalctl -u kubelet`, the plugin's log.

---

<a id="quick-reference"></a>
## Command Cheat Tables

### iproute2

| Task | Command |
|---|---|
| Show devices | `ip link show [dev X]` · `ip -d -j -p link show` |
| Bring up / set MAC | `ip link set dev eth0 up` · `ip link set dev eth0 address …` |
| Bridge | `ip link add name br0 type bridge` · `ip link set eth1 master br0` |
| VLAN | `ip link add link eth0 name eth0.2 type vlan id 2` |
| GENEVE / VXLAN | `ip link add name gen0 type geneve id 55 remote 1.2.3.4` |
| GRE | `ip tunnel add gre1 mode gre local A remote B ttl 255` |
| veth pair | `ip link add vethA type veth peer name vethB` |
| Delete device | `ip link delete dev X` |
| Address | `ip addr add 10.0.1.2/24 dev eth1` · `ip addr show` |
| Route (direct) | `ip route add 10.0.1.0/24 dev eth1` |
| Route (via router) | `ip route add 10.0.0.0/16 via 10.0.1.1 dev eth1` |
| ARP table | `ip neigh show` · `ip neigh add IP lladdr MAC dev X` · `ip neigh flush all` |
| Namespace | `ip netns add NS` · `ip netns list` · `ip link set vethB netns NS` |
| Run in namespace | `ip netns exec NS <cmd>` · `ip -n NS route …` |
| veth peer | `ethtool -S vethA` → peer_ifindex → `ip link` |

### iptables

| Task | Command |
|---|---|
| Allow return traffic | `iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` |
| Block port 80 to this host | `iptables -A INPUT -p tcp --dport 80 -j DROP` |
| Port forward | `iptables -t nat -A PREROUTING -d PUB -p tcp --dport P -j DNAT --to-destination PRIV` |
| Source NAT | `iptables -t nat -A POSTROUTING -s PRIV -o ext -j SNAT --to-source PUB` (or `-j MASQUERADE`) |
| List | `iptables -L` · `iptables -t nat -L` |

### IPVS / tc

| Task | Command |
|---|---|
| Add LB service | `ipvsadm -A -t VIP:80 -s rr` (rr / lc / sh) |
| Add backend | `ipvsadm -a -t VIP:80 -r 192.168.10.1:80 -m [-w 2]` (-g / -i / -m) |
| Token bucket | `tc qdisc add dev eth0 root handle 1: tbf rate 1mbit burst 32kbit latency 400ms` |
| HTB class | `tc class add dev eth0 parent 99:1 classid 99:10 htb rate 30mbit` |
| Filter | `tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip src 1.2.3.4 classid 99:10` |
| Emulate WAN | `tc qdisc replace dev eth0 root netem delay 100ms 12ms 10%` |

### BIRD / tools

| Task | Command |
|---|---|
| Start BIRD | `bird -c /etc/bird-alt/rtrA-bird.conf` |
| Peering state | `birdc show protocols` |
| Routing table | `birdc show route all` |
| Capture | `tshark -i eth1 -f "arp"` · `tshark -Y "tcp.port == 80"` |
| Containerlab | `sudo containerlab deploy` / `destroy` · `docker exec -it clab-<lab>-<node> <cmd>` |
| Traceroute with source | `traceroute -s 1.1.1.1 4.4.4.4` |

### Docker / Kubernetes

| Task | Command |
|---|---|
| Run container | `docker run -d --rm --name mynginx nginx` |
| Port forward | `docker run -p 8080:80 …` |
| Custom network | `docker network create --subnet=11.11.11.0/24 mynet` · `docker run --net=mynet …` |
| Container PID | `docker inspect mynginx` → `State.Pid` |
| Cluster | `kind create cluster --config …` · `kind delete cluster` |
| Apply / inspect | `kubectl apply -f x.yaml` · `kubectl get pods -o wide` · `kubectl describe pod X` · `kubectl logs X` |
| Test access | `kubectl port-forward service/S 8000:80` |
| Plugin debug | `journalctl -u kubelet` · `cat /var/log/<plugin>.log` |

---

<a id="quiz-map"></a>
## Graded Quiz Coverage Map

These summary quizzes are graded (5% each), so under the course's Conditional AI policy this guide does **not** answer them. Use this map to find which sections to review before you take each one.

| Graded item | Topics it draws on | Review |
|---|---|---|
| **Introduction to Linux Networking Summary** | Disaggregation benefits, Netlink, containerlab, pcap, tshark vs other tools, ip link device types, VLANs, Scapy's role in the bridge demo | §1.1–1.5 |
| **IP Layer with Linux Networking Summary** | What IP provides, prefix length meaning, `via` in ip route, BIRD peering config, RIPE, birdc commands, ExaBGP | §2.1–2.7 |
| **Creating a Gateway with Linux Summary** | Why filter, NAT fields for port forwarding, iptables default table, DNAT example, round robin assumptions, ipvsadm service syntax, shaping vs scheduling, token bucket, tbf qdisc | §3.2–3.7 |
| **Investigating Network Namespaces** (lab) | Counting namespaces, finding veth pairs and which namespace they're in, ping reachability, forward paths | §4.3–4.4 (method) |
| **Virtual Networking in Linux Summary** | Virtualization benefits, finding a veth's peer, veth placement, moving a veth between namespaces, docker0, Docker and NAT | §4.1–4.5 |
| **Kubernetes Networking with Linux Summary** | Pod IPs, kube-proxy's role, Ingress, network plugin files, debugging a plugin | §5.2–5.5 |

Once a quiz has been graded, Coursera shows you the feedback. Comparing it with these sections is a good way to find weak spots before the final.
