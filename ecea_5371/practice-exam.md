# ECEA 5371 — Practice Final Exam
### Network Principles in Practice: Linux Networking

**Format:** like the real final: multiple choice plus one free-response question.
**Suggested time:** 2 hours (the real exam's limit). **No notes**, same as the real exam.
**Answers:** in `practice-exam-answers.md`.

These are new questions written from the lecture material, covering all five modules. None of them are copied from the graded quizzes or labs. The real exam has 22 questions; this one has 30 so you get extra practice.

---

### Part A — Multiple Choice (1 point each)

**Module 1 — Linux networking intro**

1. Which plane of a network appliance does SNMP or Terraform belong to?
   - a) Data plane
   - b) Control plane
   - c) Management plane
   - d) Forwarding plane

2. In a containerlab topology file, why do the links connect `eth1` (and up) rather than `eth0`?
   - a) eth0 is reserved for containerlab's management network
   - b) eth0 only supports IPv6
   - c) Docker doesn't allow eth0 inside containers
   - d) eth0 is always the bridge device

3. You want tshark to show only packets whose IP TTL is greater than 10, after capturing everything. Which option do you use?
   - a) `-f "ip.ttl > 10"`
   - b) `-Y "ip.ttl > 10"`
   - c) `-r "ip.ttl > 10"`
   - d) `-i "ip.ttl > 10"`

4. In Scapy, what's the difference between `sendp()` and `send()`?
   - a) `sendp()` sends at layer 2 (you supply the Ethernet header); `send()` sends at layer 3
   - b) `sendp()` sends a pcap file; `send()` sends one packet
   - c) `sendp()` waits for a reply; `send()` doesn't
   - d) There is no difference

5. After `ip link add name br0 type bridge` and `ip link set eth1 master br0`, frames still aren't being forwarded. What's the most likely missing step?
   - a) Assigning an IP address to eth1
   - b) Running `ip link set br0 up`
   - c) Adding a VLAN to br0
   - d) Rebooting the container

6. Which command creates a VLAN interface that tags traffic on eth1 with VLAN ID 5?
   - a) `ip link add name vlan5 type bridge id 5`
   - b) `ip link add link eth1 name eth1.5 type vlan id 5`
   - c) `ip addr add vlan 5 dev eth1`
   - d) `ip link set eth1 vlan 5`

**Module 2 — IP layer**

7. You run `ip addr add 10.5.5.1/24 dev eth2` and bring eth2 up. What happens to the routing table?
   - a) Nothing; you must add every route manually
   - b) A connected route for 10.5.5.0/24 via eth2 appears automatically
   - c) A default route via 10.5.5.1 is added
   - d) All other routes are deleted

8. Host H has `ip route add 10.0.0.0/16 dev eth1` (no `via`). It pings 10.0.3.2, which is behind a router. tshark shows repeated "Who has 10.0.3.2?" ARP requests with no reply. What's the fix?
   - a) Add `via <router's address on H's subnet>` to the route (plus a route to reach the router itself)
   - b) Flush the ARP table
   - c) Add a VLAN
   - d) Increase the TTL

9. With a GRE tunnel created by `ip tunnel add gre1 mode gre local 10.0.1.2 remote 10.0.3.2`, what does a ping to the tunnel peer 192.168.0.2 look like on the physical interface eth1?
   - a) [IP 192.168.0.1→192.168.0.2][ICMP]
   - b) [IP 10.0.1.2→10.0.3.2][GRE][IP 192.168.0.1→192.168.0.2][ICMP]
   - c) [GRE][ICMP] with no IP header
   - d) [IP 10.0.3.2→10.0.1.2][ICMP]

10. Why might you add a static ARP entry with `ip neigh add`?
    - a) To make traffic encrypted
    - b) To avoid ARP lookups and get consistent performance from the first packet
    - c) To change a host's IP address
    - d) To create a VLAN

11. In a BIRD configuration, what does the `protocol kernel` block do?
    - a) Defines a BGP neighbor
    - b) Synchronizes BIRD's routing table with the Linux kernel's forwarding table
    - c) Loads the ip_vs kernel module
    - d) Sets the router's AS number

12. A BGP session in `birdc show protocols` is stuck and never reaches Established. Which is the most likely cause?
    - a) The neighbor IP or AS number in the config doesn't match the peer, or the routers can't reach each other's IPs
    - b) The routing table has too many routes
    - c) traceroute isn't installed
    - d) The kernel protocol block exports all routes

13. What makes ExaBGP different from BIRD for experiments?
    - a) ExaBGP can announce routes with arbitrary, made-up AS paths and be scripted in Python
    - b) ExaBGP forwards packets faster
    - c) ExaBGP only runs OSPF
    - d) ExaBGP replaces containerlab

**Module 3 — Gateway**

14. Which iptables rule blocks SSH (TCP 22) connections addressed to the gateway itself?
    - a) `iptables -A FORWARD -p tcp --dport 22 -j DROP`
    - b) `iptables -A INPUT -p tcp --dport 22 -j DROP`
    - c) `iptables -t nat -A POSTROUTING -p tcp --dport 22 -j DROP`
    - d) `iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT`

15. A home router lets dozens of devices share one public IP. On outbound packets, which fields does it rewrite?
    - a) Destination IP and destination port
    - b) Source IP and source port
    - c) Only the TTL
    - d) Only the MAC addresses

16. What is the difference between `-j DROP` and `-j REJECT --reject-with icmp-port-unreachable`?
    - a) DROP logs the packet; REJECT doesn't
    - b) DROP silently discards; REJECT discards and sends an ICMP error back to the sender
    - c) REJECT only works in the nat table
    - d) There is no difference

17. Why does a load balancer need flow affinity?
    - a) To encrypt traffic
    - b) Because a TCP connection's state lives on one server, so every packet of that connection must reach the same server
    - c) To make round robin stateless
    - d) To reduce the number of servers

18. Which load-balancing algorithm needs no per-flow state on the load balancer?
    - a) Round robin
    - b) Least connections
    - c) Source hashing
    - d) Weighted round robin

19. Which ipvsadm command adds backend 10.1.1.5 on port 8080 to the TCP service 203.0.113.10:80 using NAT?
    - a) `ipvsadm -A -t 203.0.113.10:80 -r 10.1.1.5:8080 -m`
    - b) `ipvsadm -a -t 203.0.113.10:80 -r 10.1.1.5:8080 -m`
    - c) `ipvsadm -a -t 203.0.113.10:80 -r 10.1.1.5:8080 -g`
    - d) `ipvsadm -a -u 10.1.1.5:8080 -s rr`

20. What is Direct Server Return (DSR)?
    - a) The load balancer answers requests itself
    - b) Backend servers reply directly to the client, bypassing the load balancer
    - c) The client connects directly to a backend without a load balancer
    - d) A tc qdisc for servers

21. Why is tail drop a poor fit for TCP traffic?
    - a) It drops packets too early
    - b) Losses only happen once the queue is full, so TCP learns about congestion late, after many more packets are sent and dropped
    - c) It reorders packets
    - d) It requires deep packet inspection

22. Which scheduling approach can starve low-priority traffic?
    - a) Round robin
    - b) Fair queuing
    - c) Strict priority
    - d) Token bucket

23. What does `tc qdisc replace dev eth0 root netem delay 100ms 12ms 10%` do?
    - a) Rate-limits eth0 to 100 Mbit/s
    - b) Emulates a real-world network on eth0 by adding delay (with variation) and probabilistic loss
    - c) Creates a VLAN with ID 100
    - d) Deletes all qdiscs on eth0

**Module 4 — Virtual networking**

24. Which kernel structure explains why each network namespace has its own forwarding table and iptables rules?
    - a) Each process's nsproxy points to a `struct net`, which holds per-namespace IPv4 (FIB) and netfilter state
    - b) Each namespace runs its own copy of the kernel
    - c) The hypervisor gives each namespace a virtual NIC
    - d) cgroups duplicate the routing table

25. You created a namespace `ns1` and a veth pair `vA`/`vB`, and moved `vB` into `ns1`. Which command assigns 10.1.1.2/24 to `vB`?
    - a) `ip addr add 10.1.1.2/24 dev vB`
    - b) `ip netns exec ns1 ip addr add 10.1.1.2/24 dev vB`
    - c) `ip netns add 10.1.1.2/24 dev vB`
    - d) `ip link set vB netns 10.1.1.2`

26. After starting a container with `docker run`, `ip netns list` shows nothing. Why?
    - a) Docker doesn't use network namespaces
    - b) Docker doesn't create an entry in /var/run/netns, which is where `ip netns` looks
    - c) The container hasn't started yet
    - d) ip netns only shows IPv6 namespaces

**Module 5 — Kubernetes**

27. In a kind cluster, what is each Kubernetes node?
    - a) A separate physical server
    - b) A Docker container
    - c) A network namespace with no processes
    - d) A VirtualBox VM

28. Which component assigns a newly created Pod to a node?
    - a) kubelet
    - b) kube-proxy
    - c) The scheduler
    - d) etcd

---

### Part B — Free Response

29. *(5 points)* You're given a Linux VM and asked to connect two network namespaces, `red` and `blue`, so they can ping each other through a bridge in the root namespace. Write the commands, in order, and briefly explain each step. Then explain how you would confirm which root-namespace veth is paired with the one inside `red`.

30. *(5 points)* A user runs `kubectl apply -f deployment.yaml` for a Deployment with 2 replicas, then creates a Service that selects those Pods. Trace what happens: which control-plane and node components act, in order, until both Pods are running with IPs. Then explain how a packet from another Pod to the Service's IP reaches one of the two Pods. Name the role of the CNI plugin and kube-proxy.

---
*End of practice exam. Scoring: Part A 28 points + Part B 10 points = 38 points.*
