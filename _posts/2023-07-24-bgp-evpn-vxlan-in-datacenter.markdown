---
title: "BGP EVPN-VxLAN in Datacenter"
layout: post
date: 2023-07-24
image: /assets/images/2023-07-24-bgp-evpn-vxlan-in-datacenter/bgp-evpn-logo.png
headerImage: true
tag:
- BGP
- EVPN
- VxLAN
- Datacenter
- Cisco
- Network
category: blog
blog: true
author: Satish Patel
description: "BGP EVPN-VxLAN in Datacenter"

---

### Understanding EVPN

* In traditional Layer 2 networks, reachability information is distributed in the data plane through flooding. With EVPN-VXLAN networks, this activity moves to the control plane.

* EVPN is an extension to Border Gateway Protocol (BGP) that allows the network to carry endpoint reachability information such as Layer 2 MAC addresses and Layer 3 IP addresses. This control plane technology uses MP-BGP for MAC and IP address endpoint distribution, where MAC addresses are treated as routes.

* Because MAC learning is now handled in the control plane, it avoids the flooding typical with layer 2 networks. EVPN can support different data-plane encapsulation technologies between EVPN-VXLAN-enabled switches. With EVPN-VXLAN architectures, VXLAN provides the overlay data-plane encapsulation.


* Network overlays are created by encapsulating traffic and tunneling it over a physical network. The VXLAN tunneling protocol encapsulates Layer 2 Ethernet frames in Layer 3 UDP packets, enabling Layer 2 virtual networks or subnets that can span the underlying physical Layer 3 network. The device that performs VXLAN encapsulation and decapsulation is called a VXLAN tunnel endpoint (VTEP). EVPN enables devices acting as VTEPs to exchange reachability information with each other about their endpoints.

* In a VXLAN overlay network, each Layer 2 subnet or segment is uniquely identified by a virtual network identifier (VNI). A VNI segments traffic the same way that a VLAN ID segments traffic - endpoints within the same virtual network can communicate directly with each other, while endpoints in different virtual networks require a device that supports inter-VNI (inter-VXLAN) routing.


### Cisco CML LAB

I'm using Cisco Modeling lab to build EVPN-VxLAN fabric. I am using Cisco Nexus 9K Switches to build fabric. [Enlarge image](https://user-images.githubusercontent.com/10041875/256011691-5bbe697f-217f-4759-b0cc-6818016906ac.png)

![<img>](/assets/images/2023-07-24-bgp-evpn-vxlan-in-datacenter/bgp-evpn-vxlan-lab.png){: width="1000" }

LAB Details

* Spine 
* Leaf (Pair of leaf in vPC setup)
* Border Leaf ( ISP connectivity)
* ASA Firewall ( Gateway for all Internal VLANs )
* IOS Routers ( To mimic ISP setup ) 


### Underlay 

An underlay network is the physical network over which the virtual overlay (vxlan) network is established. In BGP EVPN VXLAN, the underlay Layer 3 network transports the VXLAN-encapsulated packets between the source and destination VTEPs and provides reachability between them. The VXLAN overlay and the underlying IP network between the VTEPs are independent of each other.

I'm using OSPF/BGP for Underlay to build L3 fabric. 

### Overlay

An overlay network is a virtual network that is built over an existing Layer 2 or Layer 3 network by forming a static or dynamic tunnel that runs on top of the physical network infrastructure. 

In the context of BGP EVPN VXLAN, VXLAN is used as the overlay technology to encapsulate the data packets and tunnel the traffic over a Layer 3 network. VXLAN creates a Layer 2 overlay network by using a MAC-in-UDP encapsulation. A VXLAN header is added to the original Layer 2 frame and it is then placed within a UDP-IP packet. A VXLAN overlay network is also called as a VXLAN segment. Only host devices and virtual machines within the same VXLAN segment can communicate with each other.

### Spine configuration 

Spine switches has fairly simple configuration. It runs OSPF, BGP & Multicast services only. They don't know anything about VxLAN because it's job is to just push packets between leafs. 

Enable features

```
nv overlay evpn
feature ospf
feature bgp
feature pim
```

Make sure using Jambo frame in fabric to allow sending more data in single packet. 

```
policy-map type network-qos jumboframes
  class type network-qos class-default
    mtu 9216
!
system qos
  service-policy type network-qos jumboframes
```

Configure multicast for BUM traffic. 

```
ip pim rp-address 10.255.0.123 group-list 239.0.0.0/8
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 10.255.0.123 10.255.0.1
ip pim anycast-rp 10.255.0.123 10.255.0.2
```

Configure two Loopback interface which later we will use for ip unnumbered setup and multicast RP.  

```
interface loopback0
  description ** RID/BGP Overlay **
  ip address 10.255.0.1/32
  ip router ospf UNDERLAY-NET area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description ** Anycat-RP address **
  ip address 10.255.0.123/32
  ip ospf authentication-key 3 fa3ab8e90610229c
  ip router ospf UNDERLAY-NET area 0.0.0.0
  ip pim sparse-mode
```

Configure OSPF

```
router ospf UNDERLAY-NET
  router-id 10.255.0.1
  log-adjacency-changes
  area 0.0.0.0 authentication
```

Using IP Unnumbered to simplify configuration. Like following example to connect leaf switch using ip unnumbered. 

```
interface Ethernet1/1
  description ** Leaf-1-a Port E1/1 **
  no switchport
  mtu 9216
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 fa3ab8e90610229c
  ip ospf network point-to-point
  ip router ospf UNDERLAY-NET area 0.0.0.0
  ip pim sparse-mode
  no shutdown
```

BGP configuration snippet. Both Spines are OSPF route reflactor. 

```
router bgp 65001
  router-id 10.255.0.1
  log-neighbor-changes
  template peer VXLAN_LEAF
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 10.255.1.1
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Border-Leaf-1-a **
  neighbor 10.255.1.2
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Border-Leaf-1-b **
  neighbor 10.255.1.11
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Leaf-1-a **
  neighbor 10.255.1.12
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Leaf-1-b **
  neighbor 10.255.1.21
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Leaf-2-a **
  neighbor 10.255.1.22
    inherit peer VXLAN_LEAF
    description ** iBGP Peer to Leaf-2-b **
```

### Leaf configuration 

Leaf switches runs OSPF, BGP, Multicast & VTEP to form VxLAN tunnel between leafs. In LAB I'm running Cisco vPC to pair switches to create single VTEP endpoint. It act like a single switch. 

Enable features 

```
nv overlay evpn
feature ospf
feature bgp
feature pim
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature nv overlay
```

Enable jambo frames 

```
policy-map type network-qos jumboframes
  class type network-qos class-default
    mtu 9216
!
system qos
  service-policy type network-qos jumboframes
```

Setup virtual MAC for anycast gateway 

```
fabric forwarding anycast-gateway-mac 0000.dead.beef
```

Setup multicast 

```
ip pim rp-address 10.255.0.123 group-list 239.0.0.0/8
ip pim ssm range 232.0.0.0/8
```

Map local VLAN ID with VxLAN ID (vni). VLAN 444 & 555 are special VLANs. 444 use for vPC setup and 555 use for Inter-VLAN routing.

```
vlan 60
  name ops
  vn-segment 10060
vlan 61
  name sales
  vn-segment 10061
vlan 100
  name public1
  vn-segment 10100
vlan 444
  name BACKUP_VLAN_ROUTING_VPC
vlan 555
  name L3VNI-For-IRB
  vn-segment 10555
```

Configure vRF name ISP (This is my tenant to access Public network)

```
vrf context ISP
  vni 10555
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
  address-family ipv6 unicast
    route-target both auto
    route-target both auto evpn
```

Configure vPC (Assuming you know how to setup vPC)

```
vpc domain 1
  peer-switch
  role priority 10
  peer-keepalive destination 172.30.0.32 source 172.30.0.31
  delay restore 90
  peer-gateway
  delay restore interface-vlan 30
  ipv6 nd synchronize
  ip arp synchronize
```

VLAN 100 is my Public VLAN to access Internet 

```
interface Vlan100
  description ** Anycast Gateway For Public  **
  no shutdown
  mtu 9216
  vrf member ISP
  no ip redirects
  ip address 69.25.124.1/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
```

Add vRF ISP to VLAN 444 and 555 

```
interface Vlan444
  description ** Underlay Backup over vPC Peer-Link **
  no shutdown
  no ip redirects
  ip address 192.168.1.1/30
  no ipv6 redirects
  ip ospf authentication-key 3 fa3ab8e90610229c
  ip ospf network point-to-point
  ip router ospf UNDERLAY-NET area 0.0.0.0
  ip pim sparse-mode
!
interface Vlan555
  description ** L3VNI-For-IRB **
  no shutdown
  mtu 9216
  vrf member ISP
  no ip redirects
  ip forward
  no ipv6 redirects
```

Configure VTEP endpoint (VxLAN)

```
interface nve1
  no shutdown
  description ** VTEP/NVE Interface **
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10060
    mcast-group 239.1.1.1
  member vni 10061
    mcast-group 239.1.1.1
  member vni 10100
    mcast-group 239.1.1.100
  member vni 10555 associate-vrf
```

Configure OSPF and BGP 

```
router ospf UNDERLAY-NET
  router-id 10.255.1.11
  log-adjacency-changes
  area 0.0.0.0 authentication
!
!
router bgp 65001
  router-id 10.255.1.11
  log-neighbor-changes
  template peer VXLAN_SPINE
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.255.0.1
    inherit peer VXLAN_SPINE
    description ** iBGP Peer to Spine-1 **
  neighbor 10.255.0.2
    inherit peer VXLAN_SPINE
    description ** iBGP Peer to Spine-2 **
  vrf ISP
    log-neighbor-changes
    address-family ipv4 unicast
      redistribute direct route-map DIRECT-PERMIT-ALL
    address-family ipv6 unicast
      redistribute direct route-map DIRECT-PERMIT-ALL
```

Setup route-map (This to redistribute routes)

```
route-map DIRECT-PERMIT-ALL permit 10
  description ** Route-Map for BGP to redist route **
```

Configure EVPN import/export map with vni 

```
evpn
  vni 10060 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 10061 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```

### Border Leaf

This is regular leaf setup but it doesn't have vPC and any other internal VLANs. I keep it dedicated for external connectivity for ISP or connect any other EVPN fabric in future. 

Enable features 

```
feature ospf
feature bgp
feature pim
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
```

Enable Jambo frams 

```
policy-map type network-qos jumboframes
  class type network-qos class-default
    mtu 9216
!
system qos
  service-policy type network-qos jumboframes
```

Setup anycast virtual MAC

```
fabric forwarding anycast-gateway-mac 0000.dead.beef
```

Setup multicast 

```
ip pim rp-address 10.255.0.123 group-list 239.0.0.0/8
ip pim ssm range 232.0.0.0/8
```

Configure VLANs mapping with vni. We have only VLAN 100 and 555 because VLAN 100 is public VLAN and 555 use for L3VNI routing. 

```
vlan 100
  name Public_VLAN
  vn-segment 10100
vlan 555
  name L3VNI-For-IRB
  vn-segment 10555
```

Setup vRF ISP 

```
vrf context ISP
  description ** VRF-ISP **
  vni 10555
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
  address-family ipv6 unicast
    route-target both auto
    route-target both auto evpn
```

Configure VLAN interfaces. VLAN 100 is anycast address so same IP will be across all leafs. 

```
interface Vlan100
  description ** Anycast Gateway For Public  **
  no shutdown
  mtu 9216
  vrf member ISP
  no ip redirects
  ip address 69.25.124.1/24
  fabric forwarding mode anycast-gateway

interface Vlan555
  description ** L3VNI-For-IRB **
  no shutdown
  mtu 9216
  vrf member ISP
  ip forward
```

Configure VTEP endpoint for VLAN 100 & 555 VNI

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10100
    mcast-group 239.1.1.100
  member vni 10555 associate-vrf
```

Configure OSPF and BGP (You will see we have two BGP peers setup one more external ISP and second one for private iBGP peer)

```
router ospf UNDERLAY-NET
  log-adjacency-changes
  area 0.0.0.0 authentication
!
!
router bgp 65001
  router-id 10.255.1.2
  log-neighbor-changes
  template peer VXLAN_SPINE
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.255.0.1
    inherit peer VXLAN_SPINE
    description ** iBGP Peer to Spine-1 **
    no shutdown
  neighbor 10.255.0.2
    inherit peer VXLAN_SPINE
    description ** iBGP Peer to Spine-2 **
    no shutdown
  vrf ISP
    log-neighbor-changes
    address-family ipv4 unicast
      aggregate-address 69.25.124.0/23 summary-only
    neighbor 101.101.101.101
      remote-as 88888
      local-as 99999
      description ** ISP eBGP peer **
      address-family ipv4 unicast
        send-community
        send-community extended
```

Configure EVPN mapping with L2 VNI for VLAN 100 only.

```
evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```

### Firewall ASA

I have setup cisco ASA firewall for all my internal VLANs ACL and NAT to access public network. I have only single leg connected to ASA from leaf because Cisco Modeling lab doesn't support LACP bond on Virtual ASA.

Interface configuration of ASA

```
interface GigabitEthernet0/0
 description *** Link to Leaf-2-a ***
 speed 1000
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/0.60
 description *** LAN network ***
 vlan 60      
 nameif inside
 security-level 100
 ip address 10.0.0.1 255.255.255.0 
!             
interface GigabitEthernet0/0.100
 description *** Public Interface ***
 vlan 100     
 nameif outside
 security-level 0
 ip address 69.25.124.2 255.255.255.0 
```

Object group and ACL to allow ICMPs 

```
object-group network obj-NET-PRIVATE
 network-object 10.0.0.0 255.0.0.0
access-group 100 in interface outside
access-list 100 extended permit icmp any object-group obj-NET-PRIVATE echo-reply 
```

NAT (Overload mode) and Default gateway. 

```
nat (inside,outside) source dynamic any interface
route outside 0.0.0.0 0.0.0.0 69.25.124.1 1
```

### ISP Router setup 

I have mimic real ISP setup using two IOS routers. I have configure BGP ASN 88888 on it and peers with ASN 99999 (border leaf). 

R1 Interface configuration

```
interface GigabitEthernet0/0
 description *** Link to Border Leaf ***
 ip address 101.101.101.101 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 description *** Link to Internet ***
 ip address dhcp
 ip nat outside
 ip virtual-reassembly in
 duplex auto
 speed auto
 media-type rj45
!         
```

eBGP configuration. I'm using default-originate to send default route to EVPN fabric toward R1/R2. 

```
router bgp 88888
 bgp log-neighbor-changes
 network 1.1.1.0 mask 255.255.255.0
 neighbor 101.101.101.102 remote-as 99999
 neighbor 101.101.101.102 default-originate
```

I am running NAT here to have external connectivity for Internet access. I have external connector connected to R1/R2 to provide internet access. 

```
access-list 1 permit 69.25.124.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

### Validation 

iBGP peers to all leaf from spines 

```
spine-1# show ip bgp summary 

Neighbor        V    AS    MsgRcvd    MsgSent   TblVer  InQ OutQ Up/Down  State/
PfxRcd
10.255.1.1      4 65001       5479       5630       21    0    0    3d18h 0     
    
10.255.1.2      4 65001       5467       5623       21    0    0    3d18h 0     
    
10.255.1.11     4 65001       8730       8828       21    0    0    4d13h 0     
    
10.255.1.12     4 65001       6787       6972       21    0    0    4d13h 0     
    
10.255.1.21     4 65001       6745       6885       21    0    0    4d13h 0     
    
10.255.1.22     4 65001       6769       6954       21    0    0    4d13h 0     
```

Check BGP L2EVPN Routes

```
spine-1# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2111, Local Router ID is 10.255.0.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.255.1.2:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      10.255.255.2                      100          0 99999 888
88 i
*>i                   10.255.255.1                      100          0 99999 888
88 i
* i[5]:[0]:[0]:[23]:[69.25.124.0]/224
                      10.255.255.2                      100          0 i
*>i                   10.255.255.1                      100          0 i

Route Distinguisher: 10.255.1.11:4
*>i[5]:[0]:[0]:[24]:[69.25.124.0]/224
                      10.255.255.10            0        100          0 ?

Route Distinguisher: 10.255.1.11:32827
*>i[2]:[0]:[0]:[48]:[5254.001c.4a81]:[0]:[0.0.0.0]/216
                      10.255.255.10                     100          0 i

Route Distinguisher: 10.255.1.11:32867
*>i[2]:[0]:[0]:[48]:[5254.0008.df8b]:[0]:[0.0.0.0]/216
                      10.255.255.10                     100          0 i
*>i[2]:[0]:[0]:[48]:[5254.0008.df8b]:[32]:[69.25.124.100]/272
                      10.255.255.10                     100          0 i

Route Distinguisher: 10.255.1.12:4
*>i[5]:[0]:[0]:[24]:[69.25.124.0]/224
                      10.255.255.10            0        100          0 ?

Route Distinguisher: 10.255.1.12:32827
*>i[2]:[0]:[0]:[48]:[5254.001c.4a81]:[0]:[0.0.0.0]/216
                      10.255.255.10                     100          0 i

Route Distinguisher: 10.255.1.12:32867
*>i[2]:[0]:[0]:[48]:[5254.0008.df8b]:[0]:[0.0.0.0]/216
                      10.255.255.10                     100          0 i
*>i[2]:[0]:[0]:[48]:[5254.0008.df8b]:[32]:[69.25.124.100]/272
                      10.255.255.10                     100          0 i

Route Distinguisher: 10.255.1.21:4
*>i[5]:[0]:[0]:[24]:[69.25.124.0]/224
                      10.255.255.20            0        100          0 ?

Route Distinguisher: 10.255.1.21:32827
*>i[2]:[0]:[0]:[48]:[5254.0008.0572]:[0]:[0.0.0.0]/216
                      10.255.255.20                     100          0 i

Route Distinguisher: 10.255.1.21:32867
*>i[2]:[0]:[0]:[48]:[5254.001d.7a05]:[0]:[0.0.0.0]/216
                      10.255.255.20                     100          0 i
*>i[2]:[0]:[0]:[48]:[5254.001d.7a05]:[32]:[69.25.124.2]/272
                      10.255.255.20                     100          0 i

Route Distinguisher: 10.255.1.22:4
*>i[5]:[0]:[0]:[24]:[69.25.124.0]/224
                      10.255.255.20            0        100          0 ?

Route Distinguisher: 10.255.1.22:32827
*>i[2]:[0]:[0]:[48]:[5254.0008.0572]:[0]:[0.0.0.0]/216
                      10.255.255.20                     100          0 i

Route Distinguisher: 10.255.1.22:32867
*>i[2]:[0]:[0]:[48]:[5254.001d.7a05]:[0]:[0.0.0.0]/216
                      10.255.255.20                     100          0 i
*>i[2]:[0]:[0]:[48]:[5254.001d.7a05]:[32]:[69.25.124.2]/272
                      10.255.255.20                     100          0 i
```

Check mulicast and PIM neighbours 

```
spine-1# show ip pim neighbor 
PIM Neighbor Status for VRF "default"
Neighbor        Interface            Uptime    Expires   DR       Bidir-  BFD   ECMP Redirect
                                                         Priority Capable State Capable
10.255.1.11     Ethernet1/1          4d15h     00:01:42  1        yes     n/a   no
10.255.1.12     Ethernet1/2          4d14h     00:01:43  1        yes     n/a   no
10.255.1.21     Ethernet1/3          4d14h     00:01:37  1        yes     n/a   no
10.255.1.22     Ethernet1/4          4d14h     00:01:26  1        yes     n/a   no
10.255.1.1      Ethernet1/5          3d19h     00:01:23  1        yes     n/a   no
10.255.1.2      Ethernet1/6          3d19h     00:01:39  1        yes     n/a   no
```

Validate PIM RP active members

```
spine-1# show ip pim rp 
PIM RP Status Information for VRF "default"
BSR disabled
Auto-RP disabled
BSR RP Candidate policy: None
BSR RP policy: None
Auto-RP Announce policy: None
Auto-RP Discovery policy: None

Anycast-RP 10.255.0.123 members:
  10.255.0.1*  10.255.0.2  

RP: 10.255.0.123*, (0), 
 uptime: 6d14h   priority: 255, 
 RP-source: (local),  
 group ranges:
 239.0.0.0/8   
```

Check active VTEP peers (VxLAN tunnels peers) from Leaf-1-a. As you can see we have two VxLAN tunnel active leaf-2 and border-leaf. following IPs are loopback IPs. 

```
leaf-1-a# show nve peers 
Interface Peer-IP                                 State LearnType Uptime   Router-Mac       
--------- --------------------------------------  ----- --------- -------- -----------------
nve1      10.255.255.1                            Up    CP        3d18h    5215.f753.1b08   
nve1      10.255.255.20                           Up    CP        4d13h    520cb94a.1b08   
```

Check VLAN/VNI Mapping with multicast address

```
leaf-1-a# show nve vni 
Codes: CP - Control Plane        DP - Data Plane          
       UC - Unconfigured         SA - Suppress ARP        
       S-ND - Suppress ND        
       SU - Suppress Unknown Unicast 
       Xconn - Crossconnect      
       MS-IR - Multisite Ingress Replication 
       HYB - Hybrid IRB mode
    
Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10060    239.1.1.1         Up    CP   L2 [60]                 
nve1      10061    239.1.1.1         Up    CP   L2 [61]                 
nve1      10100    239.1.1.100       Up    CP   L2 [100]                
nve1      10555    n/a               Up    CP   L3 [ISP]                
```

Check VRF ISP routes on border-leaf switches

```
border-1-a# show ip bgp vrf ISP. You can see ASA and Servers advertised their IPs in BGP. 

Network            Next Hop            Metric     LocPrf     Weight Path
*>e0.0.0.0/0          101.101.101.101                                0 99999 88888 i
*>a69.25.124.0/24     0.0.0.0                           100      32768 i
* i                   10.255.255.20            0        100          0 ?
* i                   10.255.255.20            0        100          0 ?
* i                   10.255.255.10            0        100          0 ?
* i                   10.255.255.10            0        100          0 ?
s i69.25.124.2/32     10.255.255.20                     100          0 i
s>i                   10.255.255.20                     100          0 i
s i69.25.124.100/32   10.255.255.10                     100          0 i
s>i                   10.255.255.10                     100          0 i
```

Ping from Server-1 (10.0.0.10)

```
root@server-1:~# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=36.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=41.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=111 time=42.5 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=111 time=39.6 ms
```

### LAB Configuration files for all devices

https://github.com/satishdotpatel/cisco-evpn-vxlan-lab/
















