---
title: "Mitigate DDoS using FastNetMon and goBGP"
layout: post
date: 2023-08-29
image: /assets/images/2023-08-29-mitigate-ddos-using-fastnetmon-and-gobgp/ddos-bombs.png
headerImage: true
tag:
- BGP
- DDoS
- Blackhole
- FastNetMon
- goBGP
category: blog
blog: true
author: Satish Patel
description: "Mitigate DDoS using FastNetMon and goBGP"

---

In previous [BGP RTBH Setup](https://satishdotpatel.github.io/bgp-remote-trigger-blackhole-for-ddos/) post, We have setup LAB for null route DDoS traffic using BGP blackhole method. We test null route injecting static route in BGP which is manual method but that is not practical in production environments. In this post we are going to expand same lab to do auto null route traffic using FastNetMon and goBGP tools.  

### Scope of LAB

We are going to configure NetFlow on the edge router to feed flow to fastnetmon and based on threshold it will trigger a goBGP program to inject the /32 route to iBGP peer with the edge router to send a null route community string. 

* FastnetMon is Linux server running Ubuntu 22.04

![<img>](/assets/images/2023-08-29-mitigate-ddos-using-fastnetmon-and-gobgp/fastnetmon-gobgp.png){: width="1000" }


### Install FastNetmon 

FastNetMon is awesome tool to detect DDoS in 2 second and take action according your need. In our case I am sending bgp null route to drop target IP to protect our infrastucture. 

FastNetMon comes with goBGP program which is written in GO language which support all BGP functinality. I will create iBGP peer with my router to send bgp route and community string to upstream ISP to drop targeted IP. 

You need to install wget tool before starting this tool. Just follow official website for help: https://fastnetmon.com/install/ 

```
$ wget https://install.fastnetmon.com/installer -Oinstaller
$ sudo chmod +x installer
$ sudo ./installer -install_community_edition
```

Create goBGP systemd file to start program /etc/systemd/system/multi-user.target.wants/gobgpd.service

```
[Unit]
Description=GoBGP Routing Daemon
Documentation=file:/usr/share/doc/gobgpd/getting-started.md
After=network.target syslog.service
ConditionPathExists=/etc/gobgpd.conf

[Service]
Type=notify
ExecStartPre=/opt/fastnetmon-community/libraries/gobgp_3_12_0/gobgpd -f /etc/gobgpd.conf -d
ExecStart=/opt/fastnetmon-community/libraries/gobgp_3_12_0/gobgpd -f /etc/gobgpd.conf --sdnotify --disable-stdlog --syslog yes $GOBGPD_OPTIONS
ExecReload=/opt/fastnetmon-community/libraries/gobgp_3_12_0/gobgpd -r
DynamicUser=yes
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Reload systemd and enable gobgpd services

```
$ systemctl daemon-reload
$ systemctl enable gobgpd.service
```

Create symlink for gobgp cli command to /usr/local/bin 

```
$ ln -s /opt/fastnetmon-community/libraries/gobgp_3_12_0/gobgp /usr/local/bin/.
```

### Configure goBGP to create iBGP peer to CE1

/etc/gobgpd.conf

```
[global.config]
    as = 65001
    router-id = "192.168.255.10"
 
[[neighbors]]
  [neighbors.config]
      neighbor-address = "192.168.255.1"
      peer-as = 65001
```

Restart goBGP service

```
$ systemctl start gobgpd.service
```

### Configure CE1 router to create peer to goBGP

```
router bgp 65001
 bgp log-neighbor-changes
 network 192.168.255.0
 redistribute static route-map RTBH
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 send-community
 neighbor 192.168.255.10 description PEER-TO-goBGP
 neighbor 192.168.255.10 remote-as 65001
```

Verify BGP peer on CE1 

```
CE1#show ip bgp summary

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.0.1        4        65000   24292   24329       31    0    0 2w1d            1
192.168.255.10  4        65001    2886    3150       31    0    0 1d00h           0
```

Verify bgp peer status on goBGP application

```
$ gobgp neighbor
Peer             AS     Up/Down State       |#Received  Accepted
192.168.255.1 65001 1d 00:03:40 Establ      |        2         2
```

You can inject /32 route manually using following command to test your null route functinality. 

```
$ gobgp global rib add -a ipv4 200.200.200.202/32 community 65000:666

$ gobgp global rib del -a ipv4 200.200.200.202/32 community 65000:666
```

## Configure FastNetMon to trigger goBGP 

### First configure CE1 router to send NetFlow

```
ip flow-export source GigabitEthernet0/0
ip flow-export version 9
ip flow-export template timeout-rate 1
ip flow-export template refresh-rate 10
ip flow-export destination 192.168.255.10 2055
!
ip flow-cache timeout active 1
```

### Configure FastNetMon 

File has lots of options so follow official website for all the options. 

/etc/fastnetmon.conf 

```
enable_ban = on
ban_time = 60
networks_list_path = /etc/networks_list
white_list_path = /etc/networks_whitelist
ban_for_pps = on
threshold_pps = 3       # I have setup PPS rate low so I can quickly generate DDoS

netflow = on
interfaces = eth3,eth4
netflow_port = 2055
netflow_host = 0.0.0.0
netflow_sampling_ratio = 1

gobgp = on
gobgp_next_hop = 0.0.0.0
gobgp_announce_host = on
gobgp_announce_whole_subnet = off
gobgp_community_host = 65000:666
gobgp_community_subnet = 65000:777
```

Add network in /etc/networks_list file you want to activate for FastNetMon to monitor for DDoS

```
$ cat /etc/networks_list
192.168.255.0/24
```

Restart fastnetmon daemon 

```
$ systemctl start fastnetmon
```

### Test configuration 

From Attacker-PC I am using hping3 tool to generate some traffic to simulate DDoS 

```
Attacker-PC# hping3 -c 200 --fast -p 80 192.168.255.120
HPING 192.168.255.120 (ens2 192.168.255.120): NO FLAGS are set, 40 headers + 0 data bytes
len=40 ip=192.168.255.120 ttl=61 DF id=0 sport=80 flags=RA seq=0 win=0 rtt=11.1 ms
len=40 ip=192.168.255.120 ttl=61 DF id=0 sport=80 flags=RA seq=1 win=0 rtt=6.9 ms
len=40 ip=192.168.255.120 ttl=61 DF id=0 sport=80 flags=RA seq=2 win=0 rtt=10.7 ms
len=40 ip=192.168.255.120 ttl=61 DF id=0 sport=80 flags=RA seq=3 win=0 rtt=6.5 ms
```

In a few seconds you will see 192.168.255.120/32 route in goBGP sending community 65000:666 to send null route to upstream router.

```
$ gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 192.168.255.0/24     192.168.255.1                             00:01:46   [{Origin: i} {Med: 0} {LocalPref: 100}]
*> 192.168.255.120/32   0.0.0.0                                   00:00:10   [{Origin: i} {Communities: 65000:666}]
*> 200.200.200.0/24     10.0.0.1             65000                00:01:46   [{Origin: i} {Med: 0} {LocalPref: 100}]
```

In 60 seconds you will see the route will get removed based on specified timeout in fastnetmon.

Enjoy!!! 



















