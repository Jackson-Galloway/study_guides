# ECEA 5370 — Practice Final Exam: Answer Key

## Part A — Multiple Choice

| # | Ans | Explanation | Review |
|---|---|---|---|
| 1 | **b** | The FCS is a trailer because it's the last thing received, so the NIC can check the whole frame on arrival. | §1.2 |
| 2 | **c** | 0x0806 = ARP; 0x0800 = IPv4. | §1.2 |
| 3 | **b** | 1011 0010 has four 1s (even). Odd parity needs an odd total, so add a 1. | §1.3 |
| 4 | **b** | 101110 000 ÷ 1001 using XOR leaves remainder 011. | §1.3 |
| 5 | **b** | The hidden terminal problem: both reach the AP but not each other. That's why CSMA/CA relies on ACKs (and optional RTS/CTS). | §1.4 |
| 6 | **b** | Random backoff breaks the lockstep; it also grows exponentially with repeated failures. | §1.4 |
| 7 | **b** | The switch learns the *source* (A on port 1). The destination is unknown, so it floods. | §1.5 |
| 8 | **d** | The listed options were tracking messages seen, limiting hops, Spanning Tree, and architecting the network. FCS size has nothing to do with loops. | §1.5 |
| 9 | **a** | Drop leading zeros and replace **one** run of zero hextets with `::`. Option c uses `::` twice, which is ambiguous. | §2.2 |
| 10 | **b** | Four aligned /24s (4–7: 4 is a multiple of 4) → /22. A /21 would also cover 0–3 or 8–15. | §2.2 |
| 11 | **b** | 10.20.99.1 matches /8 and /16 but not 10.20.30.0/24, so the longest match is the /16. | §2.3 |
| 12 | **b** | TTL (Hop Limit in IPv6) prevents infinite forwarding loops. | §2.2 |
| 13 | **b** | Forwarding = data plane (uses the table); routing = control plane (builds it). | §2.3 |
| 14 | **b** | Gen 2 put forwarding tables on the line cards, but the shared bus carries one packet at a time. | §2.3 |
| 15 | **b** | In SDN the controller knows the whole topology and installs forwarding entries; routers are just the data plane. | §2.4 |
| 16 | **b** | OSPF: flood link states → converge → Dijkstra → forwarding table. | §2.5 |
| 17 | **b** | Selection order: local preference, then **shortest AS path**, then MED, eBGP over iBGP, IGP cost, router ID. | §2.6 |
| 18 | **b** | A customer shouldn't give two providers free transit through itself; output filters stop those announcements. | §2.6 |
| 19 | **b** | Off-LAN traffic goes to the next-hop router, so the host ARPs for the gateway's MAC. | §2.7 |
| 20 | **b** | Discover → Offer → Request → Ack. | §2.7 |
| 21 | **a** | Increasing TTL plus ICMP Time Exceeded (type 11). Silent routers show as `*`. | §2.8 |
| 22 | **b** | Ports provide process-to-process multiplexing. | §3.2 |
| 23 | **b** | The 4-tuple (a 5-tuple with the protocol). | §3.2 |
| 24 | **b** | The ACK number is the next byte expected. The receiver has 0–99 and is missing 100. | §3.3 |
| 25 | **b** | One out-of-order arrival triggers one such ACK; repeats are a stronger (not certain) sign of loss. | §3.3 |
| 26 | **b** | A acknowledges B's SYN with ack = 9000 + 1. | §3.4 |
| 27 | **b** | Window Size = the receive window (rwnd), the receiver's free buffer space (flow control). | §3.3 |
| 28 | **a** | In flight ≤ min(rwnd, cwnd) = 16 KB. | §3.6 |
| 29 | **b** | Per-ACK growth is exponential until the slow-start threshold, then additive increase. | §3.6 |
| 30 | **b** | Equal additive increase, proportional decrease → convergence toward the fair line. | §3.6 |
| 31 | **b** | Additional records help avoid more lookups (e.g., the A record for ns2.google.com). | §4.2 |
| 32 | **b** | Anycast: one prefix announced from many places; BGP picks the closest and reroutes on failure. | §4.2 |
| 33 | **b** | listen() marks the socket as accepting connections; accept() is the call that blocks. | §4.3 |
| 34 | **b** | Receiver makes right; it works best on homogeneous infrastructure. | §4.4 |
| 35 | **b** | 1.1 = persistent connections; 2.0 = multiplexing; 3.0 = QUIC. | §4.4 |
| 36 | **b** | Alice signs with her private key; anyone verifies with her public key. | §5.2 |
| 37 | **b** | Hybrid approach: asymmetric for key distribution, symmetric for speed. | §5.2, §5.5 |
| 38 | **b** | Prefix found but wrong origin = Invalid. Prefix not in the database = Unknown. | §5.4 |

## Part B — Free Response (sample answers)

**39. Loading https://www.example.com** (1 point for each step, up to 5)
1. **Getting an address:** the laptop associates with the Wi-Fi access point (link layer, 802.11). It then runs **DHCP** (Discover → Offer → Request → Ack) to get an IP address, the gateway router, and a DNS server.
2. **Name to IP:** the browser asks the OS to resolve the name. The laptop sends a **DNS** query over **UDP** to its local resolver. The resolver answers from its cache, or walks root → TLD → authoritative servers (iterative queries) and returns the **A record**.
3. **Getting onto the LAN:** the destination isn't on the local subnet, so the laptop uses **ARP** (a broadcast request) to learn the **gateway's MAC address**. It sends frames to the gateway, using CSMA/CA on Wi-Fi.
4. **Crossing the Internet:** routers forward the packet using **longest prefix match** on forwarding tables built by **OSPF** (inside each AS) and **BGP** (between ASes).
5. **Connection:** the browser's socket calls `connect()`, and the OS runs the TCP **three-way handshake** (SYN, SYN-ACK, ACK) to port 443.
6. **Security:** the **TLS handshake** runs next. ClientHello and ServerHello pick ciphers. The server sends its **certificate**, which the browser verifies with its installed root CAs. The client sends a session key encrypted with the server's public key, and both sides send ChangeCipherSpec and Finished.
7. **Request:** the **HTTP GET** request and the response now travel encrypted and integrity-protected (**HTTPS**). TCP provides reliability and congestion control (AIMD, slow start).

**40. OSPF vs BGP**

| | OSPF (link state) | BGP (path vector) |
|---|---|---|
| Knows | The **entire topology** (all link states) | Only its **neighbors and the routes they sent** |
| Exchanges | **All link states** it knows | Its **best route per prefix** (filtered by policy) |
| Calculates | **Dijkstra shortest paths** (every router computes consistent paths) | The **locally best route**, using policy (local pref, AS path length, MED, …) |

Why BGP is used between ASes:
- **Scale:** flooding the full topology of every network in the world to every router doesn't scale. BGP only passes AS-level paths to prefixes.
- **Privacy and autonomy:** companies don't want to reveal their internal topology. BGP exposes only reachability (AS paths).
- **Policy:** the Internet is a set of businesses (customer, provider and peer relationships). BGP lets each AS filter announcements and prefer routes for business reasons. OSPF has no notion of policy; it just computes shortest paths.
- **Loop avoidance:** the AS path lets a router reject routes that already contain its own AS.

**Scoring:** 43–48 exam-ready · 36–42 review the missed sections · below 36 re-study the modules.
