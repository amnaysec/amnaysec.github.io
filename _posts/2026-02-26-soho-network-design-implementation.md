---
title: "My SOHO Network Design: VLANs, ROAS, and DHCP Implementation Journey"
date: 2026-02-26 09:00:00 +0000
categories: [Networking, Security]
tags: [cisco, vlan, routing, dhcp, soho, packet-tracer, troubleshooting]
description: A personal technical write-up on designing and implementing a Small Office Home Office (SOHO) network, detailing my approach to Inter-VLAN routing, secure wireless access, and the challenges I overcame.
---

## Executive Summary

My primary objective for this project was to design and implement a robust, scalable, and secure network infrastructure for a new branch of **XYZ Company** located in Bonalbo. As a fast-growing entity, this branch required a network that operates independently from the headquarters while ensuring seamless communication between its internal departments. 

The business requirements I focused on included departmental isolation using **VLANs**, automated IP addressing via **DHCP**, and secure **Wireless LAN (WLAN)** access for mobile users. By leveraging Cisco's **Router-on-a-Stick (ROAS)** architecture, I aimed to achieve efficient Inter-VLAN routing using minimal hardware, optimizing both cost and performance for this Small Office Home Office (SOHO) environment.

## Network Topology

I designed the architecture following a classic star topology, centered around a core routing and switching fabric. My design integrates the following components:

*   **Edge Router:** This device acts as the default gateway and DHCP server for all subnets.
*   **Access Switch:** This handles Layer 2 segmentation and trunking.
*   **Departments:** 
    *   **Admin & IT (VLAN 10):** For management and technical operations.
    *   **Finance & HR (VLAN 20):** For sensitive financial and personnel data.
    *   **Reception & Customer Service (VLAN 30):** For front-desk operations and guest interaction.
*   **Wireless Infrastructure:** Cisco Access Points provide WPA2-PSK secured connectivity for each department.

![SOHO Network Topology Diagram](/assets/img/posts/soho-topology.png)

## Technical Implementation

### 1. Basic Device Security
My initial configuration involved securing the privileged EXEC mode to prevent unauthorized access to the CLI. I used a simple password for this lab environment, but in a real-world scenario, I would implement stronger security measures.

```bash
Router> enable
Router# configure terminal
Router(config)# enable secret cisco
```

### 2. VLAN Configuration and Port Assignment
To ensure logical separation, I created three distinct VLANs. I remembered that by default, all ports reside in VLAN 1, so moving them to specific VLANs was crucial for enhancing security and reducing broadcast traffic.

```bash
Switch(config)# vlan 10
Switch(config-vlan)# name Admin_IT
Switch(config)# vlan 20
Switch(config-vlan)# name Finance_HR
Switch(config)# vlan 30
Switch(config-vlan)# name Reception
```

### 3. Trunking and Security Hardening
The link between the switch and the router must be a **Trunk** to carry multiple VLAN tags. To follow security best practices, I restricted the allowed VLANs on the trunk port to only those necessary for the operation.

```bash
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
```

> **Troubleshooting Note:** I encountered an issue where `show interfaces trunk` returned nothing. After some investigation, I realized that the router-side interface was in an `operational mode: down` state. The solution was to activate the router's interface with the `no shutdown` command. This immediately resolved the problem, and `show interfaces trunk` then displayed the expected output. This experience reinforced the importance of checking interface status on both ends of a link.

### 4. Inter-VLAN Routing (Router-on-a-Stick)
I implemented **ROAS** by creating sub-interfaces on the router's physical interface. Each sub-interface is mapped to a specific VLAN using **802.1Q encapsulation**. This command tells the router to treat any frame arriving with the specified tag number as if it belongs to that sub-interface.

```bash
Router(config)# interface GigabitEthernet0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.1.1 255.255.255.192

Router(config)# interface GigabitEthernet0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.1.65 255.255.255.192

Router(config)# interface GigabitEthernet0/0.30
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 192.168.1.129 255.255.255.192
```

After configuring the sub-interfaces, I made sure to save the running configuration to NVRAM to prevent loss of work upon device reboot.

### 5. DHCP Server Configuration
I configured the router as a DHCP server to automate IP assignment for devices in each VLAN. It was crucial to exclude the gateway addresses to prevent IP conflicts.

```bash
Router(config)# ip dhcp excluded-address 192.168.1.1
Router(config)# ip dhcp pool VLAN10_POOL
Router(dhcp-config)# network 192.168.1.0 255.255.255.192
Router(dhcp-config)# default-router 192.168.1.1

Router(config)# ip dhcp excluded-address 192.168.1.65
Router(config)# ip dhcp pool VLAN20_POOL
Router(dhcp-config)# network 192.168.1.64 255.255.255.192
Router(dhcp-config)# default-router 192.168.1.65

Router(config)# ip dhcp excluded-address 192.168.1.129
Router(config)# ip dhcp pool VLAN30_POOL
Router(dhcp-config)# network 192.168.1.128 255.255.255.192
Router(dhcp-config)# default-router 192.168.1.129
```

### 6. Wireless LAN Security
I secured each department's Access Point using **WPA2-PSK** to ensure that only authorized personnel could join the wireless network. This was a critical step to protect the wireless segments of the network.

![WPA2-PSK Configuration on Cisco AP](/assets/img/posts/ap-security-config.png)

## Verification & Troubleshooting

### Connectivity Testing
The ultimate proof of a successful implementation is end-to-end connectivity. I performed cross-VLAN pings to verify that the Router-on-a-Stick was correctly forwarding traffic between subnets.

**Test Case:** Ping from Admin PC (VLAN 10) to Reception PC (VLAN 30).

```bash
PC> ping 192.168.1.130
Pinging 192.168.1.130 with 32 bytes of data:
Reply from 192.168.1.130: bytes=32 time<1ms TTL=127
Reply from 192.168.1.130: bytes=32 time<1ms TTL=127

Ping statistics for 192.168.1.130:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

![Successful Ping Verification](/assets/img/posts/ping-test-results.png)

This successful ping confirmed that Inter-VLAN routing was functioning as expected, and connectivity was established across different departments.

### Final Configuration Audit
*   **VLAN Table:** I verified the VLAN configurations using `show vlan brief`.
*   **Trunk Status:** I confirmed the trunk link's operational status and allowed VLANs with `show interfaces trunk`.
*   **Routing Table:** I checked the routing table with `show ip route` to ensure all sub-interfaces were "Up/Up" and routes were correctly installed.

I believe this project successfully meets all business requirements, providing a secure and efficient foundation for XYZ Company's new branch operations. This hands-on experience was invaluable in solidifying my understanding of SOHO network design and implementation principles.
