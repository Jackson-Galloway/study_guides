# ECEA 5371 — Practice Final Exam: Answer Key

## Part A — Multiple Choice

| # | Ans | Explanation | Review |
|---|---|---|---|
| 1 | **c** | Management plane = how users or automation configure and monitor (CLI, Terraform, SNMP). | §1.1 |
| 2 | **a** | Containerlab uses eth0 for its management network. | §1.3 |
| 3 | **b** | `-Y` is a display filter (applied after capture); `-f` is a capture filter (BPF syntax, before capture). | §1.4 |
| 4 | **a** | sendp = layer 2; send = layer 3 (the kernel builds the Ethernet header). | §1.4 |
| 5 | **b** | New devices start down. The bridge (and its ports) must be set up. | §1.5 |
| 6 | **b** | `ip link add link <dev> name <dev>.<id> type vlan id <id>`. | §1.5 |
| 7 | **b** | An address with a prefix length installs a connected route once the device is up. | §2.2 |
| 8 | **a** | Without via, Linux ARPs for the final destination, which isn't on its segment. With via it ARPs for the router. | §2.4 |
| 9 | **b** | GRE adds an outer IP header (local → remote) and a GRE header around the inner packet. | §2.4 |
| 10 | **b** | Static neighbor entries skip ARP (consistent performance; also handy for troubleshooting). | §2.5 |
| 11 | **b** | `protocol kernel` syncs BIRD's tables with the kernel FIB; `protocol device` reads interface info. | §2.6 |
| 12 | **a** | Peering needs a TCP connection between reachable IPs, and the configured neighbor address and AS must match. | §2.6 |
| 13 | **a** | ExaBGP is a programmable test speaker that can inject arbitrary AS paths (e.g., from RIPE dumps). | §2.7 |
| 14 | **b** | Traffic *to* the box goes through the filter table's INPUT chain; FORWARD is for traffic passing through. | §3.3 |
| 15 | **b** | NAPT (dynamic mapping) rewrites the source IP and port outbound, and reverses it on replies. | §3.2 |
| 16 | **b** | REJECT tells the sender (ICMP); DROP is silent. | §3.3 |
| 17 | **b** | TCP state (sequence numbers, buffers) lives on one backend. | §3.4 |
| 18 | **c** | Source hashing: the hash of (src IP, src port) always maps a flow to the same server. | §3.4 |
| 19 | **b** | Lowercase `-a` adds a real server; `-m` = masquerading (NAT), which allows a different port. Uppercase `-A` adds a service. | §3.5 |
| 20 | **b** | DSR offloads return traffic from the load balancer. | §3.5 |
| 21 | **b** | That's why RED, CoDel and FQ-CoDel drop earlier. | §3.6 |
| 22 | **c** | Strict priority always serves the top queue first. | §3.6 |
| 23 | **b** | netem emulates delay (100 ms ± 12 ms) and loss (10%). | §3.7 |
| 24 | **a** | `task_struct → nsproxy → struct net` (netns_ipv4 with the FIB, netns_nf with netfilter). | §4.2 |
| 25 | **b** | vB lives in ns1, so configure it from inside ns1 (or `ip -n ns1 addr add …`). | §4.3 |
| 26 | **b** | Bind-mount `/proc/<pid>/ns/net` onto `/var/run/netns/<name>` to make it visible. | §4.5 |
| 27 | **b** | KinD = Kubernetes in Docker. | §5.2 |
| 28 | **c** | The scheduler filters and scores nodes and assigns the Pod; the kubelet then runs it. | §5.3 |

## Part B — Free Response (sample answers)

**29. Connecting `red` and `blue` through a bridge**
```
ip netns add red                                   # 1. create the namespaces
ip netns add blue
ip link add vR type veth peer name vR-ns           # 2. create the veth pairs
ip link add vB type veth peer name vB-ns
ip link set vR-ns netns red                        # 3. move one end of each into its namespace
ip link set vB-ns netns blue
ip link add name br0 type bridge                   # 4. create a bridge in root and bring it up
ip link set br0 up
ip link set vR master br0                          # 5. attach the root-side ends to the bridge
ip link set vB master br0
ip link set vR up; ip link set vB up               # 6. bring every end up
ip netns exec red  ip link set vR-ns up
ip netns exec blue ip link set vB-ns up
ip netns exec red  ip addr add 10.0.0.1/24 dev vR-ns   # 7. same subnet, so no via/routes needed
ip netns exec blue ip addr add 10.0.0.2/24 dev vB-ns
ip netns exec red  ping 10.0.0.2                   # 8. test: ARP broadcasts through br0, then ICMP
```
**Finding the peer:**
- Run `ip netns exec red ethtool -S vR-ns`. It prints `peer_ifindex: N`.
- In the root namespace, `ip link | grep "^N:"` shows the matching root-side veth.
- Or go the other way: `ethtool -S vR` in the root namespace, then look for that index with `ip netns exec red ip link`.

Grading: 1 point each for namespaces, veth pairs, moving the ends, bridge plus attach plus up, addressing and test. Lose half a point if the peer method is missing.

**30. Deployment → running Pods → Service traffic**
1. `kubectl` POSTs the Deployment to the **API server**. It passes **authentication**, **admission control** and **validation**, then is stored in **etcd**.
2. The **Deployment controller** (in the controller manager) is watching Deployments. It creates a **ReplicaSet**.
3. The **ReplicaSet controller** sees it and creates **2 Pod** objects (no node yet).
4. The **scheduler** is watching for unscheduled Pods. It **filters and scores** the nodes and assigns each Pod a node (by updating the Pod object).
5. The **kubelet** on each chosen node is watching for Pods assigned to it. It tells the **container runtime** (e.g., containerd) to create the Pod.
6. The runtime calls the **CNI plugin** (the config in `/etc/cni/net.d`, the binary in `/opt/cni/bin`) with **ADD**. The plugin allocates an IP from the node's subnet, creates a veth pair, moves one end into the Pod's network namespace, assigns the address and route, and attaches the other end to a bridge (cni0). Each Pod now has an IP.
7. For the Service, the **Endpoints controller** records the two Pod IPs. **kube-proxy** on every node watches Services/Endpoints and **programs iptables (or IPVS)** DNAT rules for the Service's virtual IP. For example, with iptables it uses `-m statistic --probability 0.5` for the first endpoint and a catch-all rule for the second.
8. **Packet path:** a Pod sends to the Service IP. On its node, the iptables nat PREROUTING rules rewrite the destination to one of the Pod IPs, picked at random. The packet then follows normal Pod-to-Pod networking set up by the CNI plugin (veth → bridge → host routes or a tunnel → the other node's bridge → veth). **kube-proxy itself never touches the packet.**

**Scoring:** 34–38 exam-ready · 28–33 review the missed sections · below 28 re-study the modules.
