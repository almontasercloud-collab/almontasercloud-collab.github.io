---
title: LACP NIC Teaming to Cisco Nexus vPC
date: 2026-07-13 16:25:00 +0300
categories: [Labs, Switching]
tags: [networking, Network Engineering, Computer Networks, Switching, NXOS, Nexus, vPC]
image:
  path: /assets/img/post_covers/nic_teaming_vpc.png
pin: false
---


# Introduction

This lab demonstrates how to deploy a resilient server connection using Cisco Nexus vPC and Windows Server NIC Teaming with LACP. The objective is to provide link and device redundancy while maintaining active-active connectivity between the server and the network. 

## Technology Overview
### Link Aggregation Control Protocol (LACP) in Windows Server

Windows Server supports the IEEE `802.3ad` Link Aggregation Control Protocol (LACP) to dynamically establish and maintain link aggregation groups by exchanging Link Aggregation Control Protocol Data Units (LACPDUs) with connected switches. When the Teaming Mode is configured for LACP, the server and network devices continuously negotiate bundle membership and monitor link health, automatically detecting cabling issues, configuration mismatches, or failed links that static teaming cannot identify. 

### Cisco Nexus Virtual Port Channel (vPC) Concepts

Virtual Port Channel (vPC) is a Cisco NX-OS feature that enables two independent Cisco Nexus switches to appear as a single logical Link Aggregation Group (LAG) endpoint to downstream devices. While each switch maintains an independent control plane, vPC synchronizes the information required for consistent forwarding, allowing both switches to actively forward traffic simultaneously. This eliminates the need for Spanning Tree Protocol (STP) to block redundant host-facing links, maximizing available bandwidth and providing deterministic redundancy. The technology relies on two dedicated mechanisms—the vPC Peer-Link and Peer-Keepalive—to synchronize critical state information, including MAC address tables, IGMP snooping state, and LACP system information, while continuously monitoring peer health.

Architecturally, vPC is Cisco's implementation of Multi-Chassis Link Aggregation (M-LAG). Like Huawei's M-LAG and Alcatel-Lucent Enterprise's MC-LAG, it allows a downstream device to form a single LAG across two independent switches, providing active-active forwarding and chassis-level redundancy without requiring a traditional switch stack or a single shared control plane.

## Topology & Prerequisites
### Network Diagram & EVE-NG Setup

In this lab environment, EVE-NG is deployed as a virtual machine hosted on a VMware ESXi hypervisor. All configuration tasks, verification procedures, and failover validation exercises presented throughout this article are performed using the topology illustrated below.

![CMD as Administrator](/assets/img/posts_photos/vPC/lab_topo.png)


This baseline topology uses the dedicated `Mgmt0` interface for the out-of-band vPC Peer-Keepalive link, ensuring that peer liveness detection remains independent of the data network. The vPC Peer-Link is formed as a three-member port channel spanning Eth1/1 through Eth1/3 on both Nexus switches, providing resilient state synchronization between the vPC peers. Downstream host connectivity is established via Eth1/4 on each switch, where the links terminate as a single active-active LACP port channel (Team-1) across the Windows Server's e0 and e1 network interfaces.

### Hardware & Software Requirements
#### Images: 
[**EVE Community Edition**](https://www.eve-ng.net/index.php/download/)

[**Windows Server 2019**](https://legacy.labhub.eu.org/0:/addons/qemu/Windows/winserver-2019/)

[**Nexus 9k**](https://legacy.labhub.eu.org/0:/addons/qemu/Cisco%20Nexus%209000v%20switch/)

> EVE-NG can be deployed on several supported virtualization platforms, including VMware ESXi, VMware Workstation and VirtualBox. Refer to the [**official documentation**](https://www.eve-ng.net/index.php/documentation/installation/virtual-machine-install/) for installation instructions. 


## Phase 1: Base Nexus Switch Configuration
### Enabling Management & Core Features (lacp, vpc)

>The complete configurations for both Nexus switches are available in the [**GitHub repository**](https://github.com/almontasercloud-collab/lab-nexus-vpc-nic-teaming).
{: .prompt-tip }

>This lab used a minimal, single-default-VLAN configuration to keep the focus entirely on the core behaviors of vPC and NIC Teaming.
{: .prompt-warning }

NX-OS utilizes a modular architecture where most protocol engines are disabled by default to optimize CPU and memory allocation. To build this topology, you must explicitly enable feature `lacp` to process Link Aggregation Control Protocol frames, and feature `vpc` to instantiate the multi-chassis synchronization control plane.

**These features must be enabled on both Nexus-1 and Nexus-2.** 

```bash
configure terminal
feature lacp
feature vpc
```
### Configuring the vPC Peer-Keepalive (PKAL) Link

The **vPC Peer-Keepalive (PKAL) link** is a dedicated out-of-band Layer 3 control path used exclusively for peer liveness detection and split-brain prevention. It exchanges periodic heartbeat messages over UDP port 3200, allowing each Nexus switch to verify the availability of its peer independently of the vPC Peer-Link.

The following IP addressing scheme is assigned to the Mgmt0 interfaces and will be used for the vPC Peer-Keepalive connection:

**Nexus-1:**
```bash
interface mgmt0
ip address 192.168.100.1/24
no shutdown
```
**Nexus-2:**
```bash
interface mgmt0
ip address 192.168.100.2/24
no shutdown
```
### Defining the vPC Domain

A **vPC domain** is a logical administrative grouping that pairs two distinct physical Nexus switches into a single virtualized chassis layer sharing a unique Domain ID.

The vPC role determines the operational responsibility of each Nexus peer, where the switch with the lower vPC role priority value is elected as the **Primary**, while the other becomes the **Secondary** peer.
The **Primary** vPC peer provides role coordination, performs consistency decision handling, and maintains synchronization with the Secondary through the vPC peer-link.

**Nexus-1:**
```bash
! Create the vPC domain. The domain ID must match on both Nexus peers.
!
vpc domain 1
!
! Configure the vPC Peer-Keepalive over the dedicated management network using the management VRF.
!
peer-keepalive destination 192.168.100.2 source 192.168.100.1 vrf management
!
! Set Nexus-1 as secondary peer (Nexus-2 priority will be set to 100 and become primary)
!
role priority 200
!
!
! Allow either vPC peer to forward traffic destined for the virtual gateway MAC, improving forwarding efficiency.
!
peer-gateway
!
!Automatically restores vPC member ports after a complete dual-switch outage if the peer remains unavailable.
!
auto-recovery
!
```

**Nexus-2:**
```bash
vpc domain 1
peer-keepalive destination 192.168.100.1 source 192.168.100.2 vrf management
role priority 100
peer-gateway
auto-recovery
```

### Configuring the vPC Peer-Link

The vPC Peer-Link is a high-bandwidth Layer 2 trunk port channel that serves as the primary synchronization path between the two vPC peers. It synchronizes critical control-plane state, including MAC address tables and IGMP snooping information, while also forwarding specific data-plane traffic, such as traffic destined for orphan ports or during certain failure scenarios.

> Configuration Note: The interface bundling and peer-link configurations must be identical on both Nexus-1 and Nexus-2. The configuration below creates the underlying LACP port-channel across interfaces Eth1/1 to Eth1/3 and binds it as the vPC peer-link backbone.

**Peer-Link Configuration:**
```bash
interface port-channel1
  description vPC Peer-Link
  switchport
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link
  no shutdown

interface ethernet1/1
  description vPC Peer-Link
  switchport
  switchport mode trunk
  channel-group 1 mode active
  no shutdown

interface ethernet1/2
  description vPC Peer-Link
  switchport
  switchport mode trunk
  channel-group 1 mode active
  no shutdown

interface ethernet1/3
  description vPC Peer-Link
  switchport
  switchport mode trunk
  channel-group 1 mode active
  no shutdown
```
### Verifying Peer-Link and Keepalive Status

Once the configuration is applied to both switches, you must execute validation commands to confirm that the peer-keepalive heartbeat is active and the peer-link has successfully initialized the vPC control plane.

**vPC Verification Commands:**
``` bash
show vpc
show vpc role
show port-channel summary
```

## Phase 2: Downstream vPC Configuration
### Creating the Host-Facing vPC Port-Channel

With the backbone established, you must now build the logical multi-chassis port-channel interfaces that terminate downstream directly on the Windows Server.

**Nexus-1:**
``` bash
interface port-channel10
 description ### Lab_Winserver_Portchannel ###
 switchport
 switchport mode access
 vpc 10
 no shutdown

interface ethernet1/4
  description ### Nexus_1_winserver_link ###
  switchport
  channel-group 10 mode active
  no shutdown
```
**Nexus-2:**
``` bash
interface port-channel10
 description ### Lab_Winserver_Portchannel ###
 switchport
 switchport mode access
 vpc 10
 no shutdown

interface ethernet1/4
  description ### Nexus_2_winserver_link ###
  switchport
  channel-group 10 mode active
  no shutdown
```

> At this stage the host-facing member interface (Eth1/4) is currently suspended (in both nexus switches) because it has not yet received LACP PDUs from the downstream Windows Server. Consequently, the logical host port-channel (Po10) remains down, though it has successfully passed all vPC configuration consistency checks and is completely ready to forward traffic once the server-side NIC teaming active negotiation begins.

**Nexus-1 Showing eth1/4 down state:**
``` bash

vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
10    Po10          down*  success     success               -

Ethernet        VLAN    Type Mode   Status  Reason                 Speed     Port
Interface                                                                    Ch
#
--------------------------------------------------------------------------------
Eth1/1          1       eth  trunk  up      none                     1000(D) 1
Eth1/2          1       eth  trunk  up      none                     1000(D) 1
Eth1/3          1       eth  trunk  up      none                     1000(D) 1
Eth1/4          1       eth  access down    suspended(no LACP PDUs)  auto(D) 10


```

## Phase 3: Windows Server NIC Teaming Configuration
### Accessing Server Manager

Open **Server Manager**, then navigate to Local Server to access the NIC Teaming configuration.

![CMD as Administrator](/assets/img/posts_photos/vPC/server_manager.png)

### Configuring NIC Teaming Settings

Click the underlined Enabled link next to NIC Teaming in Server Manager to open the NIC Teaming management window.

![CMD as Administrator](/assets/img/posts_photos/vPC/NIC_teaming_window.png)

The Adapters and Interfaces section lists all available physical network adapters, which in this lab are **Ethernet** and **Ethernet 2**. The remaining sections display any existing NIC teams and their associated network interfaces.

From the TASKS menu, select New Team... to create a new NIC team.

![CMD as Administrator](/assets/img/posts_photos/vPC/new_team.png)

Create a new NIC team named **Team-1**, add both Ethernet and Ethernet 2 as member adapters, then configure the teaming mode as LACP with Dynamic load balancing, as shown below.

![CMD as Administrator](/assets/img/posts_photos/vPC/NIC_teaming_configuration.png)

The NIC team initially entered a Fault state because LACP negotiation had not yet reached a synchronized state between the Windows Server and the Nexus vPC port-channel.

![CMD as Administrator](/assets/img/posts_photos/vPC/initial_NIC_team_state.png)

After exchanging LACP negotiation packets with the Nexus vPC port-channel, the Windows NIC Teaming service successfully synchronized both member adapters. The team transitioned from Fault to an active state once the LACP session became established.

![CMD as Administrator](/assets/img/posts_photos/vPC/team-1_after_negotiation.png)

## Verification & Validation
### Nexus Side Verification

The following verification output confirms that the vPC peer-link and vPC member port-channels are successfully established and operating normally.
The peer-link (Po1) is up, and the vPC port-channel (Po10) shows up status with successful consistency checks, indicating proper synchronization between both Nexus peers.

**Nexus-1:**

``` bash
vPC Peer-link status
---------------------------------------------------------------------
id    Port   Status Active vlans
--    ----   ------ -------------------------------------------------
1     Po1    up     1


vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
10    Po10          up     success     success               1

--------------------------------------------------------------------------------
Group Port-       Type     Protocol  Member Ports
      Channel
--------------------------------------------------------------------------------
1     Po1(SU)     Eth      LACP      Eth1/1(P)    Eth1/2(P)    Eth1/3(P)
10    Po10(SU)    Eth      LACP      Eth1/4(P)
```


### Windows Side Verification

Windows verification confirms successful NIC Team operation, with ipconfig showing a valid DHCP-assigned IP address from the lab network.

![CMD as Administrator](/assets/img/posts_photos/vPC/windows_success.png)

Gateway reachability is confirmed through successful ping tests.

![CMD as Administrator](/assets/img/posts_photos/vPC/win_ping.png)

## Failure Scenario & Redundancy Testing
### Scenario A: Simulating a Single Link Failure (Host to NXOS-1)

To simulate a single link failure, shut down interface Ethernet 1/4 on Nexus-1 and observe the traffic failover behavior.

![CMD as Administrator](/assets/img/posts_photos/vPC/link_failure_test.png)

Shutting down the interface causes no packet loss, but bringing it back up (no shutdown) triggers a brief drop. This happens during the LACP convergence window, where traffic is hashed over the restored link before hardware forwarding tables fully synchronize.

### Scenario B: Simulating a Switch Failure (Powering off NXOS-1)

To simulate a complete node failure, power off Nexus-1 to observe how the remaining vPC peer handles the workload.

Powering off the active switch causes a brief drop as the host's NIC team registers the dead link and the remaining vPC peer updates its forwarding tables. 

![CMD as Administrator](/assets/img/posts_photos/vPC/switch_failure_test.png)

Traffic quickly recovers and stabilizes over the single active link once the Windows Server NIC team marks the failed interface as faulted.

![CMD as Administrator](/assets/img/posts_photos/vPC/switch_failure_test_windows.png)

### Scenario C: vPC Peer-Link Failure (Dual-Active/Split-Brain Prevention)

When the primary vPC Peer-Link ruptures while the Peer-Keepalive remains up, the vPC domain invokes an active loop-prevention sequence. The operational secondary peer (**Nexus-1 in this case**) instantly disables its local host-facing vPC member ports and any vPC VLAN SVIs to isolate itself from the data plane. The vPC primary peer (**Nexus-2 in this case**) continues to forward all upstream and downstream traffic, maintaining deterministic traffic symmetry and preventing corrupted MAC address table sync across the network topology.

Verify the baseline environment using show vpc brief. Both switches should display a healthy peer adjacency, an operational peer-link (Po1), and active vPC member ports (Po10) split across the primary and secondary nodes.

![CMD as Administrator](/assets/img/posts_photos/vPC/before_peer_link_failure.png)

To simulate a pure peer-link failure, shut down Port-Channel 1 on Nexus-1. This breaks the primary data and control path between the switches while leaving the separate peer keep-alive link intact.

![CMD as Administrator](/assets/img/posts_photos/vPC/shutting_peer_link.png)

Because the keep-alive link confirms both switches are still running, dual-active prevention triggers. The secondary switch (Nexus-1) immediately suspends its vPC member ports to prevent a split-brain condition, while the primary switch (Nexus-2) keeps its ports up to handle all traffic.

![CMD as Administrator](/assets/img/posts_photos/vPC/nexus_split_brain.png)

## conclusion  

This lab demonstrated that integrating Windows Server NIC Teaming with Cisco Nexus vPC provides a resilient LACP-based high-availability solution. By testing link, switch, and peer-link failure scenarios, the design successfully maintained connectivity through automatic failover, synchronization, and split-brain prevention mechanisms. This architecture eliminates single points of failure at the server access layer and ensures continuous connectivity for critical infrastructure workloads.

