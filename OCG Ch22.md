---
topic: Enterprise Network Architecture
exam-domain: 1.0 Architecture
sources:
  ocg: "Ch. 22 (Enterprise Network Architecture)"
status: reviewed
---

# Enterprise Architecture

## Hierarchical LAN Design

In hierarchical LAN design, the network is divided into three layers: **Core**, **Distribution**, and **Access**. Each layer has specific functions and responsibilities, and each layer is built to be modular and scalable. Modularity means a failure or change in one block (e.g., one access/distribution pair) doesn't ripple through the whole network.

### Access Layer

The access layer is where endpoints connect to the network — PCs, laptops, IP phones, printers, wireless APs, cameras.

- In most cases these devices get **1 Gbps** connections and rarely use all of that bandwidth, but if there is a burst of traffic, the access layer can absorb it. Providing enough headroom for bursts is part of delivering good **QoE (Quality of Experience)**.
- Other access layer responsibilities:
  - **PoE** for phones, APs, and cameras
  - **QoS trust boundary** — marking/classifying traffic as close to the source as possible
  - **Security services**: 802.1X / NAC, port security, DHCP snooping, Dynamic ARP Inspection
  - **VLAN assignment** and IGMP snooping for multicast
- Uplinks to the distribution layer are oversubscribed by design (typical guideline ~20:1 access-to-distribution, ~4:1 distribution-to-core).

### Distribution Layer

The distribution layer acts as a bridge between the access layer and the core layer.

- It is responsible for **aggregating the access layer switches**. Depending on the architecture, it moves traffic through Layer 2 or Layer 3 domains (STP or IP routing) — it's typically the **L2/L3 boundary** of the network.
- Implements **policy and security**: ACLs, filtering, QoS policy, VLAN segmentation.
- Performs **route summarization** toward the core to keep the routing table small and contain topology changes.
- Provides **redundancy and load balancing** (dual distribution switches, FHRPs or routed uplinks) for high availability.
- By having distribution switches, you reduce the number of fiber runs needed between the access layer and the core — full-mesh cabling from every access switch to the core would be expensive and complex.

![reduce links](<images/links.png>)

### Core Layer

The core layer is the **backbone** of the network.

- Responsible for aggregating the different distribution blocks/locations and providing high-speed connectivity between them.
- Designed to be **fast and simple**: minimal latency, high throughput, and as little policy/complexity as possible (no CPU-intensive features like heavy ACL processing here).
- Should never be a single point of failure — always redundant.

## Two-Tier vs. Three-Tier Design

- **Three-tier**: Access → Distribution → Core. Used in large campuses with multiple buildings/distribution blocks. A dedicated core makes sense once you have three or more distribution blocks to interconnect.
- **Two-tier ("collapsed core")**: The core and distribution functions are combined ("collapsed") into one layer. Common in small/medium campuses — cheaper, simpler, but scales poorly if the site grows.

## Layer 2 Access vs. Layer 3 (Routed) Access

The key design question: **where does the L2/L3 boundary live?**

### Layer 2 Access (traditional / STP-based)

- Access switches are **pure L2**. Trunks (or access links) carry VLANs up to the distribution layer.
- The **distribution layer is the L2/L3 boundary**: it holds the SVIs, acts as the default gateway, and runs an **FHRP** (HSRP/VRRP/GLBP) between the two distribution switches.
- **STP** manages the redundant uplinks. Two sub-designs:
  - **Loop-free topology**: each VLAN exists on only one access switch → no loop, no blocked ports, but VLANs can't span access switches.
  - **Looped topology**: VLANs span multiple access switches → a real loop exists, so **STP blocks one uplink**. You pay for bandwidth you can't use.
- Convergence depends on **STP + FHRP timers** — slower and less deterministic than routing.
- Requires the STP toolkit for stability: PortFast, BPDU Guard, Root Guard, etc.
- Why still use it? Some environments **need VLANs to span access switches** (legacy apps, certain clustering/mobility requirements).

### Layer 3 (Routed) Access

- The **L2/L3 boundary moves down to the access switch**. Access switches hold the SVIs and are the default gateway for their own VLANs.
- Uplinks to distribution are **routed point-to-point links** running OSPF/EIGRP/IS-IS.
- Consequences:
  - **No FHRP needed** — the gateway is the access switch itself.
  - **No STP blocking** between access and distribution — both uplinks are active and load-balanced via **ECMP**.
  - **Faster, deterministic convergence** driven by the routing protocol, not STP/FHRP timers.
  - Easier troubleshooting — you can `ping`/`traceroute` hop by hop.
- Limitations:
  - **VLANs cannot span multiple access switches** (each VLAN is local to one switch).
  - Requires multicast routing if multicast must cross switches, and historically a higher software license on access switches.
- Routed access is the foundation of the **SD-Access underlay** (Ch. 23).

### Quick contrast (Anki fodder)

| | L2 access | L3 routed access |
|---|---|---|
| L2/L3 boundary | Distribution | Access |
| Default gateway | Distribution SVI + FHRP | Access switch SVI |
| Redundant uplink usage | One blocked by STP (looped design) | Both active (ECMP) |
| Convergence driven by | STP + FHRP timers | Routing protocol |
| VLAN spanning access switches | Possible | Not possible |
| FHRP required | Yes | No |

## Simplified Campus Design — Switch Stacking & StackWise Virtual

The idea: collapse multiple physical switches into **one logical switch** with a single control plane. This removes the FHRP-vs-STP-vs-routing debate at that layer entirely.

### StackWise (access layer stacking)

- Physically stacks access switches (e.g., Catalyst 9300 with **StackWise-480/1T**) using stack cables in a **closed ring** in the back of the switches.
- Up to **8 switches** act as one: **one active** switch runs the control plane, **one standby** is ready to take over (SSO within the stack), the rest are **members**.
- One management IP, one config file, one logical switch to your neighbors.
- Benefits:
  - **Cross-stack EtherChannel** — an uplink bundle can use ports on different physical stack members, so losing one member doesn't kill the uplink.
  - Add/remove/replace a member switch without taking the stack down (pay-as-you-grow port density).
  - The stack ring carries traffic between members at high bandwidth (480 Gbps / 1 Tbps depending on platform).
- Related: **StackPower** lets stacked 9300s pool their power supplies — a switch that loses its PSU can draw from the shared pool.

### StackWise Virtual / VSS (distribution & core)

- Joins **two chassis** (distribution or core switches) into one logical switch over the **StackWise Virtual Link (SVL)** — the modern successor to VSS.
- Downstream switches dual-home to both chassis with a single **Multichassis EtherChannel (MEC)**. From STP's point of view there's **one switch and one link**, so:
  - **No blocked ports** — both physical uplinks forward.
  - **No FHRP needed** — one logical gateway.
  - One control plane / one config for the pair.
- **Dual-active detection (DAD)** is required: if the SVL fails, both chassis could think they're active ("split brain"). DAD (fast hello on a separate link, or enhanced PAgP through a MEC neighbor) detects this and puts one chassis into recovery mode.
- Big picture: stacking at access + StackWise Virtual at distribution gives you a **loop-free, FHRP-free, all-links-forwarding** campus without going full routed access.

### SD-Access

Cisco's software-defined campus fabric built on top of a routed access underlay (covered in depth in Ch. 23).

## High Availability (HA) Network Design

Acronyms:

| Acronym | Meaning |
|---|---|
| RP | Route Processor |
| SSO | Stateful Switchover |
| NSF | Nonstop Forwarding |
| GR | Graceful Restart |
| NSR | Nonstop Routing |
| BFD | Bidirectional Forwarding Detection |
| UDLD | UniDirectional Link Detection |

### Hardware redundancy

- Redundant components: fans, power supplies, network interfaces/uplinks.
- Modular chassis with **hot-swappable line cards** so a failed card can be replaced without downtime.
- **Dual RPs (supervisors)** so the control plane survives a supervisor failure.

### SSO — the foundation

**SSO (Stateful Switchover)** keeps the **standby RP fully synchronized** with the active RP: startup/running config plus **Layer 2 protocol state** (STP, trunks, EtherChannels, line protocol state). If the active RP fails, the standby takes over in roughly a second and **L2 forwarding continues** uninterrupted.

The catch: **SSO by itself does nothing for Layer 3.** Routing protocol state (OSPF adjacencies, EIGRP neighbors, BGP sessions) is *not* synced. On switchover the routing processes restart, and how the network survives that restart is exactly what NSF/GR and NSR address. Two roles matter:

- The **restarting router** (the one whose RP switched over)
- Its **neighbors** (do they help or not?)

### NSF + GR — "keep forwarding, and neighbors help me rebuild"

- **NSF (Nonstop Forwarding)** is the *data-plane* half: during the switchover, the router **keeps forwarding packets using its existing CEF/FIB entries** even though the routing control plane is down/rebuilding. The FIB is only updated once the routing protocol has reconverged (stale entries are then refreshed or removed).
- **GR (Graceful Restart)** is the *control-plane* half — protocol extensions that let the restarting router tell its neighbors: *"I'm restarting. Don't tear down our adjacency, keep forwarding traffic to me, and help me relearn my routes."* Each protocol has its own mechanism (OSPF grace-LSA, EIGRP RS-bit in hellos, BGP GR capability, and there's an IETF and a Cisco flavor for several of them).
- Terminology: the restarting router is **NSF-capable**; a neighbor that understands GR and plays helper is **NSF-aware**. **GR only works if the neighbors are NSF-aware.**

### NSF *without* GR — why it mostly falls apart

If the neighbors are **not** NSF-aware, the restarting router can still keep forwarding on its stale FIB (NSF is local), but the neighbors don't know a graceful restart is happening. From their perspective the adjacency just **dies**: hellos stop, dead timers expire, they tear down the neighborship, flood the topology change, and **route around the restarting router**. Then the router comes back, adjacencies re-form, and everything converges *again*.

Net result: traffic is pulled away and the network flaps twice. NSF without GR-aware neighbors gives you very little — which is why the OCG always talks about **"NSF with GR"** as a package, and why it needs both SSO on the local box *and* cooperation from the neighbors.

One design caveat for GR: it assumes the **topology stays stable during the restart window**. The restarting router is forwarding on frozen, stale FIB entries — if the topology changes mid-restart, that stale forwarding can briefly blackhole or loop traffic.

### NSR — "sync everything internally, neighbors never know"

**NSR (Nonstop Routing)** takes the opposite approach: instead of asking neighbors for help, the active RP **continuously replicates routing protocol state to the standby RP** — adjacency/session state, and for BGP even the underlying TCP session state. On switchover, the standby continues the sessions **transparently**. Neighbors never see the adjacency flap and never know a switchover happened.

- **No neighbor cooperation required** — works even when peers aren't NSF-aware or are outside your control (e.g., a customer's router, another vendor's gear).
- Cost: **higher CPU and memory** on the RPs and constant inter-RP synchronization traffic. It's the "heavier" option, entirely self-contained (Cisco-internal implementation, not a protocol extension).
- General guidance: prefer **NSF/GR when neighbors support it** (cheaper); use **NSR when they don't** or when you can't rely on them.

### Comparison table

| | SSO alone | SSO + NSF/GR | SSO + NSR |
|---|---|---|---|
| L2 state preserved | Yes | Yes | Yes |
| L3 forwarding during switchover | FIB present, but neighbors reroute | Continues on existing FIB | Continues, sessions intact |
| Routing adjacencies | Drop and re-form | **Maintained by helpers** (GR) | **Never drop** (synced to standby) |
| Neighbor cooperation needed | — | **Yes (NSF-aware neighbors)** | **No** |
| Resource cost | Low | Low–moderate | **High (CPU/RAM, RP sync)** |
| Mechanism type | Local redundancy | Protocol extensions on the wire | Internal state replication |
| When to use | L2-only roles | Neighbors are NSF-aware | Neighbors not NSF-aware / not yours |

### Fast failure detection

- **BFD**: Lightweight hello protocol that detects link/neighbor failures in **sub-second** time and notifies the routing protocol (OSPF, EIGRP, BGP, etc.) much faster than protocol hello timers.
- **UDLD**: Detects **unidirectional links** (common fiber failure mode where one strand breaks) and err-disables the port before STP loops or blackholing can occur.

### First-Hop Redundancy Protocols (FHRPs)

FHRPs like **HSRP, VRRP, and GLBP** provide gateway redundancy. Multiple routers work together to present a single **virtual IP** that hosts use as their default gateway. If the active router fails, the standby takes over and traffic keeps flowing without hosts noticing.

- **HSRP** — Cisco proprietary; active/standby.
- **VRRP** — open standard (multi-vendor); master/backup.
- **GLBP** — Cisco proprietary; adds per-host **load balancing** using one **AVG** (Active Virtual Gateway) that hands out virtual MACs of up to four **AVFs** (Active Virtual Forwarders).

