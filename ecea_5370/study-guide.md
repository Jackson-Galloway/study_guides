# ECEA 5370 — Network Systems Foundations
## Final Exam Study Guide

*Built from the course lecture transcripts, slides and readings (instructor: Eric Keller, CU Boulder MS-ECE).*

**Final exam format** (from "About the Final Exam"): **unproctored**, **2 hours**, **34 questions**, **1 attempt / 1 submission**. Multiple-choice questions plus **one free-response question**; no programming questions. Worth **10%** of the grade (quizzes are 8% each, programming assignments 10% each).
**Not allowed:** notes or external websites. Unlike your earlier courses, there's **no notes sheet**, so the "Memorize" boxes below are things to know cold.

> **AI policy for this course: Conditional AI.** AI may be used to summarize material and generate practice questions (which is what this guide and the practice exam are). It may **not** be used to answer graded quizzes, or to enter quiz questions into an AI tool. That's why there's no quiz answer key here. The [quiz coverage map](#quiz-map) at the end points to the sections each graded quiz draws on, without answering anything.

---

## Table of Contents
1. [Module 1 — Link Layer](#module-1)
2. [Module 2 — Network Layer](#module-2)
3. [Module 3 — Transport Layer](#module-3)
4. [Module 4 — Application Layer](#module-4)
5. [Module 5 — Network Security](#module-5)
6. [Quick-Reference Tables](#quick-reference)
7. [Graded Quiz Coverage Map](#quiz-map)

---

<a id="module-1"></a>
## Module 1 — Link Layer

### 1.1 What the link layer is and the four problems it solves
- A **link** is the abstraction over a physical medium (wire or wireless) that lets communication happen.
- It's implemented mostly in **hardware on the NIC** (network interface card), plus firmware and some OS functionality.
- **The four problems:**
  1. **Encoding**: structure the 1s and 0s so the receiver knows who it's for and how to interpret it → **frames**.
  2. **Error handling**: detect errors (and ideally correct them) → **parity, CRC**.
  3. **Sharing the link**: regulate access to a shared medium → **multiple access protocols** (TDMA, FDMA, CSMA/CD, CSMA/CA).
  4. **Extending into a LAN** beyond the physical limits of one medium → **Ethernet switches, wireless access points**.

### 1.2 Encoding into frames
- A **frame** = **header + payload + trailer**. The payload is the data; the header and trailer are metadata, broken into **fields**.
- **Ethernet frame:** `Dest (48 bits) | Source (48 bits) | Type (16 bits) | Data | FCS (32 bits)`
  - **Dest**: lets the NIC **filter** frames not meant for it.
  - **Source**: who sent it, so the receiver can reply.
  - **Type (EtherType)**: what the payload is. **0x0800 = IPv4**, **0x0806 = ARP**, 0x0842 = Wake-on-LAN.
  - **FCS (Frame Check Sequence)**: error detection. It's a **trailer** because it's the last thing received, so the NIC already has all the bits needed to check it.
- **MAC address** (Media Access Control): **48 bits**, written as 6 hex pairs separated by colons (`11:22:33:AA:BB:CC`).
  - The upper bits are the **OUI** (Organizationally Unique Identifier, assigned to the vendor); the lower bits are vendor-defined.
  - Historically **globally unique**, assigned at manufacture. **Virtualization** breaks this, since VM software picks arbitrary MACs.
  - **Broadcast address: `FF:FF:FF:FF:FF:FF`** (all 1s), meaning the frame is for every node on the network.
  - Recognize formats: `20:f1:9e:bd:3b:18` is a MAC; `2001:db8::…` is IPv6; `10.10.2.10` is IPv4.
- Other link protocols (ATM, 802.11 Wi-Fi) have different frame formats. **Key takeaway:** a standard defines the format so everyone using it can talk.

### 1.3 Error handling
**Parity**
- Add **one extra bit** so the transmission has an **even** (even parity) or **odd** (odd parity) number of 1s.
- Count 1s with **XOR**: XOR all the bits together. Result 0 = even number of 1s, 1 = odd.
- **Even parity:** parity bit = XOR of the data bits. Example: `11000000` → XOR = 0 → send `110000000`; `01010111` → XOR = 1 → send `010101111`.
- **Receiver:** XOR every bit including the parity bit. Even parity: 0 = OK, 1 = error.
- **Limitation:** catches any single-bit error, but **two flipped bits cancel out** and go undetected.

**CRC (Cyclic Redundancy Check)**
- More powerful; **especially good at burst errors**. Used for the **Ethernet FCS**.
- Setup: **D** = data (d bits), **G** = generator polynomial (**r+1 bits**, fixed by the standard), **R** = the **r CRC bits** to find.
- Transmit `<D,R>` = D·2^r XOR R, chosen so `<D,R>` is **exactly divisible by G (mod 2)**.
- **Calculation:** append r zeros to D, then do long division by G using **XOR instead of subtraction**. The remainder is R.
  - Quotient bit is 1 when G "fits" (the current bits are as long as G and start with 1), otherwise 0.
- **Check:** the receiver divides the received `<D,R>` by G. **Remainder 0 = no error**, anything else = error.
- **Lecture example:** r = 3, G = 1010, D = 11101010 → append `000` → remainder **R = 010** → transmit `11101010010`. (The slides show 010, which is correct; the video misspeaks as "110".)
- **Ethernet CRC-32 detects:** any 1-bit error; any 2 adjacent 1-bit errors; **any odd number** of 1-bit errors; **any burst of length ≤ 32**.
- It **can be calculated as bytes arrive off the wire** (just XORs, so simple to do in NIC hardware/firmware).
- Ethernet's G = **0x04C11DB7**. Read the polynomial terms as bit positions (x^0, x^1, x^2 present → …0111 = 7, etc.). The real standard also complements bits at the start and end to handle leading zeros.
- **CRC detects errors but does not correct them.**

### 1.4 Sharing the link (multiple access)
- On a shared medium (the original single-cable Ethernet, or wireless), simultaneous transmissions **collide**, and **no useful data gets through**.
- **Goal:** share the link **fairly and efficiently**. Two approaches:

**Channel partitioning**
- **TDMA (Time Division Multiple Access):** equal time slots, assigned round-robin; nodes send only in their slot.
  - Example: 100-byte slots; node 1 has 10 bytes (wastes 90), node 2 has 0 (whole slot idle), node 3 has 200 (must wait for its next turn), node 4 has 150.
- **FDMA (Frequency Division Multiple Access):** each node gets its own frequency band.
- **Fair but not efficient:** the channel is **wasted unless every node has data to send**. Partitioning has **no collisions** and needs no complex arbiter.

**Random access** answers three questions: **when to send**, **how to know it worked**, **what to do on failure**.
- **When to send: CSMA (Carrier Sense Multiple Access)**, used by both Ethernet (802.3) and Wi-Fi (802.11). **Listen first**; transmit only when the channel is idle.
  - Collisions can **still** happen because signals take time to propagate (two nodes both sense idle, then both start).
- **Knowing it worked:**
  - **CSMA/CD (Collision Detection), Ethernet:** Ethernet NICs **can read while transmitting**. If what it reads differs from what it sent, there was a collision. It then sends a **jam signal** (a fixed bit pattern) so everyone detects the collision.
  - **CSMA/CA (Collision Avoidance), Wi-Fi:** 802.11 NICs **cannot read while transmitting**, and nodes may be out of range of each other (**hidden terminal**: X and Z both reach access point Y but can't hear each other). So it relies on an **explicit ACK from the receiver**; **no ACK within a timeout = assume a collision**.
    - Optional **RTS/CTS** (Request To Send / Clear To Send): the AP, which hears both sides, grants permission.
- **On failure: backoff** (wait, then retransmit).
  - The wait is **random**, so the colliding nodes don't collide again in lockstep.
  - It's **exponentially increasing**: each repeated failure waits longer.

### 1.5 Local area networks
- One shared cable doesn't scale (more collisions, length limits). **Switches** give each node a **point-to-point, full-duplex link** and direct traffic to its destination.
- **MAC address table:** maps **destination MAC → output port**. Receive a frame, look up dest, send it out that port. It's an **exact match** lookup.
- **MAC learning** (the table starts empty):
  - On receiving a frame, the switch **learns the source MAC is on the incoming port**.
  - If the **destination is unknown, flood** out all ports (except the incoming one). Non-destinations ignore it.
  - When the destination replies, the switch learns that port too.
- **Switch decision rules:**
  - Dest = **broadcast** FF:FF:FF:FF:FF:FF → **flood out all ports**.
  - Dest **in the table** → forward out **that port only**.
  - Dest **not in the table** → **flood out all ports**.
- **Topologies of switches:** switches have limited ports (4 to 48), so networks are trees of switches; each has its own MAC table (several MACs can map to one uplink port).
- **The loop problem:** broadcasts and flooding circulate and **amplify** forever in a topology with loops. Solutions:
  - Track messages seen (too much state).
  - **Limit hop count** (state carried in the frame).
  - **Spanning Tree Protocol (STP):** turns off links to make the topology loop-free (costs some bandwidth).
  - **Architect the network** so loops can't happen (e.g., topology design, or pre-populate MAC tables to avoid flooding; common in data centers).
  - In Ethernet LANs, **STP and loop-free architecture** are the popular choices.
- **Wireless LAN:** the special nodes are **access points (APs)**, usually connected to a wired network.
  - **All traffic goes through the AP**, even when two nodes are in range of each other.
  - Key challenge (not covered in depth): **associating** with an AP.

**Lab:** *Modifying Ethernet Frames* with **Scapy** (a Python library for manipulating network traffic).

> **Memorize — Module 1**
> - Ethernet header = **Dest, Src, Type** (+ FCS trailer). MAC = **48 bits**; broadcast = **FF:FF:FF:FF:FF:FF**.
> - EtherType **0x0800 = IPv4**, **0x0806 = ARP**.
> - Even parity bit = XOR of data bits; parity misses **2-bit** errors.
> - CRC: append r zeros, XOR-divide by G, remainder = R; receiver remainder 0 = OK. Detects burst ≤ 32, any odd number of errors; computed on the fly; **detect only, no correction**.
> - Partitioning = fair but **wastes the channel when nodes are idle**.
> - **CD = read while sending (Ethernet); CA = wait for ACK (Wi-Fi)**. Backoff = **random + exponential**.
> - Switch: learn source port; known dest → that port; **unknown or broadcast → flood**.

---

<a id="module-2"></a>
## Module 2 — Network Layer

### 2.1 Why a network layer?
- Link-layer assumptions break down when nodes are:
  - **On different link types** (5G vs Ethernet: different framing and error handling).
  - **Under different admins** (company 1 doesn't want company 2's broadcasts; they may not agree on access rules).
  - **In different locations** (MAC lookups, learning and broadcast don't scale to billions of devices worldwide).
- **Network layer questions:**
  1. Address nodes scalably across networks.
  2. Forward to a destination scalably.
  3. Coordinate networks so each can reach beyond its boundary.
  4. Assign addresses and map them to link-layer addresses.
  5. Troubleshoot beyond your own network.

### 2.2 The Internet Protocol (IP)
- **Internetworking:** communication between **multiple heterogeneous networks**, via a **router at the edge of each network**. The 1974 **Cerf & Kahn** paper ("A Protocol for Packet Network Intercommunication") called it a **gateway**.
- **Service model: best effort.**
  - **Delivery:** packets may be dropped.
  - **Timing:** no guarantee on delivery time.
  - **Order:** packets may be reordered.
  - **Why:** it interconnects different link layers, so it has to be the **lowest common denominator**.
- **Message names:** link = **frame**, network = **packet**, transport = **datagram**, application = **message**.
- **Addressing:** **each interface** gets an IP address.
  - **IPv4:** **32 bits**, dotted decimal (`192.168.0.1`), each number 0–255.
  - **IPv6:** **128 bits**, eight 16-bit **hextets** separated by colons.
    - Drop leading zeros: `2001:0db8:0000:0000:0000:ff00:0042:8329` → `2001:db8:0:0:0:ff00:42:8329`.
    - Replace one run of consecutive all-zero hextets with `::` → `2001:db8::ff00:42:8329`.
- **Structure adds scalability.** MAC addresses are **flat** (one table entry per destination doesn't scale). IP adds structure so devices can be **grouped**.
  - **Subnet:** a set of consecutive IP addresses. **High-order bits are the same** (the network); **low-order bits identify the host**.
  - **CIDR** ("cider", **Classless Inter-Domain Routing**): `a.b.c.d/x`, also called a **prefix**. `x` = number of high-order bits that are fixed; use 0 for the low-order bits. `2.1.1.0/24` covers 2.1.1.0–2.1.1.255.
  - **Aggregation (hierarchy):** `2.1.0.0/24` + `2.1.1.0/24` = `2.1.0.0/23` (one fixed bit becomes part of the host ID). This is a key reason the Internet scales.
  - **Rule:** 2^k contiguous, aligned /n prefixes aggregate into one /(n−k). Example: 172.16.0.0/16 + 172.17.0.0/16 = 172.16.0.0/15.
  - **Does a prefix cover an address?** Compare the first x bits. Example: 172.20.9.9 is in 172.20.8.0/21 (172.20.8.0–172.20.15.255) but not in 172.20.0.0/21 (172.20.0.0–172.20.7.255).
- **IP header fields to know:** Source, Destination, **TTL** (IPv4) / **Hop Limit** (IPv6), **Protocol** (IPv4) / **Next Header** (IPv6), **header checksum** (IPv4).
  - **TTL** handles forwarding loops: each hop **decrements** it, and the packet is **dropped at 0**.
  - **Protocol** says how to interpret the payload, like Ethernet's Type field (1 = ICMP, 6 = TCP, 17 = UDP).

### 2.3 Router data plane
- **Router roles:** (1) **gateway** between networks; (2) **forwarding** traffic toward a destination.
- **Forwarding vs routing** (often confused):
  - **Forwarding = data plane:** directs a packet to an output port using the **forwarding table**.
  - **Routing = control plane:** computes paths by **coordinating with neighbors** and **creates the forwarding table**.
- **Life of a packet:** received at the ingress **line card** → look up dest in the **forwarding table** → send over the **switch fabric** to the output port → output line card transmits.
  - The control plane runs on the **route processor** (software on a general-purpose CPU).
- **Line card processing:** lookup IP → **update header** (e.g., decrement TTL) → **queue in the packet buffer** → transmit.
  - **Key challenge: fast lookup.**
- **Ethernet lookup = exact match** on a flat MAC. **IP lookup = prefix match**, and prefixes can **overlap**.
- **Longest Prefix Match (LPM):** pick the **most specific** (longest) matching prefix.
  - 11.11.11.11 matches both 11.11.0.0/16 and 11.11.11.0/24 → use the **/24**.
  - Overlaps arise when an org carves out part of its block for another location, and from aggregation.
- **Tries** for lookup: a search tree; **left = 0, right = 1**. Store each prefix at the node where its bits end.
  - To look up, walk the address's bits, remembering the **last prefix node passed**; stop when there's no child. The last prefix seen is the longest match.
  - Example: P1 = 100\*, P2 = 11\*, P3 = 1101\*. Lookup 11001111: 1 → 1 (P2) → 0 → 0 (no child) → **P2**.
- **Switch fabric generations:**

| Gen | Design | Limitation |
|---|---|---|
| 1 | **Switching via memory**: NIC → memory → CPU looks up → memory → output NIC | **CPU processing** (and extra transfers) |
| 2 | **Switching via bus**: each line card has its **own forwarding table**; packet goes over the system bus straight to the output card | **Only one packet at a time** on the bus |
| 3 | **Switching fabric** (e.g., **crossbar**): each input connects to 0 or 1 output per timeslot; **many packets in parallel** | Needs **special hardware** (used in big core routers) |
| 4 | Back to **switching via memory in software**: faster CPUs, more cores, cloud/virtualization | Complements Gen 3 rather than replacing it |

- **Crossbar vs bus:** a crossbar can **switch multiple packets simultaneously**.
- Further topics mentioned: **link scheduling** (round robin, weighted, priority), **drop policies** under congestion (**drop-tail**, **RED: Random Early Detection**), **traffic shaping** (rate-limit to a policy).

### 2.4 Routing overview (control plane)
- **The key job of the control plane is calculating the forwarding table.** The network is a **graph**.
- **Routing protocols are distinguished by three questions:**
  1. **What does each router know?** (whole topology or just its neighbors)
  2. **What does each router exchange?** (everything it knows or just what it selected)
  3. **What calculation is performed?** (shortest path or locally best)
- **Decentralized control plane** = traditional routing protocols:
  - **Intra-domain** (within one administrative domain): **link-state, OSPF**.
  - **Inter-domain** (between organizations): **path-vector, BGP**.
- **Centralized control plane = SDN (Software-Defined Networking):** a **controller** knows the entire topology, computes forwarding for the whole network, and installs entries into routers through an **API**. **Routers become just the data plane.**

### 2.5 Link-state routing: OSPF
- OSPF is widely used **within** networks (LANs/intra-domain). The specs are RFCs (OSPFv2 RFC 2328, 244 pages; OSPFv3 for IPv6, RFC 5340).
- **Process:** (1) each router **peers with neighbors**; (2) **link states are flooded to the entire network**; (3) each router **calculates shortest paths**.
- **Link state** = the state of each interface: connected to **another router** (who the neighbor is) or to a **network of end hosts** (which prefixes are reachable).
- Routers send **link state updates** with **all the link states they know**, store them in a **link state database**, and keep exchanging until **convergence**: every router has the **same database = the entire topology**, and they all agree.
- **Calculation: Dijkstra's shortest path** from itself to every node → **shortest-path tree** → forwarding table (destination → first-hop link). Links can have **costs** (bandwidth, latency, etc.).
  - Dijkstra: repeatedly pick the **lowest-cost unvisited node**, add it, update its neighbors' costs; ties can go either way.
- **Changes** (new neighbor, link/node failure, new prefix) are link state changes → send updates until convergence → recompute → update the forwarding table.
- **OSPF's answers to the three questions:**
  - **Knows:** all link states in the network (**the entire topology**).
  - **Exchanges:** **all link states it knows about**.
  - **Calculates:** **shortest path**.

### 2.6 Path-vector routing: BGP
- **BGP-4** (Border Gateway Protocol, RFC 4271) coordinates **Autonomous Systems (ASes)**, independently run networks (AT&T, Comcast, a university). **It's the main protocol of the Internet.**
- **Process:** peer with neighbors; on any route update, failure or config change → update the **routing table (RIB)** → compute the **locally best route** for that prefix → **if it changed, tell neighbors**.
- **Route** = a path to a prefix; **path** = a **sequence of ASes**.
- **Example:** AS4 announces "D via AS4" to AS2 and AS3. They each add themselves and re-announce ("D via AS2 AS4", "D via AS3 AS4"). AS2 knows two paths and chooses the shorter direct one.
  - You don't announce a route back to the neighbor you learned it from.
- **Failures:** if the AS3–AS4 link fails, AS3 sends a **WITHDRAW** for "D via AS3 AS4", picks its backup (via AS2), and sends an **UPDATE** "D via AS3 AS2 AS4" to its other neighbors.
- **iBGP vs eBGP:** **eBGP** runs between routers in **different** ASes; **iBGP** runs **inside** an AS to share externally learned routes among its own routers.
- **How an AS learns its own prefixes:** an internal protocol (OSPF) or **static routes** configured on the router.
- **Business relationships:**
  - **Provider:** is paid to transit traffic.
  - **Customer:** pays a provider to carry its traffic.
  - **Peer:** financially neutral; two similar-sized ASes carry each other's traffic for free.
  - **Policy example:** a customer connected to two providers shouldn't announce routes learned from one provider to the other (or it would carry their transit traffic for free). BGP uses **input and output filters (policies)**.
- **Processing pipeline:** peer sessions (TCP) → per-peer tables → **input policy filters** → **best route selection** → **local RIB** → **forwarding table** → **output policy filters** → announce.
- **Best path selection, in order:**
  1. **Local preference** (highest; numeric value set by admin policy)
  2. **Shortest AS path length**
  3. **MED (Multi-Exit Discriminator)**: a neighbor AS says which of several exit points it prefers you use
  4. **eBGP over iBGP**
  5. **Shortest IGP path cost** (exit your own network as quickly as possible: "hot potato")
  6. **Router ID** (arbitrary tiebreaker)
- **BGP's answers to the three questions:**
  - **Knows:** **only its neighbors and the routes it has received**.
  - **Exchanges:** **its best route for each prefix** (subject to filtering).
  - **Calculates:** the **best local decision** (including business policy), so **neighbors may pick different best routes**.

**OSPF vs BGP at a glance**

| Property | OSPF | BGP |
|---|---|---|
| Knows entire topology | ✔ | ✘ |
| Handles device failures | ✔ | ✔ |
| Decentralized | ✔ | ✔ |
| May pick a different best route than its neighbor | ✘ (all compute the same shortest paths) | ✔ (local policy) |
| Main protocol of the Internet | ✘ | ✔ |
| Exchanges *all* routes it knows | ✔ (all link states) | ✘ (only best route per prefix) |
| Scope | Intra-domain | Inter-domain |

**Lab:** *BGP-Like Sim*: a simplified BGP router in Python that accepts **route announcements and withdrawals**, stores them in a **RIB**, builds a **FIB**, and implements **longest prefix match**.

### 2.7 Mapping IP addresses
- **Allocation:**
  - **IANA** (Internet Assigned Numbers Authority) does **global** coordination.
  - **RIRs** (Regional Internet Registries, e.g., **ARIN** in North America) manage regional allocation. IPv4 is scarce, so you must justify requests.
  - **ISPs** own large blocks and hand addresses to customers.
- **ARP (Address Resolution Protocol): IP → MAC on a LAN.**
  - Communication at the link layer uses MACs, but hosts usually know the destination's IP (or name).
  - Check the **ARP table** first. On a miss, **broadcast** "I'm looking for [IP], what's your MAC?" (dest FF:FF:FF:FF:FF:FF). **Only the owner replies** with its MAC.
  - Off the LAN, you ARP for the **next-hop router's** MAC, not the final destination's.
- **Assigning IPs:**
  - **Static:** `sudo ip addr add 10.1.1.10/24 dev eth0` (the /24 also adds a local route for that subnet).
  - **Dynamic: DHCP** (Dynamic Host Configuration Protocol); the server is often the gateway router. Exchange: **Discover → Offer → Request → Ack** (there may be several servers making offers).

### 2.8 Troubleshooting tools
- **ping:** is a server up and reachable, and what's the **round-trip time**?
  - Uses **ICMP Echo**: IP header **Protocol = 1 (ICMP)**; the ICMP message is the IP payload. An ICMP **request** goes to the destination, which sends an ICMP **reply**.
  - (The slides label the types "0 = request, 1 = reply". In the real ICMP standard, Echo Request = **8** and Echo Reply = **0**. If an exam question uses the lecture's numbering, go with the lecture.)
- **traceroute:** the **path of routers** to a destination (spots latency and reachability problems, like traffic unexpectedly going via Europe).
  - Send packets with **TTL = 1, 2, 3, …**. Each router where TTL hits 0 replies with **ICMP Type 11 (Time Exceeded)**, revealing its address.
  - Routers that don't reply show up as `* * *`.
- **iperf3:** **maximum throughput** between client and server. `iperf3 -s` on the server, `iperf3 -c <server>` on the client. It ramps up and reports per-interval bitrate, retransmits and cwnd.

> **Memorize — Module 2**
> - IP = **best effort** (drop, delay, reorder). IPv4 32 bits, IPv6 128 bits.
> - CIDR prefix /x = first x bits fixed. Aggregate 2^k aligned /n's → /(n−k).
> - **Forwarding (data plane, uses forwarding table)** vs **routing (control plane, builds it)**. Data plane looks up the dest IP in the **forwarding table**.
> - **LPM:** most specific prefix wins.
> - Crossbar beats bus: **parallel transfers**.
> - **OSPF:** knows all, floods all link states, Dijkstra. **BGP:** knows neighbors, sends best route per prefix, local policy; the Internet's protocol.
> - BGP selection: **LocalPref > AS path length > MED > eBGP > IGP cost > router ID**.
> - **ARP** = IP→MAC (broadcast request, unicast reply). **DHCP** = Discover/Offer/Request/Ack.
> - **ping = ICMP echo** (IP protocol 1). **traceroute = increasing TTL + ICMP Time Exceeded (type 11)**. **iperf3 = max throughput**.

---

<a id="module-3"></a>
## Module 3 — Transport Layer

### 3.1 Problems the transport layer solves
1. **Multiplexing:** many applications on one host. Which one is a packet for?
2. **Reliable transfer:** IP is best effort, but applications want the **same data in the same order**.
3. **Congestion control:** congestion causes drops. Senders should detect it and back off.

### 3.2 Multiplexing
- **Network layer = host-to-host** communication; **transport layer = process-to-process**. (This is the defining difference for *all* protocols.)
- Adds another header with **source and destination port numbers** (e.g., HTTP on port **80**, SSH on port **22**).
- **Connectionless:** the host demultiplexes with **dest IP + dest port** only. No setup; the only state is the (IP, port) → process mapping.
- **Connection-oriented:** a connection is identified by the **4-tuple** (src IP, dst IP, src port, dst port), often called a **5-tuple** with the protocol. Requires **establishment first**, and the host keeps **more state**. One process can have many connections.
- **UDP:** **connectionless**. Simple mux/demux, and it **inherits IP's best-effort service**. Header = src port, dst port, length (+ checksum).
- **TCP:** **connection-oriented**. Provides an **in-order, reliable byte stream**.

### 3.3 Reliable transfer (TCP)
**Why packets get lost or reordered**
1. **Bit errors** on links: Ethernet's CRC detects them but can't fix them, so the frame is **dropped**.
2. **Congestion** in router buffers: if the incoming rate exceeds the outgoing rate (e.g., several 10 Gbps inputs all heading to one 10 Gbps output), the buffer fills and packets are **dropped**.
3. **Reordering:** routing is dynamic, so a new, shorter path can let later packets overtake earlier ones (**different packets can take different paths**).

**TCP mechanisms**
- **Sequence number:** which bytes of the stream this datagram carries (the **first byte's** position). Segments of 100 bytes → seq 0, 100, 200, 300.
- **Acknowledgement number:** the **next byte the receiver expects**, i.e., everything before it was received. ACK 200 = bytes 0–199 arrived.
  - The **sequence and ACK numbers are the key header fields for reliability.**
- **Retransmission:** the sender resends unacknowledged bytes. It must **keep them buffered until ACKed**.
- **When to retransmit:**
  - **Timeout:** wait about **EstimatedRTT plus a safety margin**.
    - `EstimatedRTT = (1−α)·EstimatedRTT + α·SampleRTT` (an exponentially weighted moving average that smooths spikes without ignoring real changes).
    - Too short a timeout causes unnecessary retransmissions.
  - **Duplicate ACK:** receiving the **same ACK again** is a strong (not certain) sign a packet was dropped rather than just reordered.
- **Message exchange diagrams:** time runs down; a diagonal arrow = a message plus its transmission time; an **X = a dropped packet**; crossed arrows = reordering.

**Flow control** (protects the **receiver**)
- Problem: the sender can transmit faster than the receiver can accept (e.g., 100 Gbps sender, 10 Mbps receiver).
- The receiver **advertises its free buffer space** in the **receive window (rwnd)**, the **Window Size** field of the TCP header.
- The sender keeps **unACKed (in-flight) data ≤ rwnd**.
- **Flow control is needed when the sender is faster than the receiver**. Network bottlenecks are handled by congestion control instead.

### 3.4 TCP connection establishment
- **Why connections?** Both sides must store **state**: send/receive buffers, rwnd, ACK numbers, estimated RTT. Resources are limited, so both sides agree first.
- Uniquely identified by **(src IP, dst IP, src port, dst port)**.
- **Flags:** **SYN** = synchronize sequence numbers (start of connection); **ACK** = the ACK number field is valid.
- **Three-way handshake:**
  1. A → B: **SYN**, seq = x (A picks its starting sequence number; e.g., 1000)
  2. B → A: **SYN + ACK**, seq = y (B's own start, e.g., 600), ack = **x+1** (1001)
  3. A → B: **ACK**, ack = **y+1** (601)
- **Losses during the handshake:**
  - Packet 1 lost → A gets no ACK → A **resends the SYN** (using a default timeout, since there's no RTT history yet).
  - Packet 2 lost → same as above; A resends the SYN, B re-ACKs.
  - Packet 3 lost → B gets no ACK for its SYN → B **resends packet 2**; A ACKs it again.

### 3.5 Congestion control (general)
- **How congestion happens:** the send rate out of a link is lower than the receive rate (bandwidth mismatch, e.g., 10 Gbps in → 1 Gbps out; or many inputs → one output). The buffer grows every time unit.
- **Effects:** routers **drop** packets → hosts **retransmit** → retransmissions feed the already congested router.
- **Congestion collapse:** as **offered load** increases past a point, **goodput** (unique data actually delivered end to end) **drops sharply**. More is sent, less arrives.
- **Goal:** operate just before collapse, **efficient** (the full link bandwidth is used) and **fair** (each flow gets an equal share). On a Host-1 vs Host-2 throughput plot, the ideal is where the "efficient" line meets the "fair" line; above the efficient line is congestion.
- **Approaches:**
  - **Reservation:** reserve bandwidth on each router, in the network.
  - **Feedback and adjust:** hosts detect congestion and adjust the send rate. **TCP uses this**, with **implicit feedback** and **window-based** adjustment.

### 3.6 TCP congestion control
- **Feedback signals:**
  - **Packet loss:** **timeout**, or **duplicate ACK**.
  - **Packet delay:** rising RTT samples mean queues are growing.
  - **ECN (Explicit Congestion Notification):** explicit, but needs router support. TCP is usually assumed to use implicit signals.
- **Congestion window (cwnd):** **internal state on the sender** (not a header field). It limits how much data can be **in flight**:
  - `LastByteSent − LastByteAcked ≤ cwnd`
  - Contrast with **rwnd**, which is an explicit header field from the receiver. The sender must stay under **both**: in flight ≤ min(cwnd, rwnd).
- **AIMD (Additive Increase, Multiplicative Decrease):** shown to optimize congested flow rates network-wide and to be **stable**.
  - **Additive increase:** +**1 MSS per RTT** until loss (in practice a fraction per ACK: `Increment = MSS × (MSS / cwnd)`). cwnd 1 → 2 → 3 → 4.
  - **Multiplicative decrease:** **cut by a factor (e.g., half)** on loss. Multiplicative decrease is a **necessary condition for stability**.
  - The result is a **sawtooth**: AI probes for bandwidth, MD rapidly corrects.
  - **Why it converges to fairness:** the increase is equal for everyone, and the decrease is proportional to the send rate (the bigger sender drops more), so flows converge toward the fair line.
- **Slow start:** additive increase is too slow to find capacity, so **increase cwnd on every ACK** (instead of every RTT). That's **exponential growth** up to the **slow-start threshold**, then switch to linear (additive) increase.

**Lab:** *TCP: Find Max Bytes in Flight*. Use Scapy on a TCP session capture to find the maximum bytes in flight (highest sequence sent vs what's been ACKed), handling reordering and loss.

> **Memorize — Module 3**
> - Network = **host-to-host**; transport = **process-to-process** (ports).
> - UDP connectionless (best effort); TCP connection-oriented (reliable, in-order stream; 4-tuple).
> - Frame / **packet** / **datagram** / message.
> - Reliability = **seq #, ACK # (next byte expected), retransmit (timeout or dup ACK)**. Flow control = **rwnd** (receiver buffer).
> - Handshake: SYN(x) → SYN-ACK(y, ack x+1) → ACK(y+1).
> - **Congestion collapse = goodput falls as offered load rises.**
> - **cwnd = in-flight limit (sender state)**; signals = **timeout + duplicate ACK** (+ delay, ECN).
> - AIMD: +1 MSS/RTT, halve on loss → sawtooth, fair and stable. **Slow start** = exponential until threshold.

---

<a id="module-4"></a>
## Module 4 — Application Layer

### 4.1 Problems the application layer solves
1. **Addressing:** IP addresses aren't human readable, and one IP is one machine (YouTube has servers in Denver, New York, …) → **DNS**.
2. **Programming:** the link/network/transport protocols are concepts implemented by the **OS network stack**. How do programs use it? → **sockets**.
3. **Application-level protocol:** what processes say to each other and how data is encoded → **HTTP, gRPC**.

### 4.2 DNS (Domain Name System)
**Addressing at different layers**

| | Host name | IP address | MAC address |
|---|---|---|---|
| Example | www.cs.colorado.edu | 128.138.7.156 | 00-15-C5-49-04-A9 |
| Form | Hierarchical, **human readable**, variable length | Hierarchical, machine readable, 32 bits (IPv4) | **Flat**, machine readable, 48 bits |
| Read by | Humans, hosts | IP routers | Switches in a LAN |
| Top-level allocation | Domain, by a **registrar** (e.g., for .edu) | Variable-length prefixes, by ICANN, RIR or ISP | Fixed blocks, by IEEE to vendors |
| Low-level allocation | Host name, by local admin | Interface, by admin or DHCP | Interface, by vendor |

- **Hierarchical names:** top-level domains (.edu, .com, .gov, .mil, .org) → domains (princeton.edu) → subdomains (cs.princeton.edu).
- **DNS** = a **distributed database** mapping host names ↔ IP addresses. It's an application-layer protocol that runs **over UDP**.
- **Resource records (RR):** `(name, value, type, ttl)`.
  - **Type A:** name = hostname, value = **IP address**.
  - **Type NS:** name = domain, value = **hostname of the authoritative name server** for that domain.
- **Lookup flow:** host → **local DNS server (resolver)**, which may answer from its **cache**. Otherwise:
  - resolver → **root server** → returns the **TLD server** (.edu)
  - → **TLD server** → returns the **authoritative server** (dns.cs.umass.edu)
  - → **authoritative server** returns the **A record** → resolver replies to the host.
  - The intermediate replies are NS referrals; the final one is an A record.
- **Recursive vs iterative:**
  - **Recursive:** the server does the whole lookup for you (host → local resolver).
  - **Iterative:** the server replies "ask this server next" (resolver → root/TLD).
  - **When a server doesn't know the answer**, it responds with the name of a server it knows is **authoritative for part of the name**.
- **Root servers:** critical infrastructure. A known set of **13**.
  - **Their IP addresses are configured (hard-coded) in resolvers**; there's no protocol to discover them.
  - Each is **replicated** for redundancy, often via **anycast**.
- **Anycast:** advertise the **same IP prefix from multiple locations** via BGP. Routers pick the shortest AS path, so clients reach the closest instance; if one site fails, BGP reroutes to another.
- **DNS message format:** the **query and reply have the same format**, all in a single UDP packet.
  - **Identification:** the reply matches the query's ID.
  - **Flags:** query vs reply, etc.
  - **Counts:** # questions, # answers, # authority RRs, # additional RRs.
  - **Questions:** the name being looked up (e.g., www.google.com, type A).
  - **Answers:** the RRs answering the query (with the address).
  - **Authority:** if the resolver doesn't know, the **authoritative name server** for the name or part of it (e.g., ns2.google.com).
  - **Additional:** extra helpful records, e.g., **the IP of that name server**, to save another lookup.

### 4.3 Socket programming
- **Berkeley sockets**, from **1983**, are still the standard. A **socket** is an abstract representation of the **local endpoint** of a network communication path. The app writes into a socket; the OS handles everything below.
- **Client** = initiates the connection; **server** = accepts connections.
- **TCP server:** `socket()` → `bind()` → `listen()` → `accept()` → `recv()`/`send()` … → `close()`
- **TCP client:** `socket()` → `connect()` → `send()`/`recv()` … → `close()`
- **What each call does:**
  - **socket():** the OS creates a socket and returns a **handle (file descriptor)**. `int socket(int domain, int type, int protocol)`: domain **PF_INET / PF_INET6**; type **SOCK_STREAM (TCP)** or **SOCK_DGRAM (UDP)**.
  - **bind():** tells the OS which **address (IP + port)** to use (e.g., port 80 for a web server; a specific IP, localhost, or all interfaces).
  - **listen():** tells the OS the socket is willing to accept incoming connections (`backlog` = queue size).
  - **accept():** **blocks** until a client connects; returns the client's address and a new socket for the connection.
  - **connect():** the client asks the OS to connect to the server's IP and port. **The OS performs the TCP three-way handshake**, then `accept()` returns.
  - **send()/recv():** exchange data per the application protocol.
  - **close():** closes the socket (sends the TCP close). The peer's `recv()` returns a special value (0 bytes) so it knows to close too.
- Python's `socket` module mirrors the same calls (the lecture shows a full Python client/server).

**Lab:** *Socket Programming*: a simple **RPC system** where you pack data so client and server messages match.

### 4.4 Application protocols and HTTP
- **An application-layer protocol defines:**
  - **Message syntax:** what fields there are and how they're delineated.
  - **Types of messages:** request/response, streaming, …
  - **Message semantics:** what the fields mean.
- **Syntax requirement:** the receiver extracts exactly what was sent. Design goals trade off **debuggability, bandwidth, processing**. Application data → presentation encoding → messages → decoding.
- **Presentation taxonomy:**
  - **Data types:**
    - **Base types** (integers): number of bits? **byte order (big- vs little-endian)**?
    - **Flat types** (arrays, structs): **padding**? **length**?
    - **Complex types** (trees, linked lists with pointers): must be **serialized/flattened**.
  - **Conversion strategy:**
    - **Canonical intermediate form:** the sender converts to an agreed format and the receiver converts from it (two conversions).
    - **Receiver makes right:** the sender sends its native form plus a description, and the receiver converts only if needed. Fewer conversions; works well when the infrastructure is **homogeneous**.
  - **Tagging:**
    - **Untagged:** agree in advance on type, length and location (like the TCP header).
    - **Tagged:** the message includes tags (type, length); e.g., JSON.
- **HTTP:** the web's protocol. Client/server exchange of **web objects** (HTML, JPEG, audio). **Request/response.** **Stateless:** the server keeps no memory of past requests.
  - URL `http://www.example.com/blah/blah.gif` = protocol + host name + path.
  - **Format:** `START_LINE <CRLF> MESSAGE_HEADER <CRLF> <CRLF> MESSAGE_BODY <CRLF>`.
    - Request start line = method + path + version.
    - Response start line = version + **status code**. Headers include **Content-Type** (e.g., `text/html`).
  - **Methods:** **GET** (retrieve the object at the URL), **POST** (give information to the server), **PUT** (store a document at the URL), HEAD (metadata only), DELETE, OPTIONS, TRACE (loopback), CONNECT (for proxies).
  - **Data formats:**
    - **HTML** describes appearance (`text/html`).
    - **JSON** passes information in APIs (`application/json`).
- **JSON:** **name:value pairs**. Keys are **strings**; values can be a **string, number, object, array, boolean or null**. `{}` holds an **object** (a list of name:value pairs); `[]` holds an **array** (a list of values).
- **HTTP versions:**

| Version | Key addition |
|---|---|
| 1.0 | First major version: a new TCP connection per object (handshake RTT + request RTT + transfer, every time) |
| **1.1** | **Persistent connections**: reuse the TCP connection (saves the handshake and the slow-start ramp) |
| **2.0** | **Multiplexed requests**: many requests in parallel on one connection |
| **3.0** | **HTTP over QUIC** (instead of TCP); QUIC runs over UDP and incorporates TLS |

### 4.5 gRPC
- **RPC (Remote Procedure Call), 1981** (Bruce J. Nelson): extends a **local procedure call** (`z = add(11, 22)`) to a remote server.
  - **Client stub** encodes the arguments → network → **server stub** calls the real function → result encoded and returned → client stub returns it.
  - It defines both the **data encoding** and the **communication protocol**.
- **Useful for microservices:** e.g., a ride-share app split into trip management, billing, notifications, etc. These can use REST/JSON or gRPC.
- **gRPC = modern RPC:**
  - Based on **protobufs (Protocol Buffers)**, a **binary** serialization format: far more compact than JSON (the lecture example is **55 bytes in JSON vs 20 in protobuf**) because field names aren't sent.
  - Uses **HTTP/2** (persistent, pipelined/multiplexed messages).
- **Workflow:**
  1. Write a **`.proto`** file defining the **message** structures and the **service** functions:
     ```proto
     service Echo { rpc echo(Message) returns (Message) {} }
     message Message { string message = 1; }   // "= 1" is the field's order/number
     ```
  2. Run the **protoc compiler** (`python -m grpc_tools.protoc --proto_path=. ./simple.proto --python_out=. --grpc_python_out=.`). It **generates libraries of stubs** that serialize/deserialize data, in many languages (here `simple_pb2.py`, `simple_pb2_grpc.py`).
  3. **Server:** import the libraries and **implement** the functions (subclass the generated `EchoServicer`).
  4. **Client:** import the libraries, open a channel, create a **stub** (`EchoStub`), and call the functions.
- **Four API types:** **unary** (one request, one response; used in the demo) plus **server streaming, client streaming, bidirectional streaming**.
- **JSON vs protobuf:** the biggest difference is **JSON is textual, protobuf is binary**.

> **Memorize — Module 4**
> - DNS = distributed DB over **UDP**. **A** = name→IP; **NS** = domain→authoritative server name.
> - Unknown → reply with a **referral to an authoritative server**. **Root server IPs are hard-coded** in resolvers; 13 roots, replicated via **anycast**.
> - DNS sections: **question / answer / authority / additional**; the reply ID matches the query ID.
> - Server: **socket, bind, listen, accept**; client: **socket, connect**. SOCK_STREAM = TCP, SOCK_DGRAM = UDP.
> - App protocol = **syntax + message types + semantics**.
> - HTTP is **stateless**; GET retrieves, POST/PUT send. 1.1 persistent, 2 multiplexed, 3 QUIC.
> - JSON values: string, number, object, array, boolean, null.
> - protoc **generates stubs** for serializing/deserializing in many languages. JSON **text** vs protobuf **binary**.

---

<a id="module-5"></a>
## Module 5 — Network Security

### 5.1 Overview
- Communication crosses media the parties don't control, so attackers can strike at many points.
- **CIA triad:**
  - **Confidentiality:** only intended parties can see the information.
  - **Integrity:** data hasn't been modified or corrupted.
  - **Availability:** information can be accessed.
- **The three course examples:**

| Layer | Threat | Solution |
|---|---|---|
| Network **data plane** | Company with 2 sites; an eavesdropper sees secret traffic | **IPsec** |
| Network **control plane** | Attacker injects fake BGP paths ("I host YouTube's IPs") to attract traffic: deny service, change messages, inspect traffic | **RPKI** |
| Transport/application | Attacker pretends to be your bank or inspects/modifies traffic (e.g., public Wi-Fi) | **TLS / HTTPS** |

### 5.2 Basics
**Terminology:** Alice and Bob communicate; the attacker is **Eve** (eavesdropper; also Mallory, Trudy). **Plaintext** m → encryption with key K_A → **ciphertext** K_A(m) → decryption with K_B → m = K_B(K_A(m)).

**Symmetric encryption (shared key)**
- The same key encrypts and decrypts. **Fast**, but you **need a way to share the key**.
- Trivial example: **XOR** each 8-bit block with an 8-bit key. 'E' = 0x45 = 0100 0101, XOR 1101 0111 → 1001 0010 = 0x92. XOR again with the key → 0x45.
- Real example: **AES**, a NIST standard since **2001**, with **128/192/256-bit** keys (substitutions, row shifts, mixing).

**Asymmetric encryption (public/private key pair)**
- Bob generates a key pair. Anyone can encrypt with **Bob's public key**; **only Bob's private key** decrypts.
- **Slow and suited to short messages**, but **easy to distribute the key** (it doesn't matter who has the public key).
- **RSA:** public key (n, e), private key (n, d).
  - Encrypt **c = m^e mod n**; decrypt **m = c^d mod n**. m is the bit pattern read as an integer, with m < n.
  - Example: p = 5, q = 7 → n = 35, z = (p−1)(q−1) = 24; **e = 5** (relatively prime to z), **d = 29** (ed − 1 divisible by z). m = 12 → 12^5 = 248832 → mod 35 = **17**; 17^29 mod 35 = **12**.
- **Negatives:** symmetric needs a **shared secret key**; asymmetric is **slow**.
- **Hybrid (session key):** Alice picks a **session key**, encrypts it with **Bob's public key**, and Bob decrypts it with his private key. Now both share a secret and use fast **symmetric** encryption for the long messages.

**Integrity**
- **Hash:** a **fixed-length digest** of a variable-length message. It should be **computationally infeasible to find another message with the same digest**.
  - Used to check **software downloads**. It assumes the **digest itself is unmodified**.
- **HMAC (hash-keyed Message Authentication Code):** removes that assumption. Sender and receiver share a secret key; MAC = **H(key ∥ H(key ∥ message))**. An attacker can't modify the message and recompute a matching MAC.
- **Digital signature:** proves (1) the message is unchanged and (2) it came from Alice.
  - Alice hashes the message and **encrypts the hash with her private key** (that's signing).
  - Bob decrypts with **Alice's public key** and compares against his own hash of the message.

**Authentication**
- Problem: is this public key **really Alice's**? ("Not Alice" could hand out her own key.)
- **Digital certificate:** **binds a public key to an identity** (name, organization, country…; what counts as identity depends on context).
  - Problem: an attacker could alter the certificate.
- **Certificate Authority (CA):** a trusted party **verifies the identity** and **signs the certificate with the CA's private key**. Anyone can verify it with the **CA's public key**, which must already be trusted (e.g., **pre-installed in browsers**).

### 5.3 IPsec (network data plane)
- **How eavesdropping happens:**
  - **Wi-Fi is broadcast.**
  - A network provider is compelled by a government, has a rogue employee, or has a **compromised router**.
  - A routing **misconfiguration** sends traffic the wrong way.
  - Deliberate **route hijacking**.
  - No single fix covers all of these, so solutions get combined.
- **IPsec creates an encrypted tunnel** between two endpoints (a whole **site** or a single machine). It's commonly used for **VPNs**.
- **Setup** uses **ISAKMP/IKE** (Internet Security Association and Key Management Protocol / Internet Key Exchange). SA = **Security Association**.
  1. **SA-1 parameter exchange:** the initiator lists what it supports in **4 categories**: **encryption** (DES, AES…), **hash** (SHA, MD5…), **DH group** (1, 2, 5…: security vs CPU cost), **authentication** (pre-shared key, certificate…). **The responder chooses.**
  2. **Diffie-Hellman key exchange:** IPsec uses **shared (symmetric) keys** for encryption and integrity, and DH is how they **securely establish a shared key**.
  3. **Authentication:** each side verifies the other with a **pre-shared key** or a **digital certificate** (as chosen in SA-1). This step is already encrypted with the DH-derived keys.
  4. **SA-2 parameter exchange:** sets up the SA for the actual traffic (possibly different parameters) and the **traffic selectors**: which traffic may use the tunnel, by **5-tuple** (src/dst IP, protocol, src/dst port).
  5. **Encrypted traffic exchange.**
- **Diffie-Hellman:** lets two parties with **no prior knowledge** of each other establish a **shared secret key over an insecure channel**.
  - A picks a prime **p** and base **g** and sends them in the clear.
  - A picks secret **x** and sends h_A = g^x mod p. B picks secret **y** and sends h_B = g^y mod p.
  - A computes **K = h_B^x mod p**; B computes **K = h_A^y mod p**. Both equal **g^(xy) mod p**.
  - Security: given p, g, h_A and h_B, it's **hard to compute K** without x or y.
- **Traffic modes** (pick one from each pair):
  - **AH (Authentication Header):** **integrity only**. **ESP (Encapsulating Security Payload):** **integrity + confidentiality**.
  - **Transport mode** (mostly a single endpoint): keeps the original IP header and protects the payload. **Tunnel mode** (mostly whole sites): encrypts the **entire original packet, including its IP header**, and adds a **new outer IP header** between the site gateways.
- **What keeps IPsec confidential:** **encryption**. (Integrity comes from HMAC; authentication from certificates or pre-shared keys.)
- **Software:**
  - Vendor routers and firewalls (Juniper, Cisco, Palo Alto).
  - **Cloud VPN services (AWS, Azure, GCP) are IPsec.**
  - Open source: **strongSwan**.
- **Summary:** IPsec combines **HMAC** (integrity), **shared-key encryption** (confidentiality), **Diffie-Hellman** (key exchange), and **public keys or pre-shared keys** (authentication).

### 5.4 RPKI (network control plane)
- **Attack: route (prefix) hijacking.** The real origin AS3 announces 1.1.1.0/24 (path AS2 AS3 seen at AS1); attacker AS4 also announces 1.1.1.0/24 with path AS4. AS1 picks the **shorter** fake path, and traffic flows to AS4. The attacker can then:
  - **Deny service** (drop everything).
  - **Impersonate** the destination (a fake bank page to harvest passwords).
  - **Man-in-the-middle:** forward traffic on to the real destination while eavesdropping, so the victim doesn't notice.
- **Idealized fix:** the prefix owner tells everyone "I own 1.1.1.0/24 and I'm in AS3". BGP still picks the path, but **it must originate in AS3**, so the AS4 route gets thrown out.
- **RPKI (Resource Public Key Infrastructure):**
  - **CAs = the Regional Internet Registries (RIRs)**. They already allocate IP space and have a **business relationship** with owners, so they can verify identity.
  - The certificate binds the **IP prefix (length, max length) + origin AS** to the **owner's public key**.
- **Architecture:**
  1. The prefix owner creates a key pair and sends a certificate request to the RIR (e.g., a web form).
  2. The RIR validates identity and ownership and stores a **ROA (Route Origin Authorization**; the lecture says "Resource Ownership Authorization") in a database.
  3. A **validator** (software on a server in the AS, not the router; e.g., **Routinator**) **downloads the ROAs and verifies the signatures**.
  4. The validator **installs a cache of verified prefix/origin pairs into the routers**. Routers do **no cryptography**.
  5. On each BGP update the router checks the state:
     - **Valid:** the prefix is in the DB and the origin matches.
     - **Invalid:** the prefix is in the DB but the origin doesn't match.
     - **Unknown:** the prefix isn't in the DB (kept for **backward compatibility**, since deployment has taken over a decade and is still incomplete).
- RPKI protects against **route hijacking** (it **validates the origin AS**; it doesn't encrypt anything).

### 5.5 TLS / HTTPS (transport/application)
- **The threat is real.** Visiting chase.com = a DNS lookup, then packets to that IP. An attacker could **modify packets directly**, make **DNS return a wrong IP**, or **hijack BGP** so even the right IP goes somewhere else.
- **TLS** is a layer providing **confidentiality, integrity and authentication**. **HTTPS = HTTP over TLS over TCP**. **HTTP/3 = HTTP over QUIC** (which incorporates TLS) over UDP.
- **Problems with plain HTTP/TCP:** (1) **plaintext**; (2) **no integrity protection**; (3) **no guarantee you're talking to the intended server**.
- **How TLS solves them:**
  - **Symmetric encryption + MACs** with a shared key give confidentiality and integrity. But how do you share the key?
  - **Asymmetric encryption** exchanges the key (encrypt the session key with the server's public key). But how do you trust that public key?
  - **Certificates + CAs** bind identity (domain ownership) to the public key, signed by a trusted source, which gives authentication.
- **Let's Encrypt** (a widely used, open CA) has **no prior business relationship** with requesters, so it verifies **domain ownership with a challenge**: provision a **DNS record** under the domain, or an **HTTP resource at a well-known URI**. If Let's Encrypt can fetch the token, ownership is proven.
- **TLS 1.2 handshake** (it happens **after the TCP three-way handshake**):
  1. **Client Hello:** what the client supports and prefers (ciphers like AES/DES; integrity algorithms like SHA1/MD5).
  2. **Server Hello:** the server's choices.
  3. **Server Certificate:** signed by a CA, binding the domain to the server's public key. The client verifies it with its **installed root certificates**.
  4. **Server Hello Done.**
  5. **Client Key Exchange:** the client picks a **session key** and encrypts it with the **server's public key**.
  6. **Change Cipher Spec** (client): "parameters agreed; the next message is encrypted".
  7. **Finished** (client): a summary/hash of **all handshake messages**, so tampering is detectable.
  8. Server sends **Change Cipher Spec + Finished**. **Communication is now secured.**
  - (TLS 1.3 is more concise, with fewer messages. The course uses 1.2 for clarity.)
- **Mutual TLS (mTLS):** normally **only the client verifies the server**. With mTLS, the **server also verifies the client's certificate**, e.g., **access control for API endpoints**.
  - Browsers implicitly accept any server whose certificate domain matches; servers usually keep an **explicit access control list** of accepted client certificates.

**Lab:** *Network Security Lab: Certificates*. Act as your own CA: create keys, a certificate request and a signed certificate, then serve HTTPS and test with `curl`.

> **Memorize — Module 5**
> - CIA = **Confidentiality, Integrity, Availability**.
> - Symmetric: fast, needs a shared key (AES). Asymmetric: slow, easy key distribution (RSA: c = m^e mod n, m = c^d mod n).
> - Hash = fixed digest; **HMAC** = H(key ∥ H(key ∥ msg)); **signature** = hash encrypted with the **private** key, verified with the public key.
> - **Certificate = identity + public key, signed by a CA** (CA root keys pre-installed).
> - **IPsec:** IKE → SA-1 params → **DH** (shared key) → auth → SA-2 (traffic selectors) → traffic. **ESP = confidentiality + integrity; AH = integrity only.** Tunnel mode wraps the whole packet.
> - **DH:** K = g^(xy) mod p, established over an insecure channel.
> - **RPKI:** RIRs are the CAs; ROAs; validator → router cache; **Valid / Invalid / Unknown**; stops **route hijacking**.
> - **HTTPS fixes plaintext, missing integrity, and unknown server identity.** Let's Encrypt = **token at a well-known URL** (or a DNS record).
> - TLS 1.2: ClientHello, ServerHello, Certificate, HelloDone, ClientKeyExchange, ChangeCipherSpec, Finished (×2). **mTLS** = the server verifies the client too.

---

<a id="quick-reference"></a>
## Quick-Reference Tables

### Layers at a glance

| Layer | Unit | Address | Key protocols / devices | Problems solved |
|---|---|---|---|---|
| Application | Message | Host name | DNS, HTTP, gRPC | Naming, programming interface, app protocols |
| Transport | Datagram | Port | TCP, UDP | Multiplexing, reliability, congestion control |
| Network | Packet | IP address (prefix) | IP, OSPF, BGP, ARP, DHCP, ICMP; routers | Scalable addressing, forwarding, routing |
| Link | Frame | MAC (flat, 48 bits) | Ethernet, 802.11; switches, APs | Framing, error detection, multiple access, LANs |

### Tables in a network

| Table | Where | Maps | Lookup |
|---|---|---|---|
| MAC address table | Ethernet switch | dest MAC → port | Exact match; learned from source MACs |
| Forwarding table (FIB) | Router data plane | prefix → next hop/port | **Longest prefix match** |
| Routing table (RIB) | Router control plane | prefix → all known routes | Best-route selection fills the FIB |
| ARP table | Host OS | IP → MAC | Broadcast ARP request on a miss |

### Windows

| | rwnd (receive window) | cwnd (congestion window) |
|---|---|---|
| Protects | The **receiver** (flow control) | The **network** (congestion control) |
| Lives in | TCP header (Window Size field) | Sender's internal state |
| Set by | Receiver (its free buffer) | Sender (AIMD / slow start) |
| Rule | in-flight ≤ min(rwnd, cwnd) | |

### Security building blocks

| Goal | Mechanism |
|---|---|
| Confidentiality | Symmetric (AES) or asymmetric (RSA) encryption; session keys |
| Key exchange | Encrypt a session key with a public key, or Diffie-Hellman |
| Integrity | Hash (assumes a safe digest), HMAC (shared key) |
| Integrity + origin | Digital signature (private-key sign, public-key verify) |
| Authentication | Digital certificate signed by a CA |

### Where each attack is fixed

| Attack | Fix | CA / trust anchor |
|---|---|---|
| Eavesdropping between sites | IPsec (VPN) | Pre-shared key or certificates |
| BGP route hijack | RPKI | Regional Internet Registries |
| Fake website / Wi-Fi snooping | TLS / HTTPS | Browser root CAs (e.g., Let's Encrypt) |

### Key numbers

| Thing | Value |
|---|---|
| MAC address | 48 bits |
| IPv4 / IPv6 address | 32 / 128 bits |
| Ethernet Type / FCS | 16 / 32 bits |
| EtherType IPv4 / ARP | 0x0800 / 0x0806 |
| IP protocol ICMP / TCP / UDP | 1 / 6 / 17 |
| ICMP Time Exceeded | Type 11 |
| CRC-32 burst detection | ≤ 32 bits |
| DNS root servers | 13 (replicated, anycast) |
| Berkeley sockets | 1983 |
| RPC paper | 1981 |
| AES standardized | 2001 (128/192/256-bit keys) |
| Well-known ports | HTTP 80, SSH 22 |

---

<a id="quiz-map"></a>
## Graded Quiz Coverage Map

These quizzes are graded (8% each), so under the course's Conditional AI policy this guide does **not** answer them. Use this map to find which sections to review before you take each quiz.

| Quiz | Topics it draws on | Review |
|---|---|---|
| **Link Layer** | MAC address formats, Ethernet header fields, even parity, FCS/CRC properties, channel partitioning drawbacks, CSMA/CD vs CSMA/CA collision detection, switch forwarding decisions | §1.2–1.5 |
| **Network Layer** | Prefix aggregation and coverage, IP service model, forwarding-table lookup, longest prefix match, crossbar vs bus, OSPF vs BGP properties, ARP, how ping works | §2.2–2.8, OSPF vs BGP table |
| **Transport Layer** | Network vs transport layer roles, UDP vs TCP, causes of unreliability, reliability header fields, flow control, congestion collapse, congestion signals, cwnd | §3.2–3.6 |
| **Application Layer** | Root server discovery, DNS referrals, socket call order, socket language support, what an app protocol defines, JSON value types, protoc's role, JSON vs protobuf | §4.2–4.5 |
| **Network Security** | Symmetric vs asymmetric drawbacks, IPsec confidentiality, Diffie-Hellman's purpose, RPKI's protection and CAs, HTTPS's fixes, Let's Encrypt validation | §5.2–5.5 |

Once a quiz has been graded, Coursera shows you the feedback. Comparing that feedback with these sections is a good way to find weak spots before the final.
