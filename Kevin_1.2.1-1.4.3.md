---
topic: SD-WAN, SD-Access, and QoS Overview
exam-domain: 1.0 Architecture
tags:
  - sd-wan
  - sd-access
  - lisp
  - vxlan
  - qos
  - mqc
  - v1.2-gap
sources:
  ocg: "Ch 23 (SD-Access/SD-WAN, v1.1 print — Catalyst SD-WAN naming is a v1.2 gap); QoS chapter"
  wallace: "ENCOR v1.2 series — SD-WAN/SD-Access + QoS sections"
  jeremy: "No JITL pairing — route second pass to CBT Nuggets"
status: reviewed
---

# Overview of SD-WAN Technologies

## Why SD-WAN Exists

**Traditional WAN:** Hub-and-spoke topology with headquarters as the central point.

**The cloud shift:** Applications moved to AWS, Azure, and Google Cloud.

**Traffic backhauling:** All traffic routed through headquarters (even cloud-bound traffic), causing latency and performance issues.

**The problem:** Why backhaul traffic to HQ when the destination is the cloud?

## Core SD-WAN Benefits

- **Transport independence:** Multiple transport options (MPLS, broadband, LTE) used simultaneously, providing flexibility and redundancy.
- **Overlay abstraction:** Build logical topologies on top of physical infrastructure.
- **Direct cloud access:** Intelligent routing sends cloud traffic directly to the cloud instead of backhauling.
- **Centralized management:** Configure and monitor everything from one console.

![SD-WAN architecture](<images/sdwan.png>)

## Cisco Catalyst SD-WAN Architecture (Four Planes)

> v1.2 note: Cisco renamed all components. The blueprint uses the **new names**, but practice questions may still use the old ones — know both.

| Plane | v1.2 Name | Legacy Name | Role |
|---|---|---|---|
| Management | Catalyst SD-WAN Manager | vManage | Single-console GUI: configuration, monitoring, templates |
| Control | Catalyst SD-WAN Controller | vSmart | Distributes routes and policy to edges via **OMP** |
| Orchestration | Catalyst SD-WAN Validator | vBond | Authenticates and onboards devices; enables **ZTP**; NAT traversal |
| Data | Catalyst SD-WAN edge routers (cEdge) | vEdge (older HW) | Forward user traffic over IPsec tunnels between sites |

**OMP (Overlay Management Protocol):** The control-plane protocol between the Controller and edge routers. Carries routes, TLOCs, and policy — the edges never peer directly with each other for control information.

**Key contrast (Anki):** Controller (vSmart) handles the *control plane*; Validator (vBond) handles *orchestration/onboarding*; Manager (vManage) is *management only* — it never forwards traffic or distributes routes itself.

## SD-WAN Deployment Models

- **On-premises:** All SD-WAN controllers deployed in the organization's own data centers and managed internally.
- **Cloud-hosted:** Controllers hosted in the cloud (by Cisco or a provider), reducing on-prem infrastructure and management burden. Most common model.
- **Hybrid:** Combines on-premises and cloud-hosted components based on organizational needs.

**Zero Touch Provisioning (ZTP) workflow:** A new edge device powers on, gets an IP via DHCP, reaches the **Validator (vBond)** for authentication, is directed to the **Manager** and **Controller**, and pulls its full configuration automatically — no manual on-site configuration required.

---

# SD-Access Fabric Architecture

## The Problem with Traditional Access

Manually configured access with ACLs ties permissions to **IP addresses**. If a user with server access moves to another department or changes IP, their access breaks (or worse, follows the old IP to someone else).

**SD-Access moves policy away from *where you are* (IP) to *who you are* (identity).**

## Identity: Cisco ISE and SGTs

**Cisco ISE (Identity Services Engine)** assigns each endpoint a **Scalable Group Tag (SGT)** based on authentication:

- **802.1X** — user credentials or certificates (the primary, identity-based method)
- **MAB (MAC Authentication Bypass)** — fallback for devices that can't do 802.1X (printers, IoT); this is the only case where the MAC address drives the decision

The SGT is carried inside the **VXLAN-GPO header**, so policy follows the user anywhere in the fabric regardless of IP or physical location. This is what solves the "moved departments" problem.

## Fabric Building Blocks

**LISP (Locator/ID Separation Protocol):** A routing architecture that separates a device's *identity* from its *network location* to improve scalability, mobility, and traffic engineering.

- **EID (Endpoint Identifier):** The IP address assigned to an endpoint — identifies the device itself, independent of location.
- **RLOC (Routing Locator):** The IP address of the network device (fabric node) indicating *where* the endpoint currently sits in the topology.

**VXLAN (Virtual Extensible LAN):** An encapsulation protocol that creates Layer 2 overlay networks over Layer 3 infrastructure. SD-Access uses **VXLAN-GPO**, which adds the SGT field to the header.

- **Underlay network:** The physical devices and routed links (typically IS-IS in SD-Access).
- **Overlay network:** The virtual tunnels built on top (VXLAN).

**Core contrast (Anki):**

| Function | Protocol | Carries |
|---|---|---|
| Control plane | **LISP** | EID-to-RLOC mappings ("where is this endpoint?") |
| Data plane | **VXLAN (GPO)** | Actual user traffic + SGT for policy |
| Policy plane | **Cisco TrustSec / ISE** | SGT assignment and enforcement |

## Fabric Roles

![SD-Access fabric](<images/sdaccess.png>)

![Fabric roles](<images/Roles.png>)

- **Control plane node:** Runs the LISP map server/map resolver. The fabric's "directory" — tracks which EID is behind which RLOC.
- **Border node:** Connects the fabric to external networks (rest of the enterprise, data center, internet). Translates between fabric and non-fabric domains.
- **Edge node:** The access switch where endpoints connect. Authenticates endpoints (with ISE), applies SGTs, and encapsulates/decapsulates VXLAN.
- **Fabric WLC + fabric-mode APs:** Bring wireless into the fabric; APs tunnel client data in VXLAN to the edge node rather than CAPWAP-tunneling all data to the WLC.

**Requirement:** SD-Access = fabric + **Cisco Catalyst Center** (formerly DNA Center) for automation and assurance. Without Catalyst Center, you just have a manually built fabric — not SD-Access.

---

# QoS Mechanisms

## When QoS Helps

QoS only helps with **periodic congestion**. If a link is *always* congested (e.g., 200 Mbps of traffic on a 100 Mbps link), QoS cannot fix it — you need more bandwidth. QoS decides who suffers during temporary contention; it doesn't create capacity.

## The Three QoS Models

> Exam trap: there are **three** models. RSVP is *not* a fourth model — it is the signaling protocol that IntServ uses.

| Model | Strictness | Mechanism |
|---|---|---|
| **Best Effort** | None | Default — all traffic treated equally, no prioritization |
| **DiffServ** | Less strict | Classifies and marks packets (**DSCP** in the IP header) for per-hop differentiated treatment; scalable, most common |
| **IntServ** | Most strict | Per-flow resource reservation using **RSVP** signaling; guarantees bandwidth but scales poorly |

## Common QoS Mechanisms

**Classification:** Identifying and categorizing packets based on characteristics (source/destination IP, port, protocol, NBAR application) to determine treatment.

**Marking:** Modifying packet headers (DSCP, CoS) to indicate priority so downstream devices can act on it.

**Congestion management (queuing):** Organizing packets into queues so higher-priority traffic transmits first.

- **LLQ (Low Latency Queuing):** Adds a strict-priority queue to CBWFQ for time-sensitive traffic (voice/video) to minimize latency. The priority queue is policed so it can't starve other classes.
- **CBWFQ (Class-Based Weighted Fair Queuing):** Allocates guaranteed minimum bandwidth per traffic class.

**Congestion avoidance:** Proactively dropping packets *before* queues fill to prevent TCP global synchronization — **RED** and **WRED** (WRED drops lower-priority DSCP values earlier).

**Policing and shaping:** Rate-limiting traffic. **Policing drops** (or re-marks) excess traffic; **shaping buffers/delays** it for later transmission. Shaping is typically outbound toward a provider; policing is typically inbound or at the provider edge.

**Link efficiency:** Squeezing more from low-bandwidth links via compression and fragmentation.

- **cRTP (Compressed RTP):** Caches the mostly-static RTP/UDP/IP headers and sends only what changes, saving bandwidth on voice links.
- **LFI (Link Fragmentation and Interleaving):** Breaks large packets into fragments and interleaves small time-sensitive packets between them, reducing serialization delay. Relevant on links **≤ 768 kbps**.

## Modular QoS CLI (MQC) — Three-Step Process

1. **Classify** the traffic → `class-map`
2. **Define the policy** per class → `policy-map`
3. **Apply** to an interface → `service-policy`

**match-all vs. match-any (Anki):** `match-all` = every match statement must be true (logical AND). `match-any` = any single match statement suffices (logical OR).

### Step 1 — Classify (class-map)

```
conf t
class-map match-any EMAIL
 match protocol pop3
 match protocol smtp
 match protocol imap
 match protocol exchange
 exit
class-map VOICE
 match protocol rtp
 exit
class-map SCAVENGER
 match protocol bittorrent
 exit
```

### Step 2 — Define policy (policy-map)

```
policy-map DEMO
 class EMAIL
  bandwidth 512
  random-detect dscp-based
  exit
 class VOICE
  priority 256
  exit
 class SCAVENGER
  police 128000
  exit
```

> **Units trap (Anki):** `bandwidth 512` and `priority 256` are in **kbps**, but `police 128000` is in **bps**. Also note: `priority` = LLQ strict-priority queue; `bandwidth` = CBWFQ guaranteed minimum; `random-detect dscp-based` = WRED congestion avoidance within the class.

### Step 3 — Apply (service-policy)

```
int g0/2
 service-policy output DEMO

show policy-map interface g0/2
```

