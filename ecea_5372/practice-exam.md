# ECEA 5372 — Practice Final Exam
### Network Principles in Practice: Cloud Networking

**Format:** like the real final: multiple choice plus one free-response question.
**Suggested time:** 90 minutes (the real exam's limit). **No notes**, same as the real exam.
**Answers:** in `practice-exam-answers.md`.

These are new questions written from the lectures and assigned papers, covering all five modules. None of them are copied from the graded quizzes or labs. The real exam has 34 questions; this one has 36.

---

### Part A — Multiple Choice (1 point each)

**Module 1 — Cloud background**

1. In the IaaS model, which of these is the **customer** responsible for?
   - a) The physical servers and data center power
   - b) The operating system, middleware and applications
   - c) Nothing; the provider manages everything
   - d) Only the application's user interface

2. Why does fixed-capacity provisioning waste money, according to the Berkeley paper?
   - a) Hardware is always more expensive than cloud VMs
   - b) Demand has peaks and valleys, so fixed capacity is either over-provisioned (idle in valleys) or under-provisioned (fails at peaks)
   - c) Fixed capacity can't run Linux
   - d) It requires a service account

3. In VL2, what does the directory service replace?
   - a) TCP congestion control
   - b) Broadcast-based address resolution (ARP) across a huge L2 domain
   - c) The hypervisor
   - d) Equal-cost multipath routing

4. Which VL2 goal does TCP provide?
   - a) L2 semantics
   - b) Uniform high capacity
   - c) Performance isolation between services
   - d) Name-location separation

5. What was a key finding about traffic in VL2's measurements?
   - a) Traffic was highly predictable by time of day
   - b) Traffic patterns were volatile and unpredictable, so heavy traffic engineering would have to re-run constantly
   - c) Most traffic went to the Internet
   - d) Server-to-server traffic was negligible

6. A colleague deletes a VM in the console that Terraform created. What happens on the next `terraform apply`?
   - a) Terraform errors out permanently
   - b) Terraform sees the current state differs from the desired state and recreates the VM
   - c) Terraform deletes every other resource
   - d) Nothing; Terraform only runs once

7. What does `terraform plan` do?
   - a) Creates the resources immediately
   - b) Shows the changes Terraform would make, without making them
   - c) Deletes all resources
   - d) Downloads providers

**Module 2 — VPCs, SDN, Andromeda, AccelNet**

8. In GCP, which scope does a subnet have?
   - a) Global
   - b) Regional
   - c) Zonal
   - d) Per-VM

9. A VM has internal IP 10.0.1.5 and an external IP. What does `ip addr` inside the VM show?
   - a) Both addresses
   - b) Only the external address
   - c) Only the internal address; the external one is translated by Google's network
   - d) Neither address

10. Your firewall rule allows tcp:22 and icmp with source range `0.0.0.0/24`. SSH and ping from the Internet both fail. Why?
    - a) GCP doesn't support SSH
    - b) The source range only covers 0.0.0.0–0.0.0.255; it should be `0.0.0.0/0` to mean "anywhere"
    - c) ICMP must be allowed on port 22
    - d) Firewall rules take 24 hours to apply

11. In the OSPF example used to motivate SDN, what does the comparison of 245 pages (the full RFC) vs 4 pages (Dijkstra) illustrate?
    - a) Dijkstra's algorithm is inefficient
    - b) Most of the protocol is the distributed system for building a consistent network map; the actual path computation is tiny, which shows a lack of abstraction
    - c) OSPF is only used in data centers
    - d) RFCs are too short

12. Which was the main limitation of OpenFlow that RMT and P4 addressed?
    - a) It had no centralized controller
    - b) Switches could only match on a fixed set of header fields and perform a fixed set of actions, dictated by the ASIC
    - c) It was too expensive to license
    - d) It couldn't forward packets

13. In P4's abstract forwarding model, what is the difference between "configure" and "populate"?
    - a) Configure programs the parser, tables and control flow; populate adds and removes table entries
    - b) Configure installs the OS; populate boots it
    - c) They are the same operation
    - d) Populate defines new headers; configure installs routes

14. What problem do Andromeda's Hoverboards solve?
    - a) Encrypting VM traffic
    - b) The time and memory needed to program a rule for every VM pair on every host, by sending the long tail of low-traffic flows through gateways and only offloading busy flows to hosts
    - c) Live migration of VMs
    - d) Load balancing HTTP requests

15. Why did Azure choose FPGAs over ASICs for AccelNet?
    - a) FPGAs are cheaper than CPUs
    - b) ASICs take 12–18 months to design and can't be reprogrammed as SDN features change; FPGAs give hardware performance with programmability
    - c) ASICs can't process packets
    - d) FPGAs don't need a host CPU at all

16. What does SR-IOV provide on its own?
    - a) Full SDN policy enforcement in the NIC
    - b) Native-performance delivery of packets directly to a VM, but without SDN policy processing
    - c) Encryption between VMs
    - d) Consistent hashing

**Module 3 — WAN**

17. Which address is in an RFC 1918 private range?
    - a) 172.33.10.1
    - b) 192.169.1.1
    - c) 172.20.5.5
    - d) 11.0.0.1

18. VPC A is peered with VPC B, and VPC B is peered with VPC C. Can a VM in A reach a VM in C over private IPs through B?
    - a) Yes, peering is transitive
    - b) No, GCP peering isn't transitive; A needs its own peering with C
    - c) Yes, but only for ICMP
    - d) Only if B has Cloud NAT

19. After you create a peering from vpc1 to vpc2, its status shows inactive. What's missing?
    - a) A Cloud NAT gateway
    - b) The matching peering from vpc2 to vpc1
    - c) An external IP on each VM
    - d) A VPN tunnel

20. How does Cloud NAT translate addresses for VMs without external IPs?
    - a) Through a dedicated NAT VM you must create
    - b) Andromeda on each host performs the translation with ports assigned to that VM; there's no NAT box in the data path
    - c) It assigns each VM a temporary external IP
    - d) It tunnels traffic to the user's office

21. In SWAN, what makes naive network updates cause congestion?
    - a) Switches apply new configurations at different times, so for a moment old and new paths can overload a link
    - b) The controller sends too many packets
    - c) BGP withdrawals
    - d) TCP slow start

22. How did SWAN cope with limited switch forwarding-table memory?
    - a) It used larger switches
    - b) It used tunnel-based forwarding and installed only the working set of tunnels
    - c) It disabled routing
    - d) It moved everything to FPGAs

23. Which statement best describes Espresso's "fail static" principle?
    - a) If the global controller fails, the local control plane keeps forwarding with its last good state
    - b) Failed servers are never replaced
    - c) Traffic is dropped on any failure
    - d) BGP sessions are reset to recover

24. What does Espresso's application-aware routing allow that plain BGP doesn't?
    - a) Using IPv6
    - b) Sending traffic out a different PoP or ISP than BGP's best path, based on application performance signals
    - c) Running without any peers
    - d) Encrypting all traffic

**Module 4 — Network services**

25. In the GCP application load balancer demo, why did the backends show as unhealthy at first?
    - a) The proxy-only subnet was missing
    - b) The Flask app wasn't running yet, so the HTTP health check on /health at port 5000 failed
    - c) The VMs had no external IPs
    - d) The load balancer was global

26. Which GCP option keeps your traffic on Google's backbone until it's close to the user?
    - a) Standard tier
    - b) Premium tier
    - c) Cloud NAT
    - d) VPC peering

27. What is the address 169.254.169.254 used for in GCP?
    - a) The default gateway of every subnet
    - b) The metadata server (instance metadata like startup scripts, plus DNS and NTP), on a link-local address that's never routed
    - c) Google's public DNS
    - d) The Cloud NAT gateway

28. Why can't a load balancer use `server = hash(5-tuple) mod N` alone?
    - a) Hashing is too slow
    - b) When N changes (a server fails or is added), most flows remap to different servers and existing connections break
    - c) It can't handle UDP
    - d) It requires a proxy-only subnet

29. In Maglev, why isn't connection tracking alone enough to keep connections on the same backend?
    - a) Connection tables are too small
    - b) If the set of Maglevs changes, ECMP can send a flow to a different Maglev that has no state for it
    - c) Connection tracking only works for HTTP
    - d) Backends never change

30. What happens on the first request for an object through a CDN edge server?
    - a) The request is rejected
    - b) The edge server fetches the object from the origin server and caches it for later requests
    - c) The client is redirected to the origin permanently
    - d) The object is deleted from the origin

31. How does Resilient Overlay Networking (used by Akamai for path optimization) help?
    - a) It replaces BGP everywhere
    - b) Overlay nodes monitor paths and route around Internet outages through other overlay nodes, recovering in seconds rather than minutes
    - c) It compresses packets
    - d) It caches content at the edge

**Module 5 — Hybrid**

32. Which component dynamically exchanges routes between a GCP VPC and an on-prem network over HA VPN?
    - a) Cloud NAT
    - b) Cloud Router running BGP
    - c) A firewall rule
    - d) Cloud CDN

33. What does "HA" mean when creating a GCP VPN gateway?
    - a) Two tunnels for high availability
    - b) Hardware acceleration
    - c) HTTP access
    - d) Hybrid anycast

34. In a colocation facility, what is the meet-me room?
    - a) A conference room for customers
    - b) Where network providers' equipment is located, so customers can cross-connect to them
    - c) The cloud provider's data center
    - d) A secured cage for one customer

35. Why might a company choose a partner interconnect (e.g., through a fabric provider) over a dedicated interconnect?
    - a) The partner's global fabric can reach Google even from facilities where Google has no switch, and automates the connection
    - b) Partner interconnects are always encrypted
    - c) Dedicated interconnects require a VPN
    - d) Partner interconnects don't need BGP or a Cloud Router

---

### Part B — Free Response

36. *(6 points)* A startup runs everything in one GCP VPC: web frontends with external IPs, and backend database VMs with **no** external IPs. They want to (1) let the backend VMs download OS updates from the Internet, (2) connect their new office securely so employees can reach the backends privately, and (3) handle unpredictable traffic spikes on the frontend while presenting a single IP worldwide. For each need, name the GCP service(s), explain how each works under the hood (refer to the course's papers and systems where relevant), and note one trade-off.

---
*End of practice exam. Scoring: Part A 35 points + Part B 6 points = 41 points.*
