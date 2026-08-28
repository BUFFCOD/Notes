---
tags: [architecture, high-availability, fhrp, sso, cloud]
blueprint: [1.1.a, 1.1.b]
---

# 1.1.2 On-Premises vs. Cloud Design

When deploying IT infrastructure, organizations often face a decision between on-premises and cloud-based solutions. Each approach has advantages and disadvantages that significantly impact performance, cost, and scalability.

## On-Premises Design

- **Control**: Complete control over hardware, software, and data, allowing customization and optimization for specific business needs.
- **Security**: Organizations can implement their own security measures and protocols, which may be necessary for industries with strict regulatory requirements.
- **Performance**: Lower latency and higher performance for applications that require real-time processing or high data throughput.
- **Cost**: The initial capital expenditure is high, but ongoing operational costs may be lower long-term, especially for stable workloads.

## Cloud Design

- **Scalability**: Resources can easily scale up or down based on demand — ideal for fluctuating workloads.
- **Cost efficiency**: Pay-as-you-go pricing avoids large upfront costs; you pay only for the resources you use. This can mean significant savings, especially for startups and small businesses.
- **Accessibility**: Accessible from anywhere with an internet connection, enabling remote work and collaboration across locations.
- **Maintenance**: The cloud provider handles maintenance, updates, and security patches, freeing internal IT teams to focus on strategic initiatives.

Nowadays, many companies are moving some of their infrastructure to the cloud.

**Q: How do you access data securely in a cloud environment?**

- VPN
- Private WAN (Metro Ethernet, MPLS)
## Things to Consider

- **Cloud**: No need for redundant hardware and power; you pay for what you *use*, not what you *have*.
- **On-premises**: Easier for compliance and data sovereignty; easier to deliver a good user experience for applications requiring low latency and high performance, since the infrastructure is on-site.
- **Hybrid deployments** combine both on-premises and cloud solutions.

**Q: How do you switch between different cloud providers?**

Connect to an **Intercloud Exchange (ICX)**, which can switch between cloud providers for you. This avoids vendor lock-in and improves redundancy and disaster recovery options.

---

# Redundant Design

## Backup Types

| Type | What It Backs Up |
|---|---|
| **Full** | All data |
| **Differential** | All changes since the last *full* backup |
| **Incremental** | All changes since the last backup of *any* type (full or incremental) |
| **Snapshot** | The entire server, including state information |

## Disaster Recovery Sites

- **Cold site**: Not operational; requires time to set up after a disaster. Only basic infrastructure (power, HVAC, space) — no hardware, no data.
- **Warm site**: Power, HVAC, space, *and* server hardware — but data must be restored before operations resume.
- **Hot site**: Fully operational and ready to take over immediately. All infrastructure — servers, storage, network connectivity — is in place, allowing business to resume with minimal downtime.

## Key Metrics & Agreements

- **SLA (Service Level Agreement)**: A contract between a service provider and customer outlining expected service levels, including uptime guarantees, response times, and support availability.
- **RTO (Recovery Time Objective)**: The maximum acceptable downtime after a disaster before business operations are significantly impacted. Drives backup and recovery strategy.
- **RPO (Recovery Point Objective)**: The maximum acceptable amount of data loss in a disaster. Drives backup frequency and data replication strategy.
- **MTBF (Mean Time Between Failures)**: The average time between failures — a measure of reliability used to plan maintenance and replacement schedules.
- **MTTR (Mean Time to Repair)**: The average time to repair a system after a failure — used to assess maintenance efficiency and plan for downtime.

---

# First Hop Redundancy Protocols (FHRP)

FHRPs allow multiple routers to present the appearance of a single virtual router (virtual IP + virtual MAC) to hosts on the LAN. If the active router fails, another takes over with minimal disruption. They provide **default gateway redundancy**, and some also provide **load balancing**.

> **Exam note:** For CCNP you need to know how to *configure* HSRP and VRRP. GLBP is Cisco-proprietary and not required for the ENCOR exam.

## HSRP (Hot Standby Router Protocol)

- **Cisco-proprietary.**
- One router is **Active**, another is **Standby**; the active router forwards traffic sent to the virtual IP.
- Default timers: **3-second hello**, **10-second hold**.

## VRRP (Virtual Router Redundancy Protocol)

- **Open standard** — used in multi-vendor environments; provides functionality similar to HSRP.
- One router is the **Master**, others are **Backups**.
- Default timers: **1-second advertisement interval**, with a **~3-second** master down interval (the time a backup waits without hearing advertisements before taking over).

## GLBP (Gateway Load Balancing Protocol)

- **Cisco-proprietary.** Provides redundancy *and* per-host load balancing across multiple gateways.
- **AVG (Active Virtual Gateway)**: Responds to ARP requests for the default gateway, replying with the virtual MAC address of one of the forwarders. This is how GLBP distributes hosts across routers.
- **AVF (Active Virtual Forwarder)**: A router currently forwarding traffic for its assigned virtual MAC address.
- Load-balancing methods:
  - **Round-robin** (default): The AVG hands out the forwarders' virtual MACs in rotation.
  - **Host-dependent**: A given host always gets the same forwarder, based on its MAC address.
  - **Weighted**: Traffic is distributed in proportion to each router's configured weight/capacity.

---

# SSO (Stateful Switchover)

**Stateful Switchover (SSO)** is a high-availability feature that lets a standby route processor take over the active processor's functions without losing state information. Ongoing connections and sessions are maintained during a failover, minimizing downtime.

- **Problem**: Failing over to a backup route processor can cause routing protocol neighbor relationships to be lost, resulting in a temporary loss of connectivity.
- **Secondary problem**: Packets might be dropped until the forwarding table is rebuilt on the standby processor.

**Solutions:**

- **NSF (Non-Stop Forwarding)**: Allows the router to continue forwarding packets even while the control plane recovers, working alongside SSO so routing protocols can rebuild gracefully without dropping traffic.
- **CEF (Cisco Express Forwarding)**: A high-performance forwarding mechanism using a pre-computed forwarding table (FIB), reducing per-packet processing overhead. NSF relies on CEF to keep forwarding packets during the switchover while routing protocol information is recovered.