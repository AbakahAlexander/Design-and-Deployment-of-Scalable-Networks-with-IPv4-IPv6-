
# Design and Deployment of Scalable Networks with IPv4/IPv6 and VLAN Segmentation

In this project, I created and deployed a multi-autonomous system (AS) network to showcase routing protocols, VLAN segmentation, IPv4/IPv6 addressing, link cost manipulation, and GRE tunneling. Everything was tested in GNS3 using Cisco router and switch images.

---

## Table of Contents
- [Overview](#overview)
- [Network Topology](#network-topology)
- [IPv4 Setup](#ipv4-setup)
  - [Router Interfaces and Naming](#router-interfaces-and-naming)
  - [IPv4 Addressing](#ipv4-addressing)
  - [OSPF (AS 4010)](#ospf-as-4010)
  - [RIP (AS 20001)](#rip-as-20001)
  - [BGP (iBGP and eBGP)](#bgp-ibgp-and-ebgp)
  - [VLAN Segmentation](#vlan-segmentation)
  - [Link Cost Preference](#link-cost-preference)
  - [Loopback Advertisement Restrictions](#loopback-advertisement-restrictions)
  - [Packet Capture](#packet-capture)
- [IPv6 Setup](#ipv6-setup)
  - [IPv6 Addressing](#ipv6-addressing)
  - [IPv6 OSPF Configuration](#ipv6-ospf-configuration)
  - [IPv6 BGP Configuration](#ipv6-bgp-configuration)
  - [Tunneling](#tunneling)



---

## Overview

I wanted to build a scalable, multi-AS network that demonstrates how to manage:
- IPv4 vs. IPv6 address assignment
- Routing protocols (OSPF, RIP, and BGP)
- VLAN segmentation on switches
- GRE tunneling between autonomous systems
- Traffic engineering by adjusting OSPF link costs
- Restricting which loopback interfaces are visible outside an AS

Through this process, I verified end-to-end connectivity, checked routing tables, examined protocol operation with Wireshark, and confirmed that each device could reach the routes learned via OSPF, RIP, and BGP as intended.

---

## Network Topology

I built the following topology in GNS3:

```
   (Customer A: AS 64496)          (Nextel: AS 20001)               (Telstar: AS 4010)       (Customer B: AS 64497)
              R4 ----------------- R1 -- R2 -- R3 ------------------ R7 -- R8 -- R9 ------------------ R5
                                                           |                  |
                                                           |               R10 (Level 3)
                                                           |
                                                    (Customer C: AS 64498)
                                                             R6
```

- **AS 20001** (R1, R2, R3) is connected to the three customers (AS 64496, AS 64497, and AS 64498).
- **AS 4010** (R7, R8, R9, R10) forms a separate domain with internal OSPF routing.
- A Level 3 router (R10) interconnects with R9.
- VLAN segmentation is used in AS 64497 and AS 64498 for local subnets.

---

## IPv4 Setup

### Router Interfaces and Naming

I labeled routers `R1` through `R10` and tied interface numbers to link speeds:
- FastEthernet (`f0/0`, `f1/0`) for 100 Mbps
- GigabitEthernet (`g1/0`, `g2/0`) for 1 Gbps

This naming scheme helped me keep track of which interfaces represented high-speed vs. lower-speed links.

### IPv4 Addressing

I assigned `/30` subnets to point-to-point links where only two IPs are needed and `/29` subnets on links requiring more available addresses (e.g., three or more routers sharing a segment). Loopbacks typically used `/32` masks. Below is an example of some assignments:

| Router/Interface | IP Address           | Subnet Mask            | Notes                          |
|------------------|----------------------|-------------------------|---------------------------------|
| R1 g1/0          | 192.168.1.1         | 255.255.255.252 (/30)  | Connection to Customer A (R4)   |
| R1 g2/0          | 192.168.0.1         | 255.255.255.252 (/30)  | Connection to R2               |
| R1 loopback0     | 192.168.2.1         | 255.255.255.255 (/32)  | Used for internal iBGP, etc.    |
| R2 g1/0          | 192.168.0.2         | 255.255.255.252 (/30)  | Connection to R1               |
| R3 f0/0          | 192.168.1.5         | 255.255.255.252 (/30)  | Connection to Customer B (R5)   |
| ...              | ...                  | ...                     | ...                             |

I confirmed correct assignments by checking `show ip interface brief` on each router.

### OSPF (AS 4010)

Within AS 4010, I chose OSPF to handle routing among R7, R8, R9, and R10. I activated OSPF on relevant interfaces in area 0 and verified neighbor adjacencies with commands like `show ip ospf neighbor`. This ensured the internal routes were properly exchanged and each router had consistent topology information.

### RIP (AS 20001)

For AS 20001, I used RIP to exchange internal routes among R1, R2, and R3. After enabling RIP (version 2), I checked the routing tables on each router to verify that subnets from R1, R2, and R3 were all learned. I also confirmed that external routes destined for the customers (AS 64496, 64497, 64498) were routed correctly.

### BGP (iBGP and eBGP)

I configured iBGP sessions among routers in the same AS (e.g., R1–R2–R3 in AS 20001 and R7–R8–R9–R10 in AS 4010). Meanwhile, I used eBGP to connect different ASes (for instance, R1 <-> R4, R3 <-> R5, R9 <-> R10). When configuring eBGP, I made sure each neighbor statement specified the correct remote AS, and I advertised only the loopback0 or loopback1 addresses I wanted to share. 

Notably, I kept `loopback0` routes private in eBGP advertisements, as required. This way, external peers only saw `loopback1` addresses from each AS.

### VLAN Segmentation

AS 64497 and AS 64498 each had two VLANs (VLAN 10 and VLAN 40 for AS 64497, VLAN 10 and VLAN 40 for AS 64498). On each site’s switch, I set two ports in **access** mode for the PCs, assigning each to the correct VLAN, and used **dot1q trunk** mode on the port leading to the router. 

The router interfaces in these ASes had subinterfaces (`f0/0.10`, `f0/0.40`, etc.) with corresponding VLAN tags. This let me confirm that PCs in the same VLAN could communicate properly, while remaining isolated from other VLANs.

### Link Cost Preference

Since GigabitEthernet links run at 1 Gbps and FastEthernet at 100 Mbps, I wanted OSPF to prefer the gigabit paths. To do this, I adjusted the OSPF reference bandwidth to `1000`, making the cost for GigabitEthernet links lower than FastEthernet. This forced OSPF to route traffic through the higher-capacity links when possible.

### Loopback Advertisement Restrictions

I also ensured that `loopback0` was **not** advertised to external neighbors. For eBGP, I only included `loopback1` in the `network` statements. On the other hand, the iBGP or internal IGP configurations still carried `loopback0` so I could use it for internal router identification and stability checks.

### Packet Capture

To verify everything was working as expected, I ran Wireshark captures on the GNS3 links. I confirmed that:
- OSPF Hello packets and LSAs were exchanged within each OSPF domain.
- RIP updates circulated in AS 20001.
- BGP KEEPALIVE and UPDATE messages were flowing between peers.

This gave me visibility into the protocols’ behavior and helped confirm that adjacency formation was correct.

---

## IPv6 Setup

### IPv6 Addressing

After establishing IPv4 routing, I moved on to IPv6. I used `/66` and `/67` prefixes for router-to-router links and `/128` for loopbacks. An example of some assigned addresses:

| Router/Interface     | IPv6 Address                         | Prefix  | Notes                                     |
|----------------------|--------------------------------------|--------|-------------------------------------------|
| R5 g2/0 (AS 64497)   | fd00:0:0:2000:0002::0/66             | /66    | VLAN/Customer link for IPv6              |
| R6 f0/0 (AS 64498)   | fd00:0:0:2000:4002::0/66             | /66    | Another VLAN link                         |
| R9 g2/0 (AS 4010)    | fd00:0:0:1000:8001::0/66             | /66    | Internal Gbps link inside Telstar         |
| R10 g2/0             | fd00:0:0:3000:0002::0/67             | /67    | Connection from Telstar to Level 3        |
| Loopback0 (Various)  | fd00:0:0:4000:xxxx::/128             | /128   | iBGP endpoints, internal addresses, etc.  |

### IPv6 OSPF Configuration

Within Telstar (AS 4010), I enabled OSPFv3 (or IPv6 OSPF) by activating IPv6 unicast routing on each router, then placing the relevant router interfaces in OSPF area 0. The neighbor relationships came up similarly to the IPv4 OSPF, which let me confirm that both protocols coexisted smoothly.

### IPv6 BGP Configuration

I also set up BGP for IPv6 routes by specifying `address-family ipv6` in the `router bgp <AS>` configuration. I then activated each neighbor for IPv6. Once done, each router in the same AS formed iBGP relationships, and eBGP adjacency was established with external AS neighbors. Checking `show bgp ipv6 unicast` gave me all the prefixes being exchanged.

### Tunneling

I tested a GRE tunnel carrying IPv4 traffic over an IPv6 path (and vice versa) between two customer routers (e.g., R5 and R6). I created the `tunnel` interfaces, assigned appropriate IPv4 addresses, and pointed static routes at the tunnel for remote subnets. This allowed traffic from one customer VLAN to reach the other’s VLAN, securely tunneled across IPv6.

---


- **Platform**: I ran GNS3 version X.X with Cisco IOS images that support IPv6 routing and advanced protocols.
- **Requirements**: Sufficient computing resources for multiple routers/switches in GNS3, and Wireshark for captures.
- **Configuration Files**: If you have my exported `.gns3` project or individual `startup-config`s, you can import them directly. Otherwise, you can replicate the steps I took as described above.

