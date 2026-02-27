---
title: "My SOHO Network Design: VLANs, ROAS, and DHCP Implementation Journey"
date: 2026-02-26 09:00:00 +0000
categories: [Networking, Security]
tags: [cisco, vlan, routing, dhcp, soho, packet-tracer, troubleshooting]
description: A personal technical write-up on designing and implementing a Small Office Home Office (SOHO) network, detailing my approach to Inter-VLAN routing, secure wireless access, and the challenges I overcame.
---

## Executive Summary

My primary objective for this project was to design and implement a robust, scalable, and secure network infrastructure for a new branch of a company located in Rabat. As a fast-growing entity, this branch required a network that operates independently from the headquarters while ensuring seamless communication between its internal departments. 

The business requirements I focused on included departmental isolation using **VLANs**, automated IP addressing via **DHCP**, and secure **Wireless LAN (WLAN)** access for mobile users. By leveraging Cisco's **Router-on-a-Stick (ROAS)** architecture, I aimed to achieve efficient Inter-VLAN routing using minimal hardware, optimizing both cost and performance for this Small Office Home Office (SOHO) environment.

## Network Topology

I designed the architecture following a classic star topology, centered around a core routing and switching fabric. My design integrates the following components:

*   **Router:** This device acts as the default gateway and DHCP server for all subnets.
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

```
Router> enable
Router# configure terminal
Router(config)# enable secret #C0mpl&xP@ssuu0rd4R0<>t&R
```
As i issued the `show running-config` command, I observed the password being encrypted in the running configuration.
![image showing our password ciphered](/assets/img/posts/runconf.png)


### 2. VLAN Configuration and Port Assignment
To ensure logical separation, I created three distinct VLANs. I remembered that by default, all ports reside in VLAN 1, so moving them to specific VLANs was crucial for enhancing security and reducing broadcast traffic.
Let's first review the VLAN table.
![VLAN Table](/assets/img/posts/vtable.png)
By default all the interfaces are in vlan1.
vlan 1 and 1002 through 1005 exist by default and cannot be deleted.
**Let's configure VLANS 10, 20, and 30:**
1. VLAN 10 for Admin & IT:
![Admin VLAN](/assets/img/posts/10.png)
![VLAN admin](/assets/img/posts/10.1.png)
1. VLAN 20 for Finance & HR:
![Finance VLAN](/assets/img/posts/20.png)
3. VLAN 30 for Reception & Customer Service:
![VLAN reception](/assets/img/posts/30.png)
Let's not forget to save our configuration into switch NVRAM.
![save config](/assets/img/posts/write.png)
### 3. Trunking and Security Hardening
The link between the switch and the router must be a trunk to carry multiple VLAN tags. To follow security best practices, I restricted the allowed VLANs on the trunk port to only those necessary for the operation.

```
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 10,20,30
```
![trunk configuration in switchport](/assets/img/posts/trunk1.png)
> **Troubleshooting Note:** I encountered an issue where `show interfaces trunk` returned nothing. After some investigation, I realized that the router-side interface was in an 'operational mode: down' state. The solution was to activate the router's interface with the 'no shutdown' command. This immediately resolved the problem, and 'show interfaces trunk' then displayed the expected output. This experience reinforced the importance of checking interface status on both ends of a link.

As you see in the command line interface, no result is given back:
![No result back](/assets/img/posts/tr2.png)
Realizing there was an issue with my interface, I issued the 'show interfaces switchport' command to investigate further:
![I know the cause](/assets/img/posts/tr3.png)
And here we can clearly see the issue: The operational mode of the interface is down. This is a common issue when a link is down or not configured. The solution is to activate the interface by removing the 'shutdown' command.

As a security measure, I restricted the allowed VLANs on the trunk port to only those essential for operations. Since all VLANs are permitted by default, an attacker could potentially exploit this to perform VLAN hopping or intercept traffic from unauthorized segments. By explicitly defining the allowed list, even if new VLANs are created later, they remain isolated and cannot traverse the trunk unless explicitly permitted, thereby reducing the network's attack surface. 
![image showing allowed vlans](/assets/img/posts/tr4.png)

### 4. Inter-VLAN Routing (Router-on-a-Stick)
I implemented ROAS by creating sub-interfaces on the router's physical interface. Each sub-interface is mapped to a specific VLAN using 802.1Q encapsulation. This command tells the router to treat any frame arriving with the specified tag number as if it belongs to that sub-interface.

```
Router(config)# interface gig0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.1.62 255.255.255.192
```
The command 'encapsulation dot1Q 10' tells the router to treat any frame with the tag number 10 as if it belongs to the sub-interface.
![subinterface2](/assets/img/posts/roas0.png)
![subinterface2](/assets/img/posts/ro1.png)
![subinterface3](/assets/img/posts/ro2.png)
![subinterface3](/assets/img/posts/ro3.png)
After configuring the sub-interfaces, I made sure to save the running configuration to NVRAM to prevent loss of work upon device reboot.

### 5. DHCP Server Configuration
I configured the router as a DHCP server to automate IP assignment for devices in each VLAN. It was crucial to exclude the gateway addresses to prevent IP conflicts.
Let's activate the DHCP service first:
```
Router(config)# service dhcp
```
Her i excluded the gateway addresses to prevent IP conflicts:
![Address Exclusion](/assets/img/posts/dh1.png)
Then i moved on to configure the DHCP specifications for each Vlan, here is an example for VLAN 20:
![DHCP Configuration](/assets/img/posts/dh2.png)
After that, i configured the devices in each Vlan to automatically obtain their IP addresses from the DHCP server.
We can clearly see that the devices have obtained their IP addresses as shown below:
![DHCP Configuration](/assets/img/posts/dh3.png) 

### 6. Wireless LAN Security
I secured each department's Access Point using **WPA2-PSK** to ensure that only authorized personnel could join the wireless network. This was a critical step to protect the wireless segments of the network. Here is an example from the IT AP:

![WPA2-PSK Configuration on Cisco AP](/assets/img/posts/ap.png)

## Verification & Troubleshooting

### Connectivity Testing
The ultimate proof of a successful implementation is end-to-end connectivity. I performed cross-VLAN pings to verify that the Router-on-a-Stick was correctly forwarding traffic between subnets.

**Test Case:** Ping from PC3 (VLAN 30) on PC2 (VLAN 10).

![Successful Ping Verification](/assets/img/posts/ping.png)

This successful ping confirmed that Inter-VLAN routing was functioning as expected, and connectivity was established across different departments.

### Final Configuration Audit
*   **VLAN Table:** I verified the VLAN configurations using 'show vlan brief'.
*   **Trunk Status:** I confirmed the trunk link's operational status and allowed VLANs with 'show interfaces trunk'.
*   **Routing Table:** I checked the routing table with 'show ip route' to ensure all sub-interfaces were "Up/Up" and routes were correctly installed.
## Final Thoughts
I believe this project successfully meets the core business requirements, providing a secure and efficient foundation for the company's new branch. While it serves as a robust starting point, it is designed to be scalable, allowing for future security enhancements as the branch's operational needs evolve. This hands-on experience has been invaluable in bridging the gap between theory and practice, solidifying my understanding of SOHO network design and implementation.