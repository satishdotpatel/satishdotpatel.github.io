---
title: "BGP Remote Trigger Blackhole for DDoS"
layout: post
date: 2023-08-14
image: /assets/images/2023-08-14-bgp-remote-trigger-blackhole-for-ddos/blackhole1.png
headerImage: true
tag:
- BGP
- DDoS
- Blackhole
- RTBH
category: blog
blog: true
author: Satish Patel
description: "BGP Remote Trigger Blackhole for DDoS"

---

### BGP blackholing

Remotely Triggered Black Hole (RTBH) filtering is a self-managed feature that enables you to block unnecessary traffic before it enters your network. RTBH protects you from Distributed Denial of Service (DDoS) attacks.

BGP blackhole filtering is a routing technique used to drop unwanted traffic. Black holes are placed in the parts of a network where unwanted traffic should be dropped. For example, a customer can ask a provider to install black hole on its provider edge (PE) routers to prevent unwanted traffic from entering a customer’s network.

### Scope of LAB

In this lab, I am going to setup RTBH between ISP and Customer site. I will configure BGP community to place BGP Null router on specific ISP PE1 or PE2 routers. Customer has choice here to place null route to either device using BGP community. 

* To deploy null route on PE1 use BGP community 65000:777
* To deploy null route on PE2 use BGP community 65000:666
 

![<img>](/assets/images/2023-08-14-bgp-remote-trigger-blackhole-for-ddos/bgp-blackhole-network.png){: width="1000" }


### PE1 

Interface configuration

```
interface GigabitEthernet0/0
 description Link-to-PE2
 ip address 10.1.1.1 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 description Link-to-sw1
 ip address 200.200.200.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!     
```

BGP configuration

```
router bgp 65000
 bgp log-neighbor-changes
 network 200.200.200.0
 neighbor 10.1.1.2 remote-as 65000
 neighbor 10.1.1.2 send-community
 neighbor 10.1.1.2 route-map customer in
!         
```

Set Null route entry. This is special address which used to set bgp next-hop address of triggered bgp blackhole request. 

```
ip route 192.0.2.1 255.255.255.255 Null0
```

route-map to match community to install null route in RIB. Deny community in route-map if match to 65000:666

```
ip bgp-community new-format
ip community-list 10 permit 65000:777
ip community-list 20 permit 65000:666
!         
route-map customer deny 5
 match community 20
!         
route-map customer permit 10
 match community 10
 set ip next-hop 192.0.2.1
!         
route-map customer permit 20
!         
```


### PE2

Interface configuration

```
interface GigabitEthernet0/0
 description Link-to-PE1
 ip address 10.1.1.2 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 description Link-to-CE1
 ip address 10.0.0.1 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!         
```

BGP configuration

```
router bgp 65000
 bgp log-neighbor-changes
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 ebgp-multihop 255
 neighbor 10.0.0.2 route-map customer in
 neighbor 10.1.1.1 remote-as 65000
 neighbor 10.1.1.1 next-hop-self
 neighbor 10.1.1.1 send-community
!
```

Set Null route entry.

```
ip route 192.0.2.1 255.255.255.255 Null0
```

route-map configuration to match community. Install null route in RIB if community match 65000:6666 

```
ip bgp-community new-format
ip community-list 10 permit 65000:666
!
route-map customer permit 10
 match community 10
 set ip next-hop 192.0.2.1
!
route-map customer permit 20
!
```

### CE1 

Interface configurations

```
interface GigabitEthernet0/0
 description Link-to-sw0
 ip address 192.168.255.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!         
interface GigabitEthernet0/1
 description Link-to-ISP-PE2
 ip address 10.0.0.2 255.255.255.252
 ip flow ingress
 duplex auto
 speed auto
 media-type rj45
!         
```

BGP configurations 

```
router bgp 65001
 bgp log-neighbor-changes
 network 192.168.255.0
 redistribute static route-map RTBH
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 send-community
 neighbor 192.168.255.10 remote-as 65001
!
```

route-map configuration to send BGP community using route tag. 

```
route-map RTBH permit 10
 match tag 666
 set community 65000:666
!
route-map RTBH permit 20
 match tag 777
 set community 65000:777
!
route-map RTBH permit 30
```

### Validation 

Let's assuming attacker (200.200.200.200) sending DDoS to customer web server (192.168.255.120) and that hogging our internet pipe. Now we will trigger RTBH request to send null route for host 192.168.255.120 from CE1 router. It will send BGP update to install null route on PE1 or PE2 device based on matching community. 

```
CE1(config)#ip route 192.168.255.120 255.255.255.255 Null0 tag 666 
```

Check BGP route on PE2. As you can see entry of 192.168.255.120/32 and Next Hop is 192.0.2.1 which is pointing to Null0 that means attacker traffic will get dropped on PE2 device. 

```
PE2#show ip bgp

    Network          Next Hop            Metric LocPrf Weight Path
 *>   192.168.255.0    10.0.0.2                 0             0 65001 i
 *>   192.168.255.120/32
                       192.0.2.1                0             0 65001 ?
 *>i  200.200.200.0    10.1.1.1                 0    100      0 i
```

Check BGP route on PE1. As per our configuration it doesn't match BGP community so it's not going to install null route on PE1. 

```
PE1#show ip bgp

     Network          Next Hop            Metric LocPrf Weight Path
 *>i  192.168.255.0    10.1.1.2                 0    100      0 65001 i
 *>   200.200.200.0    0.0.0.0                  0         32768 i
```

Let's remove existing null route and replace tag from 666 to 777 and see what happen. 

```
CE1(config)#no ip route 192.168.255.120 255.255.255.255 Null0 tag 666
CE1(config)#ip route 192.168.255.120 255.255.255.255 Null0 tag 777
```

Check BGP route on PE2. If you notice Next Hop is 10.0.0.2 instead of 192.0.2.1 that means we are not dropping packet on PE2 and continute forwarding. 

```
PE2#show ip bgp

    Network          Next Hop            Metric LocPrf Weight Path
 *>   192.168.255.0    10.0.0.2                 0             0 65001 i
 *>   192.168.255.120/32
                       10.0.0.2                 0             0 65001 ?
 *>i  200.200.200.0    10.1.1.1                 0    100      0 i
```

Check BGP route on PE1. Voilla!!! Now you can see route get install in BGP table and Next Hop is 192.0.2.1 which is pointing to Null0. 

```
PE1#show ip bgp

    Network          Next Hop            Metric LocPrf Weight Path
 *>i  192.168.255.0    10.1.1.2                 0    100      0 65001 i
 *>i  192.168.255.120/32
                       192.0.2.1                0    100      0 65001 ?
 *>   200.200.200.0    0.0.0.0                  0         32768 i
```

Using BGP community you can create more controlled way to deploy RTBH on specific entry point of network to stop attacker traffic. In next Blog I will setup Netflow to send DDoS flow to FastNetMon/goBGP tool to detect DDoS and trigger BGP RTBH automatically without any human interaction. 

Enjoy!!! 







