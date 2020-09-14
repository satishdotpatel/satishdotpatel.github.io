---
title: "Cisco Spanning-tree fun troublshooting"
layout: post
date: 2020-09-13
image: /assets/images/2020-09-04-openstack-senlin-autoscaling/senlin-autoscale.png
headerImage: true
tag:
- cisco
- nexus
- spanning-tree
category: blog
blog: true
author: Satish Patel
description: "Cisco Spanning-tree fun troublshooting"

---

Last week noticed periodic packet drops across racks. on monitoring noticed that packet drops happening every 1 minute and 30 second.

```
    111113641111 11 111 11 11 1  11   1   1 11  1 11     11 12
    011245362136900971084282051974067736846931972921847680082588
100
 90
 80
 70
 60       #
 50       ##
 40      ###
 30      ###                                                 #
 20      ###   #    #                     #                  #
 10 ##################################### ########### ##########
    0....5....1....1....2....2....3....3....4....4....5....5....
              0    5    0    5    0    5    0    5    0    5
 
               CPU% per second (last 60 seconds)
                      # = average CPU%
```

You can see ping drop during  high CPU spike on switch.. 

```
C:\Users\spatel>ping 61.90.64.13 -t
 
Pinging 61.90.64.13 with 32 bytes of data:
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Request timed out.
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
Reply from 61.90.64.13: bytes=32 time=9ms TTL=53
```

# Investigation
During investigation found in CoPP (Control Plan Policing) dropping lots of glean & arp packets and that number in millions which is no way normal, glean i directly related to arp flooding which gave me clue to look into arp tables. 

```
# show policy-map interface control-plane
...
...
class-map copp-s-glean (match-any)
      police pps 500
        OutPackets    3371
        DropPackets   19911477
```

# Investigating Arp 

Found arp table getting flushed out every ~85 seconds for all hosts in that connected subnets and that is **NOT** normal, you can see that in Age column in following output all hosts has same arp age time because full tables got flushed and this is happened every ~85th second around.

```
# sh ip arp
 
Flags: * - Adjacencies learnt on non-active FHRP router
       + - Adjacencies synced via CFSoE
       # - Adjacencies Throttled for Glean
       CP - Added via L2RIB, Control plane Adjacencies       D - Static Adjacencies attached to down interface
 
IP ARP Table for context default
Total number of entries: 155
Address         Age       MAC Address     Interface
62.250.131.249  00:07:22  ec13.dbc0.3d45  Vlan3
61.90.64.3      00:00:27  INCOMPLETE      Vlan202
61.90.64.4      00:00:27  INCOMPLETE      Vlan202
61.90.64.5      00:00:28  INCOMPLETE      Vlan202
61.90.64.10     00:00:02  INCOMPLETE      Vlan202
61.90.64.11     00:00:05  1060.4bac.ad60  Vlan202
61.90.64.12     00:00:05  1060.4bb3.fd60  Vlan202
61.90.64.13     00:00:05  1060.4baa.0100  Vlan202
61.90.64.14     00:00:05  6c3b.e5b0.7a70  Vlan202
61.90.64.15     00:00:05  6c3b.e5a9.f670  Vlan202
61.90.64.16     00:00:05  6c3b.e5bd.c320  Vlan202
61.90.64.17     00:00:05  6c3b.e5bd.c1c8  Vlan202
61.90.64.18     00:00:05  1060.4bb3.3f18  Vlan202
61.90.64.19     00:00:05  6c3b.e5aa.d2f0  Vlan202
...
...
```

# Investigating Spanning Tree Protocol (STP)

Lets find out recent changes in STP, you can see in output just recently something changed on Interface Ethernet1/36 (Minutes ago it detected TCM)

```
# show spanning-tree detail | inc ieee|occurr|from
  Number of topology changes 4 last change occurred 3287:50:33 ago
          from port-channel1
  Number of topology changes 139 last change occurred 141:18:14 ago
          from Ethernet1/47
  Number of topology changes 139 last change occurred 309:32:43 ago
          from Ethernet1/47
  Number of topology changes 5867 last change occurred 260:38:12 ago
          from Ethernet1/47
  Number of topology changes 154 last change occurred 309:32:42 ago
          from Ethernet1/47
  Number of topology changes 118639 last change occurred 0:01:06 ago
          from Ethernet1/36
  Number of topology changes 124315 last change occurred 0:01:06 ago
          from Ethernet1/36
  Number of topology changes 137 last change occurred 309:32:42 ago
          from Ethernet1/47
```

what is connected to Interface Ethernet 1/36 ?

# sh cdp neighbors interface e1/36
Capability Codes: R - Router, T - Trans-Bridge, B - Source-Route-Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater,
                  V - VoIP-Phone, D - Remotely-Managed-Device,
                  s - Supports-STP-Dispute
 
Device-ID          Local Intrfce  Hldtme Capability  Platform      Port ID
swt-n3k.foo.com(FOC1703R0EE)
                    Eth1/36        170    R S I s   N3K-C3064PQ-1 Eth1/1
 
Total entries displayed: 1
```

Lets see who is triggering spanning tree convergence on switch (swt-n3k.foo.com), Interesting something just changed at interface e1/24  (now i am very close), look like e1/24 detected TCM...

```
swt-n3k# show spanning-tree detail | inc ieee|occurr|from
  Number of topology changes 5744 last change occurred 260:40:25 ago
          from Ethernet1/1
  Number of topology changes 221898 last change occurred 0:00:38 ago
          from Ethernet1/24
  Number of topology changes 221905 last change occurred 0:00:38 ago
          from Ethernet1/24
  Number of topology changes 49 last change occurred 309:34:56 ago
          from Ethernet1/1
```

This is the configuration on port e1/24 and no description so hard to know what server is connected, observium showing interface is down and sending small amount of traffic so definitely its not something in production. 

```
interface Ethernet1/24
  switchport mode trunk
  switchport trunk allowed vlan 202,3008
  spanning-tree port type edge
  spanning-tree guard root
```

Lets shutdown e1/24 port and see.

```
swt-n3k(config)# int e1/24
swt-n3k(config)# shutdown
```

Lets check spanning tree convergence again to see anything changed in recently in spanning tree. ( last 1 hour nothing changed bingo!!!! )

```
# show spanning-tree detail | inc ieee|occurr|from
  Number of topology changes 4 last change occurred 3289:17:28 ago
          from port-channel1
  Number of topology changes 139 last change occurred 142:45:09 ago
          from Ethernet1/47
  Number of topology changes 139 last change occurred 310:59:38 ago
          from Ethernet1/47
  Number of topology changes 5867 last change occurred 262:05:07 ago
          from Ethernet1/47
  Number of topology changes 154 last change occurred 310:59:37 ago
          from Ethernet1/47
  Number of topology changes 118646 last change occurred 1:18:55 ago
          from Ethernet1/36
  Number of topology changes 124322 last change occurred 1:18:55 ago
          from Ethernet1/36
  Number of topology changes 137 last change occurred 310:59:38 ago
          from Ethernet1/47
```

Lets check CoPP packet drops  ( in last 1 hour only 451 drop bingo!!!!) 

```
class-map copp-s-glean (match-any)
      police pps 500
        OutPackets    51284
        DropPackets   451
```

