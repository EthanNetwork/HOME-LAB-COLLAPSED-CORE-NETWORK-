# Enterprise LAN Lab: Cisco Home Lab Configuration

## Overview

This repository contains the running configurations for a home lab built to simulate a scaled-down enterprise LAN. The lab demonstrates routing redundancy, Layer 2 security hardening, management plane segmentation, and OSPF with MD5 authentication across a five-device topology.

All device configurations are provided as exported running-config text files.


##DIAGRAM

<img width="887" height="587" alt="image" src="https://github.com/user-attachments/assets/7e1f6c95-cd8a-4371-bc98-584726649eb2" />


---

## Lab Devices

| Device | Model | Role |
|---|---|---|
TopRouter | Cisco ISR 4331 | WAN Edge Router VLANs 10/20) |
BottomRouter | Cisco ISR 4331 | WAN Edge Router VLANs 30/40) |
CORESW00 | Cisco Catalyst 3650 | Layer 3 Core Switch |
TopSwitch | Cisco Catalyst 2960X-24PS-L | Access Layer Switch |
BottomSwitch | Cisco Catalyst 2960X-48LPS-L | Access Layer Switch |



## VLAN Design



10: 10.10.10.0/24 (Data),
20: 172.168.20.0/24 (Data),
30: 192.168.30.0/24 (Data),
40 : 192.168.40.0/24 (Data),
99 : 10.99.99.0/24 (OOB Management),
1 : Native VLAN

 VLAN 99 is the dedicated OOB management plane, isolated from production data traffic using a separate EIGRP routing protocol not shared by any other devices.

---

## WAN Edge: HSRP Active/Active Split

The two ISR 4331 routers implement HSRP version 2 in an active/active split configuration across the four data VLANs. HSRP active roles are split between routers to distribute traffic rather than running a simple active/standby pair:

- TopRouter: HSRP active for VLANs 10 and 20 (priority 255), with BottomRouter as standby (priority 254)
- BottomRouter: HSRP active for VLANs 30 and 40 (priority 255), with TopRouter as standby (priority 254)

Both routers have preempt configured, ensuring the designated active router reclaims its role after recovery from a failure. DHCP pools for each VLAN are hosted on their respective active router, with the HSRP virtual IP as the default gateway. The DHCP pool of addresses is split between the two routers, ensuring that in the instance of device failure, only half the DHCP assignment space will go down instead of taking down the entire network.

---

## Routing: OSPF with MD5 Authentication

OSPF process 1 runs across all routed devices with the following design decisions:

- Router IDs are assigned via Loopback0 interfaces (10.1.1.1 on TopRouter, 10.1.1.2 on BottomRouter), and enforced by router-id manual setting, providing stable OSPF adjacencies independent of physical interface state
- MD5 authentication is enforced on area 0 across all participating interfaces, preventing unauthorized routers from forming adjacencies
- Reference bandwidth is set to 100,000 Mbps on all devices to normalize cost calculations across the topology
- Area 1 is configured with MD5 authentication as a placeholder for future topology expansion: no interfaces are currently assigned to area 1. 
- CORESW00 participates in OSPF via its SVI interfaces for each data VLAN

---

## Management Plane Segmentation: EIGRP on OOB VLAN

VLAN 99 (10.99.99.0/24) is the dedicated OOB management subnet. Rather than including it in the OSPF domain, EIGRP process 1 is used exclusively for management plane reachability across all devices.

This is an intentional routing plane segmentation decision. A device on any data VLAN has no ability to influence OSPF routing toward the management subnet, and a compromised host on a production VLAN has no visibility into how management addresses are reached. OSPF and EIGRP are not redistributed between each other, keeping the two routing domains fully isolated. Beside security concerns, this organization has the added benefit of creating cleaner routing tables, especially as the topology expands. 

---

## Core Switch: CORESW00 (Catalyst 3650)

CORESW00 is the core switch with IP routing enabled and SVI interfaces for all VLANs. It connects to both access switches via EtherChannel (Port-channel1 and Port-channel2) and to both routers via individual trunk links. Spanning tree priorities are set to ensure CORESW00 is the root bridge for all VLANs.

CORESW00 also has ip routing enabled and all SVIs participate in OSPF to enable faster LAN local routing rather than having to have all routing go to the network edge and come back needlessly. 

---

## Access Layer Security: TopSwitch and BottomSwitch

Both 2960X access switches implement a full Layer 2 security stack consistently across all access ports.

DHCP Snooping: Enabled on VLANs 10, 20, 30, and 40. Only uplink and EtherChannel ports are trusted. Access ports have DHCP rate limiting applied to prevent starvation attacks. The information option (Option 82) is disabled to avoid conflicts with the upstream DHCP servers on the routers, as option 82 is a known source of conflict when it adds additional header details mapping port topology data which gets flagged as alteration by DHCP snooping. 

Dynamic ARP Inspection (DAI): Enabled on VLANs 10, 20, 30, and 40 with source MAC and IP validation. DAI uses the DHCP snooping binding table to validate ARP packets (validation for src-mac and ip enabled), blocking ARP poisoning and on path attacks. ARP inspection rate limiting is applied on all access ports.

Port Security: All access ports are configured with a maximum of 10 MAC addresses, violation mode restrict (drops frames and logs without disabling the port), and inactivity-based aging at 30 minutes.

Rapid PVST Spanning Tree is configured on both switches. All access ports have PortFast edge and BPDU Guard enabled, preventing rogue switches from being introduced on end-user ports. Loop Guard is enabled globally. Errdisable recovery is configured for all relevant causes at a 30-second interval. BPDU filter is enabled globally for portfast interfaces and BPDU Guard is enabled on the interface level due to those features coexisting on the same configuration level canceling each other out (BPDU filter directs the interface not to send or check for BPDU and BPDU guard relies on BPDU checks), however the construction of the CISCO IOS allows for them to exist on separate config levels where at an interface level Guard performs the first check and at the global level filter directs BPDUs to not be sent out, and since global exists higher than the interface level, the direction to not check BPDU does not apply to the interface level Guard. 

ACL Design: Two separate ACLs serve distinct purposes on the access switches:

- denyssh(standard ACL, applied inbound on access ports of access layer switches): Blocks traffic sourced from the management subnet (10.99.99.0/24) from entering the network on end user ports. This prevents a rogue device from spoofing a management address on an access port, which lacking a DHCP server issuing those IPs should be blocked by DHCP snoop but defense in depth principles applied here; the ACL is an extra backstop. 
- sshvty/sshvty99 (extended ACL, applied to VTY lines of all devices): Restricts SSH management access to network devices to only hosts originating from the management subnet. This controls who can reach the management plane.


---

## Out of Band Access

All devices restrict VTY access to SSH version 2 only. RSA crypto keys are used for ssh (modulus 2048), VTY lines on all devices have access-class applied, limiting remote management sessions to hosts within the 10.99.99.0/24 management subnet. Console lines have logging synchronous configured to prevent log output from interrupting active CLI sessions, a QOL feature. 

A Raspberry Pi console server is planned as a jump box for true out-of-band access via VLAN 99 subinterfaces on the routers. Currently SSH is possible from any router or switch to any other router or switch via console cable, local login, and then SSH to the other device. 

---

## NTP

All devices synchronize to 1.1.1.1 (Cloudflare) as the NTP server.

## Future Expansion

The core design philosophy is collapsed core, with a dual home WAN edge that upon rack expansion (planned Juniper EX4300 layer three switch) will also feature planned PAT pool translation to translate LAN IPs into a small SP provided public address pool with ephemeral port no.s assigned to each translation, allowing device allocation in shared public IPs. This will be accomplished in later configs by using a standard ACL to define what private IP space should be translated, setting a ip nat pool starting ip to end ip mask and setting overload on the ACL pooled translation. 



