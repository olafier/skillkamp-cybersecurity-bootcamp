# Week 01 — Network Fundamentals & Security Configuration

**Program:** SkillKamp Cybersecurity Bootcamp  
**Date:** October 31, 2026  
**Platform:** Cisco Packet Tracer 9.0.1  
**Lab Environment:** Simulated Enterprise Network  
**Format:** Instructor-Guided Hands-on Workshop

---

## 1. Overview

The first workshop focused on fundamental networking concepts and practical network configuration using Cisco Packet Tracer.

The activities involved configuring routers, switches, end devices, wireless networks, and network services within a simulated enterprise infrastructure.

The workshop also introduced fundamental network security concepts, including VLAN segmentation, wireless authentication, PPP/PAP authentication, static routing, and Access Control Lists (ACLs).

### Learning Objectives

- Understand the OSI model and fundamental networking protocols.
- Configure Cisco router and switch interfaces.
- Assign and verify IPv4 addresses.
- Configure and test DHCP, HTTP, HTTPS, and FTP services.
- Implement VLAN segmentation using access and trunk ports.
- Explore wireless security configurations.
- Configure PPP/PAP authentication between routers.
- Implement static and default routing.
- Apply ACL rules to control network traffic.
- Verify network connectivity and troubleshoot configuration issues.

---

## 2. Network Topology

The workshop used Cisco Packet Tracer to simulate a multi-network environment consisting of branch and headquarters networks, an ISP router, switches, wireless devices, and servers.

The topology demonstrated connectivity across LAN and WAN segments, routing between different networks, and traffic control through network security configurations.

### Enterprise Network Simulation

![Enterprise Network Topology](screenshots/network-topology.png)

*Figure 1. Enterprise network topology used during the Week 01 workshop.*

### Network Components

| Component | Purpose |
|---|---|
| Routers | Connect multiple networks and forward packets |
| Switches | Connect devices within local networks |
| Wireless Router | Provide wireless connectivity |
| PCs and Laptops | Simulate end-user devices |
| Servers | Provide network services |
| Serial WAN Links | Connect routers across WAN segments |

---

## 3. Router Configuration

### Objective

Configure a Cisco router interface with a static IPv4 address and enable network connectivity.

### Configuration

```bash
enable
configure terminal

interface gigabitEthernet0/0
ip address 192.168.2.254 255.255.255.0
no shutdown
exit

interface loopback0
ip address 8.8.8.8 255.255.255.255
end

write
```

### Verification

```bash
show ip interface brief
```

Expected interface states based on the workshop:

| Interface | IP Address | Status | Protocol |
|---|---|---|---|
| GigabitEthernet0/0 | 192.168.2.254 | up | up |
| Loopback0 | 8.8.8.8 | up | up |

### Key Learning

The `no shutdown` command enables an administratively disabled router interface.

The `show ip interface brief` command allows administrators to verify interface addressing and operational status.

The loopback interface was configured with `8.8.8.8/32` as a simulated local interface address for connectivity testing.

---

## 4. IP Addressing and Connectivity Testing

### Objective

Configure end-device IP settings and verify connectivity between devices.

### Example IP Configuration

| Parameter | Value |
|---|---|
| IPv4 Address | 192.168.2.253 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.2.254 |
| DNS Server | 8.8.8.8 |

### Testing Commands

```bash
ipconfig
ipconfig /all
ping 192.168.2.254
ping 8.8.8.8
```

### Observed Results

The workshop records showed successful ICMP connectivity with four packets sent and received and zero packet loss.

### Key Learning

IP addressing and default gateways determine whether devices can communicate within and outside their local networks.

ICMP testing helps verify reachability but does not guarantee that all application services are functioning.

---

## 5. Network Services Configuration

### Objective

Configure and test common network services within the simulated environment.

### DHCP

Configured a DHCP pool to assign IP addresses automatically to network clients.

The workshop demonstrated automatic addressing for wireless devices, including smartphones and tablets.

### HTTP and HTTPS

Tested access to a simulated web server using:

```text
http://192.168.2.253
https://192.168.2.253
```

### FTP

Established an FTP session and listed server files.

```bash
ftp 192.168.2.253
dir
quit
```

### Key Learning

Network services support device configuration, web communication, and file transfer. Their accessibility depends on correct addressing, connectivity, and relevant security rules.

---

## 6. VLAN Configuration and Segmentation

### Objective

Create separate logical networks using VLANs and configure switch ports.

### VLAN Design

| VLAN ID | Name | Purpose |
|---|---|---|
| 20 | Internet | Logical network segment |
| 30 | User | User network segment |

### Switch SW-A Configuration

```bash
enable
configure terminal

hostname SW-A

vlan 20
name Internet
exit

vlan 30
name User
exit

interface fastEthernet0/24
switchport mode trunk
exit

interface range fastEthernet0/1-2
switchport mode access
switchport access vlan 20
exit

interface fastEthernet0/3
switchport mode access
switchport access vlan 30
end
```

### Switch SW-B Configuration

```bash
enable
configure terminal

hostname SW-B

interface fastEthernet0/24
switchport mode trunk
exit

vlan 20
name Internet
exit

vlan 30
name User
exit

interface fastEthernet0/2
switchport mode access
switchport access vlan 20
exit

interface fastEthernet0/3
switchport mode access
switchport access vlan 30
end
```

### Verification

```bash
show vlan brief
show interfaces trunk
show mac address-table
```

### Key Learning

VLANs logically separate devices into different broadcast domains.

Access ports connect devices to a specific VLAN, while trunk ports can carry traffic from multiple VLANs between switches.

VLANs support network segmentation, which can be combined with routing policies and ACLs to enforce communication boundaries.

---

## 7. Wireless Network Security

### Objective

Explore wireless connectivity and security settings using a simulated wireless router.

### Topics Practiced

- SSID configuration
- Wireless client connection
- WPA2 Personal
- WPA2 Enterprise
- AES encryption
- Wireless MAC address filtering

### Configuration Exploration

The workshop included wireless router configuration screens and wireless client settings in Cisco Packet Tracer.

### Key Learning

Wireless authentication and encryption help protect wireless network access.

WPA2 Enterprise supports centralized authentication, while MAC filtering provides an additional but limited access restriction mechanism.

---

## 8. WAN Configuration and PPP Authentication

### Objective

Configure authentication between the B1 and HQ routers over a Serial WAN connection.

### B1 Router

```bash
enable
configure terminal

username admin password cisco

interface serial0/0/0
encapsulation ppp
ppp authentication pap
ppp pap sent-username user password 123456
end
```

### HQ Router

```bash
enable
configure terminal

username user password 123456

interface serial0/0/0
encapsulation ppp
ppp authentication pap
ppp pap sent-username admin password cisco
end
```

*The credentials shown above are examples from the educational lab and should not be used in production.*

### Verification

```bash
show interfaces serial0/0/0
ping 22.22.22.22
```

### Observed Results

The WAN interface was reported as operational, and the ICMP test recorded a 100% success rate (5/5 packets).

### Key Learning

PPP provides a data-link protocol for point-to-point connections.

PAP introduces basic authentication concepts, but its password handling makes it unsuitable as a modern security mechanism without additional protection.

---

## 9. Static Routing

### Objective

Configure routes between the branch router, headquarters, and ISP environment.

### B1 Default Route

```bash
ip route 0.0.0.0 0.0.0.0 22.22.22.22
```

### HQ Router

```bash
ip route 0.0.0.0 0.0.0.0 33.33.33.34
ip route 11.11.11.12 255.255.255.252 22.22.22.21
ip route 192.169.1.0 255.255.255.0 22.22.22.21
```

### ISP Router

```bash
ip route 0.0.0.0 0.0.0.0 33.33.33.33
```

### Verification

```bash
show ip route
```

### Routing Table Indicators

| Code | Meaning |
|---|---|
| C | Connected Network |
| L | Local Interface Address |
| S | Static Route |
| S* | Candidate Default Route |

### Key Learning

Static routing allows administrators to manually define network paths.

Default routes direct traffic toward a specified next hop when no more specific matching route exists.

---

## 10. Access Control Lists (ACLs)

### Objective

Implement network traffic filtering using an Extended ACL.

### ACL Configuration

Configured on the HQ router:

```bash
enable
configure terminal

ip access-list extended ACL1
deny tcp any any eq 80
deny icmp any any
permit ip any any
exit

interface serial0/0/1
ip access-group ACL1 out
end
```

### Verification

```bash
show ip access-lists
```

### Traffic Testing

| Protocol | Test | Observed Result |
|---|---|---|
| HTTP | TCP Port 80 | Blocked |
| ICMP | Ping | Blocked |
| HTTPS | TCP Port 443 | Allowed |
| FTP | File Transfer | Allowed |

### Key Learning

Extended ACLs allow administrators to control traffic based on conditions such as protocols, IP addresses, and port numbers.

ACL placement and direction determine which packets are evaluated.

The exercise demonstrated basic network traffic filtering rather than full stateful firewall functionality.

---

## 11. Troubleshooting and Verification Commands

| Command | Purpose |
|---|---|
| `ping` | Test network reachability |
| `ipconfig` | Display client IP configuration |
| `ipconfig /all` | Display detailed network configuration |
| `show ip interface brief` | Verify router interface status |
| `show mac address-table` | Inspect learned MAC addresses |
| `show vlan brief` | Verify VLAN assignments |
| `show interfaces trunk` | Verify trunk configuration |
| `show ip route` | Inspect the routing table |
| `show ip access-lists` | Inspect ACL rules |

---

## 12. Security Lessons Learned

### Network Segmentation

VLANs can separate devices into logical groups and help establish network boundaries.

### Authentication

The PPP/PAP and wireless security exercises introduced the role of authentication in controlling network access.

### Traffic Filtering

Extended ACLs demonstrated how traffic can be selectively permitted or denied based on configured rules.

### Network Visibility

Understanding routing tables, interface states, and traffic behavior supports effective network monitoring and troubleshooting.

### Cybersecurity Relevance

These concepts establish a technical foundation for network security monitoring, security operations, and cloud networking.

---

## 13. Workshop Reflection

The first workshop helped me connect networking theory with practical configuration exercises.

Working with Cisco Packet Tracer provided a clearer understanding of how routers, switches, servers, and end devices communicate within a network.

The most valuable learning experience was observing how network configuration and security policies directly influence connectivity and traffic behavior.

This workshop reinforced the importance of networking fundamentals as a prerequisite for more advanced cybersecurity topics.

---

## 14. Tools and Technologies

| Category | Technologies |
|---|---|
| Network Simulator | Cisco Packet Tracer 9.0.1 |
| Network Configuration | Cisco IOS CLI |
| Networking | IPv4, LAN, WAN, VLAN, Static Routing |
| Network Services | DHCP, DNS, HTTP, HTTPS, FTP |
| Authentication | PPP, PAP, WPA2 |
| Security Controls | Extended ACL, Network Segmentation |
| Troubleshooting | ICMP, ipconfig, Routing Table Inspection |

---

## 15. Disclaimer

This documentation is based on an instructor-guided workshop and simulated exercises conducted for educational purposes.

IP addresses, credentials, and network configurations shown are lab examples and are not intended for production deployment.

The documentation reflects the workshop activities and learning outcomes. Original third-party training materials remain the property of their respective owners.
