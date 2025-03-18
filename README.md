# Design and Deployment of Scalable Networks with IPv4/IPv6 and VLAN Segmentation

This project demonstrates how to design and deploy a multi-autonomous system (AS) network using both IPv4 and IPv6. It covers key topics such as VLAN configuration, OSPF routing, RIP routing, BGP (both iBGP and eBGP), link cost manipulation for preferred paths, and tunneling. The setup was tested and visualized in GNS3.

---

## Table of Contents
- [Overview](#overview)
- [Network Topology](#network-topology)
- [IPv4 Configuration](#ipv4-configuration)
  - [Router Interfaces and Naming](#router-interfaces-and-naming)
  - [IP Addressing Plan](#ip-addressing-plan)
  - [OSPF Configuration (AS 4010)](#ospf-configuration-as-4010)
  - [RIP Configuration (AS 20001)](#rip-configuration-as-20001)
  - [BGP Configuration (iBGP and eBGP)](#bgp-configuration-ibgp-and-ebgp)
  - [VLAN Configuration](#vlan-configuration)
  - [Link Cost Preferences](#link-cost-preferences)
  - [Restricting Loopback Advertisement](#restricting-loopback-advertisement)
  - [Packet Capture Example](#packet-capture-example)
- [IPv6 Configuration](#ipv6-configuration)
  - [IPv6 Addressing Plan](#ipv6-addressing-plan)
  - [IPv6 OSPF Configuration](#ipv6-ospf-configuration)
  - [IPv6 BGP Configuration](#ipv6-bgp-configuration)
  - [Tunneling](#tunneling)
- [How to Reproduce](#how-to-reproduce)
- [Contact](#contact)

---

## Overview

In this project, multiple autonomous systems (AS) are interconnected to form a scalable network supporting both IPv4 and IPv6. The following major topics are included:

- **IPv4 & IPv6**: Fundamental addressing and routing
- **Routing Protocols**: OSPF, RIP, and BGP (both iBGP and eBGP)
- **VLAN Segmentation**: Isolating networks using VLANs on switches
- **GRE Tunnels**: Tunneling IPv4 over IPv6 (or vice versa) connections
- **Link Cost Manipulation**: Adjusting OSPF metrics to prefer certain link speeds
- **Loopback Address Management**: Controlling which loopback interfaces are advertised

---

## Network Topology

Below is a simplified representation of the overall topology (as configured in GNS3). Each router is labeled (R1, R2, R3, etc.) and belongs to a specific AS:

```
( Customer A: AS 64496 )      (AS 20001: Nextel)                 (AS 4010: Telstar)     ( Customer B: AS 64497 )
       R4 ------------------ R1 -- R2 -- R3 ------------------ R7 -- R8 -- R9 ------------------ R5
                                                     |                    |
                                                     |                R10 (Level 3)
                                                     |
                                                   ( Customer C: AS 64498 )
                                                     R6
```

> *Note: Replace this ASCII diagram with your actual GNS3 screenshot or network diagram as needed.*

---

## IPv4 Configuration

### Router Interfaces and Naming

- Each router is given names from `R1` to `R10`.
- Interface numbering correlates with link type: 
  - FastEthernet (`f0/0`, `f1/0`, etc.) is used for 100 Mbps links.
  - GigabitEthernet (`g1/0`, `g2/0`, etc.) is used for 1 Gbps links.

### IP Addressing Plan

- **/30 subnets** are used for point-to-point links that only need two usable addresses.
- **/29 subnets** are used on links that require up to 6 usable addresses (e.g., router-to-router links with more interfaces).
- **Loopback** addresses typically use a /32 mask in IPv4.

Below is a sample of the address allocation (partial excerpt):

| Router/Interface    | IP Address        | Subnet Mask         | Notes                      |
|---------------------|-------------------|----------------------|----------------------------|
| **R1 g1/0**         | 192.168.1.1      | 255.255.255.252 (/30)| Towards Customer A (R4)    |
| **R1 g2/0**         | 192.168.0.1      | 255.255.255.252 (/30)| Core link to R2           |
| **R1 loopback0**    | 192.168.2.1      | 255.255.255.255 (/32)| Loopback for iBGP, etc.    |
| **R2 g1/0**         | 192.168.0.2      | 255.255.255.252 (/30)| Connected to R1           |
| ...                 | ...               | ...                  | ...                        |

> **Tip:** Use the CLI commands like:
> ```
> conf t
> interface g1/0
>  ip address 192.168.1.1 255.255.255.252
> end
> wr
> ```

### OSPF Configuration (AS 4010)

AS 4010 runs **OSPF** on its internal routers (`R7`, `R8`, `R9`, and possibly `R10` if it belongs to the same AS for IPv4).

1. Enter configuration mode:
   ```bash
   conf t
   router ospf 1
   network <subnet_id> <wildcard_mask> area 0
   end
   wr
   ```
2. Verify with `show ip protocols` or `show ip ospf neighbor`.

### RIP Configuration (AS 20001)

AS 20001 (e.g., `R1`, `R2`, `R3`) runs **RIP** to exchange routing information. Sample config:
```bash
conf t
router rip
 network 192.168.0.0
 network 192.168.1.0
 version 2
end
wr
```
Check with `show ip route rip`.

### BGP Configuration (iBGP and eBGP)

- **iBGP** is configured between routers within the same AS.
- **eBGP** is configured for connections between routers in different ASes.
- **Note:** The assignment specifies that `loopback1` addresses should **not** be advertised to external neighbors.

Sample configuration snippet (on a router in AS 20001):
```bash
conf t
router bgp 20001
  neighbor 192.168.0.2 remote-as 20001     ! iBGP
  neighbor 192.168.1.2 remote-as 64496     ! eBGP (customer link)
  network 192.168.2.0 mask 255.255.255.0   ! Advertise loopback0 only inside the AS
  ! do not advertise loopback1 over eBGP
end
```

### VLAN Configuration

Two switches (in AS 64497 and AS 64498) separate VLANs to isolate traffic among hosts. Each switch port connecting a PC is set as an access port in the appropriate VLAN (e.g., VLAN 10 or VLAN 40).

Example for creating/accessing VLAN 10 and VLAN 40 on a switch:
```bash
conf t
vlan 10
 name VLAN10
vlan 40
 name VLAN40
!
interface f0/1
 switchport mode access
 switchport access vlan 10
!
interface f0/2
 switchport mode access
 switchport access vlan 40
!
interface f0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

On the router (`R5`, for example), subinterfaces are configured for each VLAN:
```bash
interface f0/0.10
 encapsulation dot1q 10
 ip address 10.0.10.1 255.255.255.0

interface f0/0.40
 encapsulation dot1q 40
 ip address 10.0.40.1 255.255.255.0
```

### Link Cost Preferences

To prefer **GigabitEthernet** links over **FastEthernet** in OSPF, adjust the reference bandwidth:
```bash
conf t
router ospf 1
 auto-cost reference-bandwidth 1000
end
```
This ensures Gigabit links have lower cost (better preference) compared to FastEthernet.

### Restricting Loopback Advertisement

Per the assignment, `loopback0` addresses must **not** be advertised to external neighbors:
- Simply omit `loopback0` from the `network` statements in eBGP neighbor configurations.
- Only add `loopback1` if you need a loopback interface externally visible.

### Packet Capture Example

Below is a snippet from a Wireshark capture of OSPF hello/LSA exchanges between `R7` and `R8`:

```
OSPFv2 Hello Packet
    Version: 2
    Message Type: Hello
    ...
```

---

## IPv6 Configuration

### IPv6 Addressing Plan

For the same ASes, the IPv6 addresses are assigned with suitable prefixes (e.g., /66, /67, /128 for loopbacks). A sample of the IPv6 address plan:

| Router/Interface       | IPv6 Address                         | Prefix  | Notes                              |
|------------------------|--------------------------------------|--------|-------------------------------------|
| **R5 g2/0** (AS 64497) | fd00:0:0:2000:0002::0/66             | /66    | VLAN/Customer link                  |
| **R6 f0/0** (AS 64498) | fd00:0:0:2000:4002::0/66             | /66    | VLAN/Customer link                  |
| **R9 g2/0** (AS 4010)  | fd00:0:0:1000:8001::0/66             | /66    | Internal Gbps link in Telstar AS    |
| **R10 g2/0**           | fd00:0:0:3000:0002::0/67             | /67    | Level 3 link                        |
| **Loopback0**          | fd00:0:0:4000:0001::0/128            | /128   | Used for iBGP in IPv6, etc.         |

### IPv6 OSPF Configuration

On each router within AS 4010 (Telstar), enable OSPFv3 (or IPv6 OSPF) for interfaces:
```bash
conf t
ipv6 unicast-routing
ipv6 cef
interface g1/0
 ipv6 address fd00:0:0:1000:0001::1/66
 ipv6 ospf 1 area 0
```
Check with `show ipv6 ospf neighbor` or `show ipv6 route ospf`.

### IPv6 BGP Configuration

Similarly, configure BGP for IPv6:
```bash
conf t
ipv6 unicast-routing
router bgp 4010
 bgp router-id 7.7.7.7
 neighbor fd00:0:0:1000:0002::1 remote-as 4010
 address-family ipv6
  neighbor fd00:0:0:1000:0002::1 activate
  network fd00:0:0:4000:0001::/128
 exit
end
```
- Use `address-family ipv6` to specify IPv6 routes.
- iBGP vs. eBGP is determined by whether the `remote-as` matches your own AS.

### Tunneling

To enable GRE IPv4-over-IPv6 (or IPv6-over-IPv4) tunnels, configure **tunnel interfaces** on each endpoint:
```bash
conf t
interface Tunnel1
 no shutdown
 ip address 192.168.1.129 255.255.255.252
 tunnel source <ipv6 or ipv4 interface/address>
 tunnel destination <remote ipv6 or ipv4 interface/address>
 tunnel mode gre ipv6   ! or 'tunnel mode gre ip'
exit

ip route 10.0.10.0 255.255.255.0 Tunnel1
```
This allows hosts in different ASes (e.g., Customer B & C) to reach each other via the tunnel.

---

## How to Reproduce

1. **Software Requirements**  
   - [GNS3](https://www.gns3.com/) or a similar network emulator  
   - Cisco router/switch images (IOS) compatible with your GNS3 version  

2. **Clone/Download Project**  
   - Clone this repository to get the base configuration files if you’ve exported them.

3. **Import / Re-create Topology**  
   - Open GNS3 and either import the provided `.gns3project` file or manually recreate the topology:
     - Create routers R1 through R10.
     - Assign interfaces per the address plan.
     - Configure OSPF, RIP, BGP, VLAN, etc.

4. **Configure the Devices**  
   - Use the sample configuration commands from the sections above.
   - Adjust the IP addresses, VLAN IDs, and AS numbers to match your local environment.

5. **Testing**  
   - Verify connectivity with `ping`, `traceroute`, `show ip route`, and `show ip bgp`.
   - Capture traffic with Wireshark to confirm routing protocol exchanges.

---

## Contact

For any questions or clarifications, feel free to reach out:

- **Email:** alexanderabakah70@gmail.com

---

> **Disclaimer**: This README and sample configuration are for demonstration and learning purposes. Always follow your organizational policies and best practices when configuring production networks.

