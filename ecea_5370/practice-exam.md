# ECEA 5370 — Practice Final Exam
### Network Systems Foundations

**Format:** like the real final: multiple choice plus one free-response question.
**Suggested time:** 2 hours (the real exam's limit). **No notes**, same as the real exam.
**Answers:** in `practice-exam-answers.md`.

These are new questions written from the lecture material, covering all five modules. None of them are copied from the graded quizzes. The real exam has 34 questions; this one has 40 so you get extra practice.

---

### Part A — Multiple Choice (1 point each)

**Link Layer**

1. Why is the Ethernet FCS placed at the end of the frame rather than in the header?
   - a) So switches can skip it when forwarding
   - b) Because it's the last thing received, so the NIC already has every bit it needs to check
   - c) Because the trailer is encrypted
   - d) To make room for the QoS field in the header

2. A frame arrives with EtherType 0x0806. What does the payload contain?
   - a) An IPv4 packet
   - b) An IPv6 packet
   - c) An ARP message
   - d) A Wake-on-LAN message

3. Using **odd** parity, what parity bit is appended to the data `1011 0010`?
   - a) 0
   - b) 1
   - c) It depends on the generator polynomial
   - d) Odd parity doesn't use an extra bit

4. Using G = `1001` (r = 3), what CRC bits R are appended to D = `101110`?
   - a) 000
   - b) 011
   - c) 101
   - d) 110

5. Why do two Wi-Fi laptops that are both in range of an access point sometimes fail to sense each other's transmissions?
   - a) Wi-Fi NICs use frequency division, so they're on different channels
   - b) They may be out of range of each other even though both reach the AP (the hidden terminal problem)
   - c) The AP blocks carrier sense to save power
   - d) CSMA is only used in Ethernet

6. After a collision, why does each node wait a **random** time before retransmitting?
   - a) To save battery
   - b) So the colliding nodes don't retransmit at the same moment and collide again
   - c) Because the jam signal has a random length
   - d) Random timing is required by TDMA

7. A switch's MAC table is empty. Host A (port 1) sends a frame to host C (port 3). What does the switch do?
   - a) Drops the frame and sends an ARP request
   - b) Learns that A is on port 1 and floods the frame out all other ports
   - c) Learns that C is on port 3 and forwards only to port 3
   - d) Sends the frame back out port 1

8. Which is **not** a technique the lecture listed for dealing with loops in a switched Ethernet network?
   - a) Spanning Tree Protocol
   - b) Limiting the number of hops a frame can take
   - c) Architecting the topology to avoid loops
   - d) Increasing the FCS to 64 bits

**Network Layer**

9. Which IPv6 address is a valid shortened form of `2001:0db8:0000:0000:0000:0000:1428:57ab`?
   - a) `2001:db8::1428:57ab`
   - b) `2001:db8:1428:57ab`
   - c) `2001::db8::1428:57ab`
   - d) `2001:db8:0:1428:57ab`

10. Which single prefix exactly covers 192.168.4.0/24, 192.168.5.0/24, 192.168.6.0/24 and 192.168.7.0/24?
    - a) 192.168.4.0/21
    - b) 192.168.4.0/22
    - c) 192.168.0.0/22
    - d) 192.168.4.0/23

11. A router's forwarding table has 10.0.0.0/8 → port 1, 10.20.0.0/16 → port 2, and 10.20.30.0/24 → port 3. Where does a packet to 10.20.99.1 go?
    - a) Port 1
    - b) Port 2
    - c) Port 3
    - d) It's dropped

12. What is the main purpose of the TTL field in the IPv4 header?
    - a) To record how long the packet has been queued
    - b) To prevent packets from looping forever: each hop decrements it, and the packet is dropped at 0
    - c) To tell the receiver how long to cache the packet
    - d) To set the TCP retransmission timeout

13. Which statement correctly distinguishes forwarding from routing?
    - a) Forwarding computes paths with neighbors; routing moves packets to output ports
    - b) Forwarding is the data plane using the forwarding table; routing is the control plane that builds it
    - c) They're two names for the same thing
    - d) Routing happens in the switch fabric; forwarding happens in the route processor

14. What was the main limitation of second-generation (bus-based) routers?
    - a) The CPU had to look up every packet
    - b) Only one packet could cross the shared bus at a time
    - c) They required special crossbar hardware
    - d) They couldn't store forwarding tables

15. In a centralized control plane (SDN), what role do routers play?
    - a) They each run OSPF and BGP independently
    - b) They become just the data plane; a controller computes and installs forwarding entries
    - c) They act as DNS resolvers
    - d) They elect one router to run Dijkstra for everyone

16. In OSPF, what does each router do after the link state database converges?
    - a) Asks its neighbors which path they prefer
    - b) Runs Dijkstra's shortest-path algorithm from itself to build its forwarding table
    - c) Floods its forwarding table to every host
    - d) Picks the route with the highest local preference

17. Two BGP routes to the same prefix have the same local preference. Route X has AS path length 3 and route Y has AS path length 2. Which is preferred, and why?
    - a) X, because longer paths have more redundancy
    - b) Y, because shorter AS path length is the next tiebreaker after local preference
    - c) X, because it was learned first
    - d) Whichever has the lower router ID

18. A small company buys Internet service from two providers. Which BGP policy should it apply?
    - a) Announce all routes learned from provider 1 to provider 2
    - b) Don't announce routes learned from one provider to the other, so it doesn't carry transit traffic between them
    - c) Set every route's local preference to 0
    - d) Run OSPF with both providers instead of BGP

19. Host 10.1.1.5 wants to send to 8.8.8.8, which is on a different network. Whose MAC address does it put in the Ethernet destination field?
    - a) 8.8.8.8's MAC
    - b) Its default gateway (next-hop router)'s MAC
    - c) FF:FF:FF:FF:FF:FF
    - d) The DNS server's MAC

20. What is the correct order of DHCP messages?
    - a) Request → Offer → Discover → Ack
    - b) Discover → Offer → Request → Ack
    - c) Discover → Request → Offer → Ack
    - d) Offer → Discover → Ack → Request

21. How does traceroute learn the address of each router along a path?
    - a) It sends packets with TTL = 1, 2, 3, … and records which router returns an ICMP Time Exceeded message
    - b) It queries each router's BGP table
    - c) It reads the IP Record Route option from a single packet
    - d) It uses ARP to discover every router on the Internet

**Transport Layer**

22. Which field lets the operating system deliver an arriving datagram to the right application?
    - a) The IP TTL
    - b) The destination port number
    - c) The Ethernet type
    - d) The TCP window size

23. Which combination uniquely identifies a TCP connection?
    - a) Destination IP and destination port only
    - b) Source IP, destination IP, source port and destination port
    - c) Source MAC and destination MAC
    - d) The sequence number

24. A sender transmits segments with sequence numbers 0, 100, 200 and 300 (100 bytes each). The segment at 100 is lost. What ACK number will the receiver keep sending?
    - a) 0
    - b) 100
    - c) 200
    - d) 400

25. Why is a **duplicate** ACK a stronger loss signal than a single ACK for an old sequence number?
    - a) Duplicate ACKs come from routers
    - b) A single out-of-order arrival can trigger one such ACK; repeated ACKs for the same byte suggest the packet really was dropped
    - c) Duplicate ACKs carry ECN bits
    - d) TCP ignores single ACKs

26. In the TCP three-way handshake, host A sends SYN with seq = 4000 and host B replies with SYN+ACK, seq = 9000. What does A send next?
    - a) ACK, ack = 4001
    - b) ACK, ack = 9001
    - c) SYN, seq = 4001
    - d) FIN, ack = 9000

27. What does TCP's Window Size header field advertise?
    - a) The sender's congestion window
    - b) How many bytes the receiver currently has buffer space for
    - c) The maximum segment size of the path
    - d) The round-trip time estimate

28. A host has rwnd = 64 KB from the receiver and cwnd = 16 KB. What is the most unacknowledged data it may have in flight?
    - a) 16 KB
    - b) 48 KB
    - c) 64 KB
    - d) 80 KB

29. What does TCP slow start do?
    - a) Waits one RTT before sending any data
    - b) Increases cwnd on every ACK, growing it exponentially until a threshold, then switches to linear increase
    - c) Halves cwnd on every ACK
    - d) Sends at the receiver's advertised rate immediately

30. Why does AIMD lead competing flows toward a fair share?
    - a) Routers enforce equal rates
    - b) Every flow increases by the same amount, but a flow sending more loses more on each multiplicative decrease
    - c) Flows take turns in time slots
    - d) The receiver tells each sender its fair share

**Application Layer**

31. In a DNS response, what does the **additional** section typically contain?
    - a) The original question repeated
    - b) Extra records, such as the IP address of a name server listed in the authority section, to save another lookup
    - c) The resolver's cache contents
    - d) The TTL of the root zone

32. How does anycast give DNS root servers redundancy?
    - a) Each copy of a root server has a different IP, and resolvers rotate through them
    - b) The same IP prefix is announced via BGP from several locations, so traffic goes to the nearest one and fails over if a site goes down
    - c) Root servers multicast every response to all resolvers
    - d) Resolvers use ARP to find the closest root server

33. What does `listen()` do on a TCP server socket?
    - a) Blocks until data arrives
    - b) Tells the OS the socket is willing to accept incoming connections
    - c) Sends the first SYN
    - d) Assigns the socket's IP address and port

34. Which conversion strategy has the sender transmit data in its own internal format, along with a description of that format?
    - a) Canonical intermediate form
    - b) Receiver makes right
    - c) Untagged encoding
    - d) Base-type flattening

35. What did HTTP/1.1 add over HTTP/1.0?
    - a) HTTP over QUIC
    - b) Persistent connections that reuse one TCP connection for several requests
    - c) Binary protobuf encoding
    - d) Mandatory encryption

**Network Security**

36. Bob wants to verify that a message really came from Alice and wasn't changed. Which key does he use?
    - a) Alice's private key
    - b) Alice's public key
    - c) Bob's private key
    - d) A Diffie-Hellman group number

37. Why do protocols like TLS use asymmetric encryption only to exchange a session key, then switch to symmetric encryption?
    - a) Symmetric encryption provides authentication but asymmetric doesn't
    - b) Asymmetric encryption is slow and suited to short messages, while symmetric is fast for bulk data
    - c) Asymmetric keys can't be shared
    - d) Symmetric encryption doesn't need a key

38. In RPKI, a router receives a BGP update for a prefix that's in the ROA database, but the origin AS doesn't match. What is its validation state?
    - a) Valid
    - b) Invalid
    - c) Unknown
    - d) Pending

---

### Part B — Free Response

39. *(5 points)* Walk through what happens, layer by layer, when you type `https://www.example.com` into a browser on a laptop that just joined a Wi-Fi network. Cover how the laptop gets an IP address, how the name becomes an IP address, how the first packet gets onto the LAN, how the connection is established, and how the traffic is secured. Name the protocol used at each step.

40. *(5 points)* Compare OSPF and BGP using the course's three questions: what each router knows, what each router exchanges, and what calculation it performs. Then explain why the Internet uses BGP between autonomous systems instead of OSPF.

---
*End of practice exam. Scoring: Part A 38 points + Part B 10 points = 48 points.*
