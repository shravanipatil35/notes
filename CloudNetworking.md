# Cloud Networking — Complete Notes (Basics → Advanced + Interview Prep)

*Concepts are provider-agnostic where possible, with AWS terminology used as the primary reference (most common in interviews) and Azure/GCP equivalents noted throughout.*

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Cloud Networking vs Traditional Networking](#2-cloud-networking-vs-traditional-networking)
3. [Core Networking Fundamentals](#3-core-networking-fundamentals)
4. [Virtual Private Cloud (VPC)](#4-virtual-private-cloud-vpc)
5. [Subnets](#5-subnets)
6. [Route Tables & Routing](#6-route-tables--routing)
7. [Internet Gateway & NAT Gateway](#7-internet-gateway--nat-gateway)
8. [Security Groups vs Network ACLs](#8-security-groups-vs-network-acls)
9. [VPC Peering](#9-vpc-peering)
10. [Transit Gateway (Hub-and-Spoke)](#10-transit-gateway)
11. [VPN Connectivity](#11-vpn-connectivity)
12. [Direct Connect / ExpressRoute / Cloud Interconnect](#12-dedicated-private-connectivity)
13. [DNS in the Cloud](#13-dns-in-the-cloud)
14. [Load Balancing](#14-load-balancing)
15. [Content Delivery Network (CDN)](#15-content-delivery-network-cdn)
16. [Private Endpoints / PrivateLink](#16-private-endpoints--privatelink)
17. [NAT Deep Dive](#17-nat-deep-dive)
18. [Firewalls & WAF](#18-firewalls--waf)
19. [Multi-Region & Multi-AZ Networking](#19-multi-region--multi-az-networking)
20. [Hybrid Cloud Networking](#20-hybrid-cloud-networking)
21. [Network Monitoring & Flow Logs](#21-network-monitoring--flow-logs)
22. [IPv6 in the Cloud](#22-ipv6-in-the-cloud)
23. [Cost Considerations (Data Transfer)](#23-cost-considerations-data-transfer)
24. [Security Best Practices](#24-security-best-practices)
25. [Troubleshooting](#25-troubleshooting)
26. [Best Practices Summary](#26-best-practices-summary)
27. [Cheat Sheet / Terminology Map](#27-cheat-sheet--terminology-map)
28. [Interview Questions & Answers](#28-interview-questions--answers)

---

## 1. Introduction

**Cloud networking** is the set of services and concepts that let you build, connect, and secure virtual network infrastructure inside a public cloud (AWS, Azure, GCP) — and connect it back to on-premises networks, other clouds, and the public internet. Unlike physical networking, almost everything is software-defined: virtual routers, virtual firewalls, and virtual gateways that you configure declaratively (often via Infrastructure as Code).

**Why it matters:**
- Almost every cloud resource (VMs, databases, Kubernetes clusters, serverless functions with VPC access) lives inside a network boundary you control
- Misconfigured networking is one of the most common sources of security incidents (overly open security groups, exposed databases, public S3 buckets reachable due to missing network controls)
- Performance, cost, and availability all depend heavily on how you architect networking (cross-AZ traffic costs, NAT Gateway costs, latency between regions)

---

## 2. Cloud Networking vs Traditional Networking

| Aspect | Traditional (On-Prem) | Cloud |
|---|---|---|
| Hardware | Physical routers, switches, firewalls | Virtualized/software-defined equivalents |
| Provisioning | Manual, slow (rack, cable, configure) | API-driven, minutes, fully automatable (IaC) |
| Scaling | Requires buying/installing new hardware | Elastic, on-demand |
| Cost model | CapEx (upfront hardware purchase) | OpEx (pay-as-you-go, often per-GB or per-hour) |
| Topology changes | Physical rewiring/reconfiguration | Configuration change via console/API/Terraform |
| Redundancy | You build and maintain it | Built into cloud regions/AZs by design |

**Key concept carried over from traditional networking:** IP addressing, subnetting, routing, and firewall concepts are exactly the same — cloud networking is the same fundamentals, expressed as software-defined, API-driven services instead of physical boxes.

---

## 3. Core Networking Fundamentals

These underpin everything in cloud networking — essential to know cold for interviews.

**IP Addressing & CIDR notation:**
```
10.0.0.0/16   → 65,536 IP addresses (10.0.0.0 – 10.0.255.255)
10.0.1.0/24   → 256 IP addresses    (10.0.1.0 – 10.0.1.255)
10.0.1.0/28   → 16 IP addresses     (10.0.1.0 – 10.0.1.15)
```
The number after `/` is the number of bits used for the **network portion** — fewer bits "free" for hosts means a *smaller* range. `/16` = large network; `/24` = common subnet size; `/32` = single host.

**Private IP ranges (RFC 1918)** — used internally in cloud VPCs, never routable on the public internet:
```
10.0.0.0/8        (10.0.0.0 – 10.255.255.255)
172.16.0.0/12     (172.16.0.0 – 172.31.255.255)
192.168.0.0/16    (192.168.0.0 – 192.168.255.255)
```

**OSI Model (quick reference, commonly tested):**
| Layer | Name | Example |
|---|---|---|
| 7 | Application | HTTP, DNS, SMTP |
| 6 | Presentation | TLS/SSL encryption |
| 5 | Session | Session establishment |
| 4 | Transport | TCP, UDP (ports) |
| 3 | Network | IP, routing |
| 2 | Data Link | MAC addresses, switches |
| 1 | Physical | Cables, signals |

**L4 vs L7 (very common interview distinction):** Layer 4 (Transport) operates on IP address + port, doesn't inspect application content (faster, protocol-agnostic — e.g., Network Load Balancer). Layer 7 (Application) understands the actual protocol content (e.g., HTTP headers, URL paths, cookies) and can route/filter based on it (e.g., Application Load Balancer, WAF).

---

## 4. Virtual Private Cloud (VPC)

A **VPC** (AWS/GCP term; Azure calls it a **Virtual Network/VNet**) is an isolated, logically-separated virtual network within a cloud provider's infrastructure — your own private slice of the cloud where you control IP addressing, subnets, routing, and security.

```
VPC: 10.0.0.0/16  (us-east-1)
├── Public Subnet A   10.0.1.0/24   (us-east-1a)
├── Public Subnet B   10.0.2.0/24   (us-east-1b)
├── Private Subnet A  10.0.10.0/24  (us-east-1a)
└── Private Subnet B  10.0.11.0/24  (us-east-1b)
```

**Key properties:**
- A VPC is **region-scoped** (it doesn't span multiple regions, though it can span multiple Availability Zones within that region)
- You define the CIDR block (IP address range) for the VPC, then carve it into smaller subnets
- Comes with a default route table, default security group, and default network ACL — all customizable
- Resources (VMs, databases, load balancers) are launched **inside** subnets, never directly in the VPC

**Provider terminology mapping:**
| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Virtual network | VPC | Virtual Network (VNet) | VPC |
| Compute instance | EC2 | Virtual Machine | Compute Engine |
| Network security | Security Groups + NACLs | Network Security Groups (NSGs) | Firewall Rules |
| Load balancer | ELB (ALB/NLB/CLB) | Azure Load Balancer / App Gateway | Cloud Load Balancing |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Private connectivity | Direct Connect | ExpressRoute | Cloud Interconnect |

---

## 5. Subnets

A **subnet** is a range of IP addresses within a VPC, tied to a single Availability Zone, used to logically (and for security purposes, physically) separate resources.

**Public vs Private subnet — defined by routing, not naming:**
- **Public subnet** — has a route to an **Internet Gateway**, so resources can have public IPs and be directly reachable from/to the internet
- **Private subnet** — has **no** direct route to the internet; outbound internet access (if needed) goes through a NAT Gateway; inbound from the internet is not possible at all without something like a load balancer in front

**Typical 3-tier subnet design:**
```
Public subnet    → Load balancers, bastion hosts, NAT gateways
Private subnet   → Application servers (no direct internet exposure)
Database subnet  → Databases (most restricted, often no internet route of any kind)
```

This tiered design is the backbone of almost every secure cloud architecture — minimizing the "blast radius" of anything directly exposed to the internet.

---

## 6. Route Tables & Routing

A **route table** is a set of rules ("routes") determining where network traffic from a subnet is directed.

```
Destination          Target
10.0.0.0/16           local                  (traffic within the VPC)
0.0.0.0/0              igw-xxxxx               (all other traffic → Internet Gateway = public subnet)
0.0.0.0/0              nat-xxxxx               (all other traffic → NAT Gateway = private subnet)
172.16.0.0/16          pcx-xxxxx               (peered VPC traffic → VPC Peering connection)
```

**Key facts:**
- Every subnet is associated with exactly one route table (though one route table can be shared by multiple subnets)
- The **most specific matching route wins** (longest prefix match) — same principle as traditional networking
- `local` route (within the VPC's own CIDR) always exists implicitly and can't be removed
- A subnet's classification as "public" or "private" is determined entirely by whether its route table sends `0.0.0.0/0` traffic to an Internet Gateway or not

---

## 7. Internet Gateway & NAT Gateway

**Internet Gateway (IGW):**
- A horizontally-scaled, redundant VPC component that allows **two-way** communication between resources in a public subnet and the internet
- Performs 1:1 NAT for instances with a public IP (translates between the instance's private IP and its assigned public IP)
- Attached at the VPC level, referenced in a public subnet's route table

**NAT Gateway:**
- Allows resources in a **private** subnet to initiate **outbound** connections to the internet (e.g., to download a package update or call an external API) **without** being directly reachable from the internet (i.e., one-way: outbound-initiated only)
- Lives in a **public** subnet, has its own Elastic IP, and private subnets route their `0.0.0.0/0` traffic to it
- Managed/fully redundant within its AZ (in AWS); you typically deploy one per AZ for high availability (a NAT Gateway is itself AZ-scoped — cross-AZ traffic through a single NAT Gateway adds cost and a single point of failure if that AZ goes down)

**NAT Gateway vs NAT Instance (older approach):** A NAT Gateway is a fully managed service (no patching, auto-scales bandwidth, AWS-managed HA within the AZ). A NAT Instance is just a regular EC2 instance configured to forward traffic — cheaper at very low scale, but you manage patching/scaling/HA yourself. NAT Gateway is the modern default recommendation.

**Internet Gateway vs NAT Gateway (very common interview question):** An Internet Gateway enables full bidirectional internet connectivity for resources with a public IP (used by public subnets). A NAT Gateway enables only outbound-initiated internet access for resources **without** a public IP (used by private subnets) — nothing on the internet can initiate a connection to a resource behind a NAT Gateway.

---

## 8. Security Groups vs Network ACLs

Both are virtual firewalls, but operate very differently — a classic, very common interview topic.

| Aspect | Security Group | Network ACL (NACL) |
|---|---|---|
| Level | Instance/resource level (attached to ENI/VM) | Subnet level |
| State | **Stateful** — return traffic automatically allowed | **Stateless** — must explicitly allow both inbound AND outbound |
| Rules | Allow rules only (no explicit deny) | Both allow AND deny rules |
| Rule evaluation | All rules evaluated, most permissive applies | Rules evaluated **in order** by rule number, first match wins |
| Default behavior | Denies all inbound by default, allows all outbound by default | Default NACL allows all traffic; custom NACLs deny all by default until rules added |
| Scope | Applies to specific instances/resources you attach it to | Applies to every resource in the associated subnet |

```
# Security Group example (stateful — only need inbound rule, return traffic auto-allowed)
Inbound:  Allow TCP 443 from 0.0.0.0/0
Inbound:  Allow TCP 22 from 203.0.113.0/24 (office IP range)
Outbound: Allow all (default)

# NACL example (stateless — need explicit rules both directions)
Inbound:  100  Allow TCP 443 from 0.0.0.0/0
Inbound:  200  Deny  TCP all from 198.51.100.0/24 (blocked IP range)
Outbound: 100  Allow TCP 1024-65535 from 0.0.0.0/0 (ephemeral ports for return traffic!)
```

**Why NACLs need ephemeral port rules (interview gotcha):** Because NACLs are stateless, if you allow inbound traffic on port 443, you must *also* explicitly allow outbound traffic on the **ephemeral port range** (typically 1024-65535) for the response packets to leave — Security Groups handle this automatically since they're stateful.

**When to use which:** Security Groups are the primary, day-to-day tool for controlling access to specific resources (e.g., "only allow port 5432 from the app servers' security group"). NACLs are typically used as a coarser, subnet-wide safety net — e.g., explicitly blocking a known-malicious IP range across an entire subnet regardless of which security groups are attached to individual instances.

---

## 9. VPC Peering

A **VPC Peering connection** is a direct, private networking connection between two VPCs, allowing resources in each to communicate as if they were on the same network — traffic stays on the cloud provider's private backbone, never traversing the public internet.

```
VPC A (10.0.0.0/16) ◄──── Peering Connection ────► VPC B (10.1.0.0/16)
```

**Key facts:**
- **Not transitive** — if VPC A is peered with VPC B, and VPC B is peered with VPC C, A **cannot** reach C through B. Each pair needs its own direct peering connection (or you need a Transit Gateway instead — see next section).
- CIDR blocks of peered VPCs **must not overlap**
- Each side must update its route table to point traffic destined for the other VPC's CIDR at the peering connection
- Can be cross-account and cross-region (with some provider-specific limitations/costs)
- Security Groups can sometimes be referenced across a peering connection (same-region, same-account scenarios), enabling fine-grained access control across the peered VPCs

**Scaling problem:** With N VPCs all needing to talk to each other via peering, you'd need N(N-1)/2 individual peering connections — quickly becomes unmanageable. This is exactly the problem Transit Gateway solves.

---

## 10. Transit Gateway

A **Transit Gateway** (AWS term; Azure: **Virtual WAN**; GCP: **Network Connectivity Center**) acts as a central hub that VPCs, VPNs, and Direct Connect connections attach to — solving VPC Peering's non-transitive, N² scaling problem.

```
                    ┌─────────────────────┐
                    │   Transit Gateway        │
                    └─┬───────┬───────┬──────┘
                      │        │        │
                 ┌────▼──┐ ┌──▼───┐ ┌──▼────┐
                 │ VPC A   │ │ VPC B  │ │ VPC C    │
                 └───────┘ └──────┘ └────────┘
                      │
                 ┌────▼──────────┐
                 │ On-prem (via VPN/   │
                 │ Direct Connect)        │
                 └────────────────────┘
```

**Benefits over VPC Peering:**
- **Transitive routing** — any attached VPC can reach any other attached VPC (and on-prem networks) through the single hub, without needing pairwise connections
- Centralized route table management — you control exactly which attachments can talk to which others (route table segmentation/isolation between business units, for example)
- Scales to thousands of VPC attachments
- Single place to attach VPN and Direct Connect connections for all VPCs to share

**Trade-off:** Adds a small amount of latency/cost per hop compared to direct peering, and is itself a critical piece of infrastructure to design with redundancy in mind.

---

## 11. VPN Connectivity

**Site-to-Site VPN:** An encrypted tunnel (typically IPsec) connecting your on-premises network/router to your VPC, over the public internet — quick to set up, no special hardware install required at the cloud end, but subject to public internet latency/reliability variability.

```
On-prem Router ◄──── IPsec Tunnel (encrypted, over public internet) ────► VPC (Virtual Private Gateway)
```

**Client VPN:** Allows individual users (e.g., remote employees) to connect securely into a VPC from their laptop, as if on the internal network — commonly used for accessing private resources (databases, internal tools) without exposing them publicly.

**High availability pattern:** Cloud providers typically provision Site-to-Site VPN connections as **two tunnels** (active/active or active/passive) to two different endpoints, so a single tunnel failure doesn't cause an outage — best practice is to configure your on-prem router to use both.

---

## 12. Dedicated Private Connectivity

**Direct Connect** (AWS) / **ExpressRoute** (Azure) / **Cloud Interconnect** (GCP) — a dedicated, private physical network connection between your on-premises data center and the cloud provider, bypassing the public internet entirely.

**VPN vs Direct Connect (common interview comparison):**
| Aspect | Site-to-Site VPN | Direct Connect / ExpressRoute |
|---|---|---|
| Path | Over the public internet (encrypted) | Dedicated private physical circuit |
| Setup time | Minutes to hours | Weeks to months (physical provisioning) |
| Bandwidth | Limited/variable (internet-dependent) | Consistent, high (1Gbps–100Gbps+) |
| Latency | Variable | Consistent, lower |
| Cost | Low (pay for gateway + data transfer) | Higher (dedicated circuit + port fees) |
| Best for | Quick setup, lower bandwidth needs, backup connectivity | Large, consistent data transfer; latency-sensitive workloads; compliance requirements |

A common production pattern is using Direct Connect as the primary path with a Site-to-Site VPN as an automatic failover backup.

---

## 13. DNS in the Cloud

Cloud-managed DNS services (**Route 53** on AWS, **Azure DNS**, **Cloud DNS** on GCP) provide authoritative DNS hosting plus advanced routing capabilities beyond simple A/CNAME records.

**Routing policies (interview-relevant, especially for AWS Route 53):**
| Policy | Behavior |
|---|---|
| **Simple** | Single record, no special logic |
| **Weighted** | Distribute traffic across multiple resources by assigned weight (e.g., canary releases, A/B testing) |
| **Latency-based** | Route users to the region with lowest latency from their location |
| **Geolocation** | Route based on the user's geographic location (e.g., compliance, localized content) |
| **Geoproximity** | Like geolocation but with a "bias" to shift traffic between regions |
| **Failover** | Active-passive — route to a secondary resource if health checks on primary fail |
| **Multivalue answer** | Return multiple IP addresses with health checks (lightweight load balancing/redundancy at DNS level) |

```
# Private DNS — resolvable only within the VPC, never exposed publicly
internal-db.mycompany.internal → 10.0.5.12

# Public DNS — resolvable from anywhere on the internet
www.mycompany.com → CDN/Load Balancer
```

**Private Hosted Zones:** DNS zones that only resolve within specified VPCs — used for internal service discovery without leaking internal hostnames/IPs to the public internet.

---

## 14. Load Balancing

Distributes incoming traffic across multiple backend targets for availability, scalability, and fault tolerance.

| Type | Layer | Use Case | AWS Example |
|---|---|---|---|
| **Layer 4 (Network) Load Balancer** | Transport (TCP/UDP) | Extreme performance, static IP needed, non-HTTP protocols | NLB |
| **Layer 7 (Application) Load Balancer** | Application (HTTP/HTTPS) | Content-based routing (path/host-based), WebSocket, microservices | ALB |
| **Classic/Gateway Load Balancer** | Varies | Legacy / third-party network appliances (firewalls, IDS) insertion | CLB / GWLB |

**L4 vs L7 load balancing (very common interview question):** An L4 load balancer makes routing decisions based only on IP address and port — extremely fast, protocol-agnostic, can't inspect HTTP content. An L7 load balancer understands the application protocol (typically HTTP/HTTPS) and can route based on URL path, hostname, headers, or cookies (e.g., `/api/*` → API backend, `/images/*` → image service) — more flexible but with slightly more overhead.

**Health checks:** Load balancers periodically probe backend targets (HTTP GET to a health endpoint, or a simple TCP connection) and automatically stop routing traffic to unhealthy targets — foundational to high availability.

**Cross-zone load balancing:** Determines whether the load balancer distributes traffic evenly across targets in *all* Availability Zones, or only evenly within each AZ it received traffic in — affects both even load distribution and (in some providers) cross-AZ data transfer cost.

---

## 15. Content Delivery Network (CDN)

A **CDN** (CloudFront on AWS, Azure CDN/Front Door, Cloud CDN on GCP) caches content at geographically distributed **edge locations** close to end users, reducing latency and offloading traffic from origin servers.

```
User (Tokyo) ──► Nearest Edge Location (Tokyo) ──► [cache hit: serve immediately]
                                                   ──► [cache miss: fetch from Origin (e.g., us-east-1), cache it, serve]
```

**Benefits:**
- Reduced latency (content served from a nearby edge location instead of a far-away origin region)
- Reduced load on origin servers (cached content doesn't hit the origin repeatedly)
- Built-in DDoS absorption at the edge
- Often integrates with WAF for edge-level security filtering

**Key concepts:** **TTL (Time To Live)** controls how long content is cached at the edge before re-validating with the origin; **cache invalidation** lets you force-purge stale cached content immediately when needed; CDNs can serve both **static** content (images, JS/CSS, video) and, increasingly, **dynamic** content via smarter caching/routing rules.

---

## 16. Private Endpoints / PrivateLink

A mechanism to access cloud provider services (e.g., a managed database, storage, or even another company's service in a different VPC/account) **privately**, without traffic ever traversing the public internet — even though the service technically lives outside your VPC.

```
Your VPC ──► VPC Endpoint (ENI with private IP) ──► AWS S3 / DynamoDB / other VPC's service
                  (traffic never leaves AWS's private network, no NAT Gateway/IGW needed)
```

**Two main types (AWS terminology, concepts apply broadly):**
- **Gateway Endpoints** — for S3 and DynamoDB specifically; added as a route table target, no extra cost
- **Interface Endpoints (PrivateLink)** — creates an Elastic Network Interface with a private IP inside your subnet, used for most other AWS services and for privately exposing **your own** service to other VPCs/accounts (a common SaaS architecture pattern — vendors expose their service via PrivateLink so customers never need to open it to the public internet)

**Why this matters (interview point):** Without a private endpoint, traffic from a private subnet to a service like S3 would need to go out through a NAT Gateway to the public internet and back in — adding cost, latency, and a broader security exposure surface. PrivateLink/VPC Endpoints keep that traffic entirely within the cloud provider's private network.

---

## 17. NAT Deep Dive

**Network Address Translation (NAT)** translates private IP addresses to public ones (and vice versa) so internal resources can communicate externally without each one needing its own public IP.

| NAT Type | Description |
|---|---|
| **1:1 (Static) NAT** | One private IP maps to exactly one public IP (used by Internet Gateways for public subnet resources) |
| **PAT (Port Address Translation)** / NAT Overload | Many private IPs share a single public IP, distinguished by source port (what a NAT Gateway does for outbound traffic from many private instances) |

This is exactly why a NAT Gateway only enables **outbound-initiated** connections: since many internal hosts share the gateway's single public IP via PAT, there's no way for an external host to initiate a new connection to a *specific* internal host — only return traffic for connections that internal hosts initiated can find their way back via the gateway's PAT translation table.

---

## 18. Firewalls & WAF

**Network firewalls** (Security Groups, NACLs, dedicated Network Firewall services) operate primarily at L3/L4 — filtering based on IP address, port, protocol.

**Web Application Firewall (WAF)** operates at L7 — inspects actual HTTP(S) request content to block application-layer attacks:
- SQL injection attempts
- Cross-site scripting (XSS)
- Known bad bot/IP signatures
- Rate limiting (block IPs making excessive requests)
- Geo-blocking at the application layer

```
WAF Rule example:
IF request contains SQL injection pattern → BLOCK
IF request rate from single IP > 100/min → RATE LIMIT
IF request originates from blocked country → BLOCK
```

WAFs are commonly deployed in front of a CDN or Application Load Balancer, inspecting traffic before it ever reaches your application servers.

---

## 19. Multi-Region & Multi-AZ Networking

**Availability Zone (AZ):** One or more discrete, physically separate data centers within a region, with independent power/cooling/networking — designed so a failure in one AZ doesn't take down others. Deploying across multiple AZs is the baseline for high availability within a single region.

**Region:** A geographically distinct area containing multiple AZs. Regions are isolated from each other by default (a VPC doesn't span regions) — used for disaster recovery, data residency/compliance requirements, and reducing latency for geographically distributed users.

**Cross-AZ vs Cross-Region traffic:**
- Cross-AZ traffic (within the same region) has low latency, but cloud providers typically **charge for data transferred between AZs** — an important real-world cost consideration in architecture decisions
- Cross-region traffic has materially higher latency (driven by physical distance/speed of light) and is typically more expensive per GB

**Common multi-region patterns:** Active-passive (DR site only activated on failover), active-active (traffic split across regions, often via latency-based DNS routing), and read-replica patterns (writes to a primary region, read replicas distributed globally for lower-latency reads).

---

## 20. Hybrid Cloud Networking

Connecting on-premises infrastructure with cloud infrastructure as a unified network — common in enterprises migrating gradually to the cloud or maintaining regulatory/legacy constraints requiring some on-prem presence.

**Components typically involved:**
- Site-to-Site VPN and/or Direct Connect/ExpressRoute for connectivity
- Consistent IP addressing strategy across on-prem and cloud (non-overlapping CIDR ranges are essential)
- DNS integration (often via DNS forwarding/conditional forwarding between on-prem DNS servers and cloud-managed DNS, or hybrid DNS resolver services)
- Identity federation (e.g., Active Directory extended into the cloud) for consistent access control
- Transit Gateway/Virtual WAN as the hub connecting multiple VPCs and the on-prem connection together

**Common interview scenario:** "Design connectivity for a company with a data center and multiple AWS VPCs that all need to talk to each other and to on-prem." Answer generally: Transit Gateway as the hub, VPCs attached to it, Direct Connect (primary) + Site-to-Site VPN (backup) from the Transit Gateway to on-prem, non-overlapping CIDR ranges planned up front, DNS forwarding configured between on-prem and Route 53 Resolver (or equivalent).

---

## 21. Network Monitoring & Flow Logs

**VPC Flow Logs** (AWS term; similar concepts exist across providers) capture metadata about IP traffic flowing through network interfaces in a VPC — source/destination IP, port, protocol, bytes transferred, accept/reject — **without** capturing actual packet payload content.

```
Flow log record fields (simplified):
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

**Use cases:**
- Security incident investigation (who talked to what, when)
- Troubleshooting connectivity issues (was traffic rejected by a security group/NACL, or did it never arrive?)
- Compliance/audit requirements
- Feeding into analysis tools (e.g., to detect unusual traffic patterns)

**Other monitoring tools:** Reachability analyzers (simulate whether traffic *would* be allowed between two points, useful for pre-deployment validation), Traffic Mirroring (copy and inspect actual packet-level traffic for deep debugging/security analysis), and standard cloud monitoring/metrics dashboards (bandwidth, packet loss, connection counts on load balancers/gateways).

---

## 22. IPv6 in the Cloud

Cloud providers increasingly support dual-stack (IPv4 + IPv6) VPC configurations.

**Key differences from IPv4 in cloud contexts:**
- IPv6 addresses are vastly more numerous — NAT is generally **not needed** for IPv6 (every resource can have a globally unique address), though an **egress-only internet gateway** concept exists (AWS) to allow outbound-only IPv6 internet access analogous to what a NAT Gateway provides for IPv4
- Security Groups/NACLs need separate IPv6 rules (an IPv4-only rule set won't cover IPv6 traffic)
- Many organizations run dual-stack during a transition period rather than going IPv6-only, due to legacy system/tooling support gaps

---

## 23. Cost Considerations (Data Transfer)

Cloud networking costs are a frequently underestimated/misunderstood part of cloud bills — a common practical interview/real-world topic.

| Traffic Type | Typical Cost Behavior |
|---|---|
| Within same AZ | Usually free (or very cheap) |
| Cross-AZ, same region | Charged both ways (a common surprise cost) |
| Cross-region | Charged, generally higher than cross-AZ |
| Data transfer OUT to internet | Charged, often the largest networking cost line item |
| Data transfer IN from internet | Usually free |
| Through NAT Gateway | Charged per GB processed, **in addition to** standard data transfer charges — can get expensive at scale |
| Through VPC Endpoints/PrivateLink | Often cheaper than the NAT Gateway + internet path for AWS-service traffic, plus more secure |

**Practical cost-optimization techniques:** Use VPC Endpoints/PrivateLink instead of routing AWS-service traffic through a NAT Gateway; be deliberate about cross-AZ traffic in latency/cost-sensitive high-throughput services (e.g., keep chatty services co-located in the same AZ where consistent with your HA requirements); use a CDN to reduce repeated data transfer out of the origin region; right-size and consolidate NAT Gateways.

---

## 24. Security Best Practices

1. **Principle of least privilege networking** — only open the specific ports/protocols/sources actually required (avoid `0.0.0.0/0` on anything beyond public-facing load balancers).
2. **Use private subnets for anything that doesn't need direct internet exposure** — databases, internal app servers, etc.
3. **Layer your defenses** — Security Groups (instance-level) + NACLs (subnet-level) + WAF (application-level) provide defense in depth, not redundant duplication.
4. **Use VPC Endpoints/PrivateLink** to avoid sending sensitive internal traffic over the public internet path, even when "encrypted."
5. **Segment networks by trust level/function** — separate subnets/VPCs for public-facing, application, and data tiers; separate VPCs per environment (dev/staging/prod) or per team/business unit where appropriate.
6. **Enable flow logs** on production VPCs for visibility and incident response readiness.
7. **Encrypt data in transit** — TLS for application traffic, IPsec for VPN tunnels, even for traffic that stays "internal" to the cloud provider's network where compliance requires it.
8. **Use bastion hosts or Session Manager/SSM-style access** instead of exposing SSH/RDP directly to the internet on instances in private subnets.
9. **Regularly audit security group rules** — overly permissive rules accumulate over time ("temporary" rules that are never removed are a very common real-world audit finding).
10. **Plan non-overlapping CIDR ranges** across all VPCs/on-prem networks from the start — overlapping ranges block peering/VPN connectivity later and are painful to re-architect.

---

## 25. Troubleshooting

**Common connectivity troubleshooting checklist (interview-relevant — many "design/debug" questions follow this exact flow):**
1. **Security Group** — does the relevant SG allow the traffic (right port, right source)?
2. **NACL** — does the subnet's NACL allow it in *both* directions (remember: stateless, ephemeral ports needed)?
3. **Route table** — does the subnet's route table have a path to the destination (local route, IGW, NAT, peering connection, or Transit Gateway attachment)?
4. **Internet/NAT Gateway** — if internet-bound, is the right gateway attached and referenced correctly?
5. **DNS resolution** — is the hostname resolving to the expected IP at all (rule out DNS before assuming a network path issue)?
6. **Target health** — if going through a load balancer, are the backend targets passing health checks?
7. **Host-level firewall** — don't forget the OS-level firewall (iptables/Windows Firewall) inside the instance itself, which cloud network controls don't manage.

```bash
# Common diagnostic commands
ping <destination>                 # basic reachability (note: ICMP may itself be blocked)
traceroute <destination>           # see where traffic stops along the path
telnet <destination> <port>        # test specific port connectivity
nslookup / dig <hostname>          # DNS resolution check
curl -v https://endpoint           # full HTTP-level diagnostic, see TLS/connection details
```

**Reachability Analyzer pattern (AWS, similar tools elsewhere):** A tool that statically analyzes your network configuration (route tables, security groups, NACLs) between a source and destination and tells you whether traffic *would* be allowed and, if not, exactly which component is blocking it — much faster than manual checklist debugging for complex setups.

---

## 26. Best Practices Summary

- Design with a tiered subnet structure (public / private-app / private-data)
- Plan CIDR ranges carefully and non-overlapping from day one, across all VPCs and on-prem
- Use Security Groups as your primary access control; treat NACLs as a coarse subnet-level backstop
- Use one NAT Gateway per AZ for production high availability (avoid cross-AZ NAT traffic)
- Use VPC Endpoints/PrivateLink for traffic to cloud-native services instead of routing through NAT/internet
- Use Transit Gateway (or equivalent) once you have more than a handful of VPCs needing interconnectivity
- Enable flow logs in production for visibility
- Use a CDN and WAF in front of public-facing applications
- Automate networking infrastructure via IaC (Terraform/CloudFormation) — manual network changes are error-prone and hard to audit
- Regularly review and tighten security group/NACL rules
- Understand and monitor data transfer costs, especially cross-AZ and NAT Gateway processing charges

---

## 27. Cheat Sheet / Terminology Map

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Virtual network | VPC | VNet | VPC |
| Subnet | Subnet | Subnet | Subnet |
| Instance-level firewall | Security Group (stateful) | NSG (stateful) | Firewall Rules (stateful) |
| Subnet-level firewall | NACL (stateless) | NSG can apply to subnet too | N/A (firewall rules are global/tag-based) |
| Public internet gateway | Internet Gateway | N/A (public IP direct) | N/A (implicit default internet gateway) |
| Outbound-only NAT | NAT Gateway | NAT Gateway | Cloud NAT |
| Cross-VPC private connection | VPC Peering | VNet Peering | VPC Peering |
| Multi-VPC hub | Transit Gateway | Virtual WAN | Network Connectivity Center |
| Site-to-site VPN | Site-to-Site VPN | VPN Gateway | Cloud VPN |
| Dedicated private link | Direct Connect | ExpressRoute | Cloud Interconnect |
| DNS service | Route 53 | Azure DNS | Cloud DNS |
| L7 Load Balancer | ALB | Application Gateway | HTTP(S) Load Balancer |
| L4 Load Balancer | NLB | Azure Load Balancer | Network Load Balancer |
| CDN | CloudFront | Azure CDN / Front Door | Cloud CDN |
| Private service access | PrivateLink / VPC Endpoint | Private Link | Private Service Connect |
| Traffic logging | VPC Flow Logs | NSG Flow Logs | VPC Flow Logs |
| WAF | AWS WAF | Azure WAF | Cloud Armor |

---

## 28. Interview Questions & Answers

**Q1: What is a VPC and why do cloud providers use this model?**
A: A Virtual Private Cloud is an isolated, logically-separated virtual network within the cloud provider's infrastructure, where you control IP addressing, subnets, routing, and security. It gives each customer (or each environment/application) a private, software-defined network boundary — equivalent to having your own data center network, but provisioned and managed entirely through APIs/configuration rather than physical hardware.

**Q2: Explain the difference between a public and a private subnet.**
A: The distinction is purely about routing, not naming: a public subnet's route table sends `0.0.0.0/0` traffic to an Internet Gateway, allowing resources with public IPs to be directly reachable from the internet. A private subnet has no route to an Internet Gateway — resources there have no direct inbound internet exposure, and outbound internet access (if needed) goes through a NAT Gateway instead.

**Q3: What's the difference between a Security Group and a Network ACL?**
A: Security Groups operate at the instance/resource level and are **stateful** — if inbound traffic is allowed, the corresponding return traffic is automatically allowed without a separate outbound rule. NACLs operate at the subnet level and are **stateless** — you must explicitly allow both inbound and outbound traffic, including the ephemeral port range for return traffic. NACLs also support explicit deny rules and are evaluated in numbered order, while Security Groups only support allow rules with all matching rules combined.

**Q4: What's the difference between an Internet Gateway and a NAT Gateway?**
A: An Internet Gateway provides full bidirectional internet connectivity for resources with a public IP — used by public subnets. A NAT Gateway allows resources in a private subnet (no public IP) to initiate outbound connections to the internet, but nothing on the internet can initiate an inbound connection back to them — it enables one-way, outbound-initiated access only.

**Q5: Why is VPC Peering not transitive, and what problem does Transit Gateway solve?**
A: A VPC Peering connection is a direct, point-to-point link — if A is peered with B and B is peered with C, A cannot reach C through B, because peering doesn't propagate routing transitively; each pair needs its own explicit peering connection, which doesn't scale well (N² connections for N VPCs). A Transit Gateway acts as a central hub that all VPCs attach to once, enabling any-to-any (transitive) routing between attached VPCs through a single managed hub, scaling far more cleanly.

**Q6: When would you use a Site-to-Site VPN versus Direct Connect/ExpressRoute?**
A: Site-to-Site VPN is faster to provision (minutes/hours), cheaper, and runs encrypted over the public internet — suitable for lower-bandwidth needs, quick setups, or as a backup connection. Direct Connect/ExpressRoute is a dedicated private physical circuit bypassing the public internet entirely — offering consistent, higher bandwidth and lower, more predictable latency, but takes weeks to provision and costs more — used for large, steady data transfer needs or latency-sensitive/compliance-driven workloads. A common pattern is Direct Connect as primary with VPN as automatic failover.

**Q7: What's the difference between Layer 4 and Layer 7 load balancing?**
A: Layer 4 (Network) load balancing routes traffic based only on IP address and port, without inspecting application-layer content — fast and protocol-agnostic. Layer 7 (Application) load balancing understands the actual application protocol (typically HTTP/HTTPS) and can route based on URL path, hostname, headers, or cookies — enabling content-based routing to different backend services, at the cost of slightly more processing overhead.

**Q8: How does a CDN improve performance, and what is a cache hit vs miss?**
A: A CDN caches content at edge locations geographically close to end users. On a **cache hit**, the requested content is already cached at the nearby edge and served immediately, with low latency and no load on the origin. On a **cache miss**, the edge location doesn't have a valid cached copy, so it fetches the content from the origin server, caches it for future requests, and serves it to the user — slower for that first request but fast for subsequent ones until the cache entry's TTL expires.

**Q9: What is a VPC Endpoint / PrivateLink, and why would you use one instead of going through a NAT Gateway?**
A: A VPC Endpoint creates a private connection from your VPC directly to a cloud provider service (or another VPC's exposed service via PrivateLink) without that traffic ever traversing the public internet — even though the destination service technically isn't "inside" your VPC. Compared to routing that traffic out through a NAT Gateway and the public internet, a VPC Endpoint is typically more secure (traffic stays on the provider's private network), often cheaper (avoids NAT Gateway per-GB processing charges), and lower latency.

**Q10: Explain the difference between an Availability Zone and a Region.**
A: A Region is a distinct geographic area; an Availability Zone (AZ) is one or more physically separate data centers within a region, with independent power, cooling, and networking, designed so a failure in one AZ doesn't affect others. Applications are typically deployed across multiple AZs within a region for high availability, and across multiple regions for disaster recovery, data residency compliance, or reducing latency for geographically distributed users.

**Q11: Why are NAT Gateways typically deployed one-per-AZ in production rather than a single shared one?**
A: A NAT Gateway is itself scoped to a single AZ. Using just one NAT Gateway for an entire multi-AZ VPC creates a single point of failure (if that AZ has an outage, private subnets in other AZs lose outbound internet access too) and incurs cross-AZ data transfer charges for traffic from instances in other AZs routing through it. Deploying one NAT Gateway per AZ, with each AZ's private subnets routing to their own local NAT Gateway, avoids both issues.

**Q12: What information does a VPC Flow Log capture, and what does it NOT capture?**
A: Flow logs capture **metadata** about IP traffic — source/destination IP and port, protocol, packet/byte counts, timestamps, and whether the traffic was accepted or rejected. They do **not** capture the actual packet payload/content — so they're useful for understanding traffic patterns, debugging connectivity (was it rejected by a security control, or never arrived at all), and security investigation, but not for inspecting the actual data being transmitted.

**Q13: How would you design network connectivity for a company with on-premises infrastructure and multiple VPCs that all need to communicate?**
A: Use a Transit Gateway (or equivalent hub service) as the central connectivity point — attach all VPCs to it, and connect on-premises via Direct Connect (primary, for consistent bandwidth/latency) with a Site-to-Site VPN as automatic failover backup. Plan non-overlapping CIDR ranges across every VPC and the on-prem network up front, and configure DNS forwarding between on-prem DNS and the cloud's DNS resolver so hostnames resolve correctly across the hybrid environment.

**Q14: What's the difference between Gateway Endpoints and Interface Endpoints (AWS-specific, but the concept generalizes)?**
A: Gateway Endpoints are specifically for S3 and DynamoDB — implemented as a route table target, with no additional cost. Interface Endpoints (PrivateLink) create an actual network interface with a private IP inside your subnet, used for most other services, and are also the mechanism used to privately expose your own services to other VPCs/accounts (a common pattern for SaaS providers who want customers to reach their service without traversing the public internet).

**Q15: A user reports they can't connect to an application running on a private EC2 instance behind a load balancer. How would you troubleshoot?**
A: Work through the path systematically: confirm DNS resolves to the expected load balancer; check the load balancer's security group allows inbound traffic on the relevant port from the user's source; check the load balancer's target group health checks — are the backend instances passing; check the instances' security group allows traffic from the load balancer's security group/IP; check the subnet's NACL allows the traffic in both directions, including ephemeral return ports; check the route tables have valid paths between all the involved subnets; and finally check for any host-level firewall (e.g., iptables) on the instance itself that cloud network controls wouldn't show you.

**Q16: Why might cross-AZ traffic be a meaningful cost and architecture consideration?**
A: Most cloud providers charge for data transferred between Availability Zones within the same region (both directions), even though it's "internal" traffic — a cost that's easy to overlook but can add up significantly for high-throughput, chatty services (e.g., frequent calls between microservices, or distributed database replication). It's also a latency consideration, though far smaller than cross-region. Architects often weigh keeping tightly-coupled, high-traffic services co-located within an AZ against the availability benefits of spreading across AZs.

**Q17: What's the purpose of a WAF, and how is it different from a Security Group?**
A: A Web Application Firewall (WAF) operates at Layer 7, inspecting actual HTTP(S) request content to block application-layer attacks like SQL injection, XSS, bad bot traffic, or excessive request rates from a single source. A Security Group operates at Layer 3/4, filtering only based on IP address, port, and protocol — it has no visibility into the actual content of allowed traffic. They're complementary, not redundant: a Security Group might correctly allow all traffic on port 443, while a WAF inspects what's actually inside those HTTPS requests for malicious patterns.

**Q18: How does DNS-based latency routing work, and when would you use it?**
A: Latency-based DNS routing (e.g., Route 53's latency policy) returns different IP addresses to users depending on which of your deployed regions/endpoints currently offers the lowest network latency from the user's resolving DNS server location — used in multi-region active-active architectures to automatically direct each user to their geographically/network-closest deployment, improving response times without requiring the application itself to handle that routing logic.

**Q19: What does "stateful" mean in the context of a Security Group, and why does it matter practically?**
A: Stateful means the Security Group automatically tracks established connections and allows the return traffic for any connection that was permitted by an inbound (or outbound) rule, without needing a matching explicit rule in the opposite direction. Practically, this means if you allow inbound traffic on port 443, you don't need a separate outbound rule for the response — simplifying configuration significantly compared to a stateless firewall like a NACL, where you'd have to explicitly allow the ephemeral return ports as well.

**Q20: Why is planning non-overlapping CIDR ranges important before building out cloud network architecture?**
A: VPC Peering and most VPN/hybrid connectivity options require that the connected networks have non-overlapping IP address ranges — overlapping CIDRs make it impossible to establish routing between them (the network layer can't tell which network a given IP address actually belongs to). Since changing a VPC's CIDR after resources are already deployed is disruptive and sometimes impossible without rebuilding, IP address planning needs to happen upfront, accounting for all VPCs, accounts, and on-premises networks that might eventually need to interconnect.

---

### Final interview tip
Be ready to **draw a typical 3-tier VPC architecture from memory** (public subnet with ALB/NAT, private app subnet, private database subnet, each across 2+ AZs), clearly explain **Security Groups (stateful) vs NACLs (stateless)**, and walk through the **troubleshooting checklist** for "traffic isn't reaching my instance" step by step (DNS → route table → security group → NACL → target health → host firewall). These two come up constantly in both conceptual and scenario/whiteboard-style interview questions.
