# ECEA 5372 — Practice Final Exam: Answer Key

## Part A — Multiple Choice

| # | Ans | Explanation | Review |
|---|---|---|---|
| 1 | **b** | IaaS: the provider runs the infrastructure; you handle the OS, middleware, VMs, apps and data. | §1.2 |
| 2 | **b** | Elasticity (pay for use) avoids over- and under-provisioning. | §1.2 |
| 3 | **b** | Name-location separation: look up the destination's ToR in the directory instead of broadcasting. | §1.4 |
| 4 | **c** | VL2 enforces the hose model for isolation using existing mechanisms, namely TCP. | §1.4 |
| 5 | **b** | Volatile, unpredictable traffic led VL2 to Valiant LB (randomization) instead of frequent optimization. | §1.4 |
| 6 | **b** | Terraform is declarative; its reconciliation brings the current state back to the desired state. | §1.6 |
| 7 | **b** | plan previews; apply makes the changes; init downloads providers; destroy tears down. | §1.6 |
| 8 | **b** | VPCs are global in GCP; subnets are regional. | §2.1 |
| 9 | **c** | The external IP is mapped by Andromeda; the guest only sees its internal IP. | §1.5 |
| 10 | **b** | 0.0.0.0/0 = any address. The lecture hit exactly this bug. | §2.2 |
| 11 | **b** | Networking mastered complexity instead of extracting abstractions. | §2.3 |
| 12 | **b** | A fixed parser, fields and actions → reconfigurable match tables / P4. | §2.3 |
| 13 | **a** | Configure = parser, tables, control flow; populate = entries. | §2.3 |
| 14 | **b** | Most VM pairs never talk or talk very little; Hoverboards made programming 37× faster. | §2.4 |
| 15 | **b** | Agility (SDN changes) plus hardware performance; embedded CPUs didn't scale. | §2.5 |
| 16 | **b** | SR-IOV alone has no SDN policy, which is why the FPGA sits in front. | §2.5 |
| 17 | **c** | 172.16.0.0–172.31.255.255 is private; 172.33.x.x is public; 192.169 isn't 192.168. | §3.1 |
| 18 | **b** | Non-transitive. | §3.2 |
| 19 | **b** | Peering must be created in both directions. | §3.2 |
| 20 | **b** | Cloud NAT is distributed into Andromeda; it needs a Cloud Router for configuration. | §3.3 |
| 21 | **a** | Asynchronous updates; SWAN uses scratch capacity for congestion-free steps. | §3.4 |
| 22 | **b** | Working-set tunnels (compare Andromeda's Hoverboards). | §3.4 |
| 23 | **a** | Hierarchical control with fail-static local control. | §3.5 |
| 24 | **b** | +13% traffic served, better user experience. | §3.5 |
| 25 | **b** | The health check needs the app responding on /health. | §4.3 |
| 26 | **b** | Premium = cold potato (exit near the destination); standard = exit near the source. | §4.2 |
| 27 | **b** | Link-local metadata server. | §4.4 |
| 28 | **b** | That's why consistent hashing exists. | §4.5 |
| 29 | **b** | ECMP changes plus local state → Maglev adds consistent hashing so every Maglev picks the same backend. | §4.5 |
| 30 | **b** | Cache miss → fetch from origin → cache; later hits are served from the edge. | §4.6 |
| 31 | **b** | RON recovers from outages in <20 s vs minutes for BGP. | §4.7 |
| 32 | **b** | The Cloud Router advertises subnets and learns on-prem routes via BGP. | §5.2 |
| 33 | **a** | HA VPN = two tunnels; Classic = one. | §5.2 |
| 34 | **b** | Cage = yours; meet-me room = the providers'; a cross connect joins them. | §5.3 |
| 35 | **a** | Partners (Megaport, Console Connect, Equinix) extend reach and automate connections; you still need a Cloud Router, VLAN attachment and BGP. | §5.3 |

## Part B — Free Response (sample answer, 2 points per need)

**(1) Backend VMs need outbound Internet: Cloud NAT + Cloud Router.**
- **How it works:**
  - Cloud NAT gives VMs **without external IPs** outbound access by mapping their private IP and port to a shared public IP and port pool (many-to-one NAT, like a home router).
  - In GCP there's **no NAT box in the data path**. **Andromeda**, the virtual switch on each host, performs the translation using ports assigned to each VM.
  - Cloud NAT is configured through a Cloud Router.
- **Trade-off:** it's outbound only. Nobody on the Internet can initiate connections to the backends, which is good for security but means you'd need a load balancer if inbound access were ever required. There are also port allocation limits per VM.

**(2) Office ↔ VPC privately: HA VPN (IPsec) + Cloud Router (BGP).**
- **How it works:**
  - Create an **HA VPN gateway** (two tunnels), and a **peer gateway** with the office firewall's public IPs.
  - **IKE** negotiates parameters, does a **Diffie-Hellman** key exchange, and authenticates; then traffic flows encrypted.
  - A **Cloud Router** (private ASN) runs **BGP** with the office router, advertising the VPC's subnets and learning the office's routes.
  - The office firewall/router is configured for IPsec + BGP (Google offers config templates).
- **Trade-off:** it runs over the public Internet, so bandwidth is lower, latency higher, and there's no SLA. If they later have colocation space, **Dedicated or Partner Interconnect** (a port on Google's switch in a colo, plus a VLAN attachment, Cloud Router and BGP) gives higher bandwidth and reliability at higher cost.

**(3) Frontend spikes + single global IP: a global external Application Load Balancer + a managed instance group with autoscaling (Premium tier).**
- **How it works:**
  - Build an image → **instance template** → **managed instance group** with an **autoscaler**: threshold-based (e.g., CPU above 50% adds a VM) or predictive (ML, as in Google's **Autopilot**). It adds and removes VMs and updates the LB's backend set, with **health checks** removing unhealthy ones.
  - The **application LB** is an L7 proxy (**Envoy**, running in a proxy-only subnet) that terminates TCP and can route by URL.
  - With **Premium tier**, the LB's single IP is **announced via BGP anycast from many PoPs**, so each user reaches the nearest Google Front End, and egress rides Google's backbone ("cold potato").
  - Alternatively, an L4 network load balancer would use **Maglev**: ECMP spreads packets across Maglev servers, and consistent hashing plus connection tracking keep each TCP connection on the same backend even as servers change.
  - Adding **Cloud CDN** would cache static content at the edge.
- **Trade-off:** autoscaling reacts with a delay (VM boot time), so set an initialization period and maximum instance count to cap cost. Premium tier costs more than Standard. L7 proxies add processing compared with L4 pass-through.

**Scoring:** 37–41 exam-ready · 31–36 review the missed sections · below 31 re-study the modules.
