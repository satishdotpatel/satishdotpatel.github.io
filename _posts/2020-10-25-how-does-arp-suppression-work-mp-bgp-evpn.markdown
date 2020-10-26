---
title: "How does arp suppression work with MP-BGP EVPN"
layout: post
date: 2020-10-25
image: /assets/images/2020-10-25-how-does-arp-suppression-work-mp-bgp-evpn/spine-leaf.png
headerImage: true
tag:
- Cisco
- EVPN+VxLAN
- Spine-Leaf
category: blog
blog: true
author: Satish Patel
description: "How does arp suppression work with MP-BGP EVPN"

---

ARP suppression is an enhancement provided by the MP-BGP EVPN control plane to reduce network flooding caused by broadcast traffic from ARP requests. When ARP suppression is enabled for a VNI, its VTEPs each maintain an ARP suppression cache table for known IP hosts and their associated MAC addresses in the VNI segment. when an end host in the VNI sends an ARP request for another end-host IP address, its local VTEP intercepts the ARP request and checks for the ARP-resolved IP address in its ARP suppression cache table. If it finds a match, the local VTEP sends an ARP response on behalf of the remote end host. The local host then learns the MAC address of the remote host in the ARP response. If the local VTEP doesn’t have the ARP-resolved IP address in its ARP suppression table, it floods the ARP request to the other VTEPs in the VNI. This ARP flooding can occur for the initial ARP request to a silent host in the network. The VTEPs in the network don’t see any traffic from the silent host until another host sends an ARP request for its IP address, and an ARP response is sent back. After the local VTEP learns about the MAC and IP addresses of the silent host, the information is distributed through the MP-BGP EVPN control plane to all other VTEPs. Any subsequent ARP requests do not need to be flooded.

I'm going to demonstrate how ARPs get suppressed with small lab work.

![<img>](/assets/images/2020-10-25-how-does-arp-suppression-work-mp-bgp-evpn/spine-leaf-arp-diag.png)

As you can see in above diagram i have two host in my spine-leaf fabric which i am going to use generate arp to see if my arp getting suppress or not populating in MP-BGP contole plane. 

Vlan-to-vni mapping

```
vlan 64
  name inside
  vn-segment 10064
```

nve1 interface mapping with vni/multicast (without suppress-arp option)

```
interface nve1
  no shutdown
  description ** VTEP/NVE Interface **
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10064
    mcast-group 239.1.2.3
```

Lets start sending arp packets from leaf-2-host (from 10.64.0.200 to 10.64.0.100). I'm using following command to delete arp and then send 10 icmp packet to generate arp. (following one liner script will generate 10 arp boradcast packets) 

```

[root@leaf-2-host ~]# for qw in `seq 1 10`; do arp -d 10.64.0.100 ; ping 10.64.0.100 -c 1; done

```

Run tcpdump on leaf-1-host (10.64.0.100) to capture arp packets. (as you can see in following output we received 10 arp boradcast packets, This results showing us without suppress-arp we are seeing flood of arp packets).

```
[root@leaf-1-host ~]# tcpdump -i eth0 -nn arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:56:31.285166 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.285179 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.290919 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.290932 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.296903 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.296916 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.302835 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.302848 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.308753 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.308767 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.314646 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.314659 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.320567 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.320580 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.326435 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.326448 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.332247 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.332260 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:31.337968 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
23:56:31.337981 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28
23:56:36.623865 ARP, Request who-has 10.64.0.200 tell 10.64.0.100, length 28
23:56:36.623992 ARP, Reply 10.64.0.200 is-at 8c:dc:d4:b1:38:28, length 46
```

Lets enable suppress-arp on VNI 10064/Vlan64 on both leaf-1 and leaf-2 and run same test to see the results.

```
leaf-1(config)# int nve1
leaf-1(config-if-nve)# member vni 10064
leaf-1(config-if-nve-vni)# suppress-arp

leaf-2(config)# int nve1
leaf-2(config-if-nve)# member vni 10064
leaf-2(config-if-nve-vni)# suppress-arp
```

Let's verify arp suppression is enabled on vlan 64 or not.

```
leaf-2# show ip arp suppression topo-info 64
ARP L2RIB Topology information
Topo-id  ARP-suppression mode(HMM SDB value)
64       L2 ARP Suppression
```
In above output its saying you does have arp suppression enabled for L2 ( we don't have Anycast distibuted gateway otherwise you should see L2/L3 ARP Suppression)

Let's verify arp suppression entries in table. (as you can see suppression-cache table is empty because host didn't send any arp packet yet)

```
leaf-2# show ip arp suppression-cache vlan 64

Flags: + - Adjacencies synced via CFSoE
       L - Local Adjacency
       R - Remote Adjacency
       L2 - Learnt over L2 interface
       PS - Added via L2RIB, Peer Sync
       RO - Dervied from L2RIB Peer Sync Entry

Ip Address      Age      Mac Address    Vlan Physical-ifindex    Flags    Remote Vtep Addrs

```

Let send 1 icmp packet from leaf-2-host to leaf-1-host and capture arp packet on tcpdump on leaf-1-host

```
[root@leaf-2-host ~]# ping 10.64.0.100 -c 1
```

On host-1-leaf tcpdump we noticed 1 arp packet has been recived from leaf-2-host and it reply back with own mac address.

```
[root@leaf-1-host ~]# tcpdump -i eth0 -nn arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
00:19:42.678381 ARP, Request who-has 10.64.0.100 tell 10.64.0.200, length 46
00:19:42.678399 ARP, Reply 10.64.0.100 is-at 48:df:37:40:8c:04, length 28

```

Lets verify leaf-2 VTEP arp suppression-cache table. ( Now we can see both IP/MAC entires in arp suppression-cache table where L2 flags saying this is local host and R flags saying its located on remote Vtep Addrs 10.255.255.10, MP-BGP pushed IP/MAC entires to all VTEP so now each Vtep knows where these ip/macs located so it doesn't need to send BUM traffic again to discover them )

```
leaf-2# show ip arp suppression-cache vlan 64

Flags: + - Adjacencies synced via CFSoE
       L - Local Adjacency
       R - Remote Adjacency
       L2 - Learnt over L2 interface
       PS - Added via L2RIB, Peer Sync
       RO - Dervied from L2RIB Peer Sync Entry

Ip Address      Age      Mac Address    Vlan Physical-ifindex    Flags    Remote Vtep Addrs

10.64.0.200     00:05:38 8cdc.d4b1.3828   64 port-channel621     L2
10.64.0.100     00:05:38 48df.3740.8c04   64 (null)              R        10.255.255.10

```

Lets run our previous arp flood test again and see how arp-suppression reduce flood of arp, sending arp packet from leaf-2-host to leaf-1-host, sending 10 arp packet using for loop. Other side on leaf-1-host we will run tcpdump to capture packets to see how many arp packet we will receive. 

```
[root@leaf-2-host ~]# for qw in `seq 1 10`; do arp -d 10.64.0.100 ; ping 10.64.0.100 -c 1; done

```

On leaf-1-host tcpdump output. (no activity on leaf-1-host that means it doesn't received any arp flood) 

```
[root@leaf-1-host ~]# tcpdump -i eth0 -nn arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

```

Lets check arp-suppression statistics to check hit or miss cache. (In following output you can see line Suppressed: Total 10, Requests 0, Requests on L2 10, that indicate received 10 L2 request and all 10 got suppressed, Vtep didn't flood arp packet)

```
leaf-2# show ip arp suppression-cache statistics

ARP packet statistics for suppression-cache

Suppressed:
Total 10, Requests 0, Requests on L2 10, Gratuitous 0, Gratuitous on L2 0       <------ 10 request suppressed

Forwarded :
Total: 0
 L3 mode : 	Requests 0, Replies 0
		Request on core port 0, Reply on core port 0,
		Flood ARP Probe 0,
		Dropped 0
 L2 mode : 	Requests 0, Replies 0
		Request on core port 0, Reply on core port 0,
		Flood ARP Probe 0,
		Dropped 0

Received:
Total: 10
 L3 mode: 	Requests 0, Replies 0                       <------ you can see L3 distributed gateway counters here.
		Local Request 0, Local Responses 0
		Gratuitous 0, Dropped 0
 L2 mode : 	Requests 10, Replies 0                       <------ 10 L2 request recived
		Gratuitous 0, Dropped 0
ARP suppression-cache Local entry statistics
Adds 10, Deletes 0

```

Lets clear arp tables and try to send 10 more arp packet to see how does it behave. (This time it forwarded 4 arp packets out of 10 packets and 6 got suppressed )

```
leaf-2# show ip arp suppression-cache statistics

ARP packet statistics for suppression-cache

Suppressed:
Total 16, Requests 0, Requests on L2 16, Gratuitous 0, Gratuitous on L2 0     <---- This time 6 arp packet got suppressed.

Forwarded :
Total: 5
 L3 mode : 	Requests 0, Replies 0
		Request on core port 0, Reply on core port 0,
		Flood ARP Probe 0,
		Dropped 0
 L2 mode : 	Requests 4, Replies 1                                         <---- 4 arp packets floods out because it has no entries in cache
		Request on core port 4, Reply on core port 0,
		Flood ARP Probe 0,
		Dropped 0

Received:
Total: 31
 L3 mode: 	Requests 0, Replies 0
		Local Request 0, Local Responses 0
		Gratuitous 0, Dropped 0
 L2 mode : 	Requests 20, Replies 1
		Gratuitous 0, Dropped 0
ARP suppression-cache Local entry statistics
Adds 36, Deletes 2

```

Above test showing arp-suppression has clear advantage on large scale network to reduce overall BUM traffic. 
