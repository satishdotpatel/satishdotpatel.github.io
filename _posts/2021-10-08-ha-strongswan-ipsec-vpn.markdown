---
title: "High Availability Strongswan IPsec VPN"
layout: post
date: 2021-10-08
image: /assets/images/2021-10-08-ha-strongswan-ipsec-vpn/tunnel.png
headerImage: true
tag:
- Keepalived
- Strongswan
- High Availiability
- IPsec
- Cisco ASA
- Ubuntu
category: blog
blog: true
author: Satish Patel
description: "High Availability Strongswan IPsec VPN"

---

This blog post is about demonstrating how to setup Linux based strongswan IPsec vpn. In this lab work I have multiple sites which are connected to HQ sites. 

### Remote Sites Info

 
----------------------------------
#### HQ:
- VPN Device: Linux Server
- PublicIP - fw-1: 192.168.255.12
- PublicIP - fw-2: 192.168.255.13
- PublicIP - VIP: 192.168.255.11 
- LAN: 10.11.0.0/24

----------------------------------

#### Boston:
- VPN Device: Linux Server
- PublicIP: 192.168.255.22
- LAN: 10.22.0.0/24

----------------------------------

#### Phoenix:
- VPN Device: Cisco ASA
- PublicIP: 192.168.255.33
- LAN: 10.33.0.0/24

### Network Setup

----------------------------------

![<img>](/assets/images/2021-10-08-ha-strongswan-ipsec-vpn/vpn-network-setup.png){: width="800" }

----------------------------------

### HQ Site setup

Install keepalived & strongswan

```
$ apt-get install keepalived 
$ apt-get install strongswan
```

Configure keepalived on both firewall

fw-1

```
vrrp_sync_group G1 {
    group {
        EXT
        INT
    }
    notify "/usr/local/sbin/notify-ipsec.sh"
}

vrrp_instance INT {
    state MASTER
    interface ens3
    virtual_router_id 11
    priority 50
    advert_int 1
    unicast_src_ip 10.11.0.2
    unicast_peer {
        10.11.0.3
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.0.1/24 dev ens3
    }
    nopreempt
    garp_master_delay 1
}

vrrp_instance EXT {
    state MASTER
    interface ens2
    virtual_router_id 22
    priority 50
    advert_int 1
    unicast_src_ip 192.168.255.12
    unicast_peer {
        192.168.255.13
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.255.11/24 dev ens2
    }
    nopreempt
    garp_master_delay 1
}
```

fw-2

```
vrrp_sync_group G1 {
    group {
        EXT
        INT
    }
    notify "/usr/local/sbin/notify-ipsec.sh"
}

vrrp_instance INT {
    state BACKUP
    interface ens3
    virtual_router_id 11
    priority 25
    advert_int 1
    unicast_src_ip 10.11.0.3
    unicast_peer {
        10.11.0.2
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.11.0.1/24 dev ens3
    }
    nopreempt
    garp_master_delay 1
}

vrrp_instance EXT {
    state BACKUP
    interface ens2
    virtual_router_id 22
    priority 25
    advert_int 1
    unicast_src_ip 192.168.255.13
    unicast_peer {
        192.168.255.12
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.255.11/24 dev ens2
    }
    nopreempt
    garp_master_delay 1
}
```

Configure strongswan for IPsec VPN setup, both fw-1 and fw-2 should have identical files because they are in HA. On Ubuntu, you would modify these two files with configuration parameters to be used in the IPsec tunnel. You can use your favorite editor to edit them.

/etc/ipsec.secrets

```
# This file holds shared secrets or RSA private keys for authentication.

#  Source         Destination       Key
192.168.255.11 192.168.255.22 : PSK "MyKey123"
192.168.255.11 192.168.255.33 : PSK "MyKey123"
```

/etc/ipsec.conf

```
# basic configuration
config setup
        charondebug="all"
        uniqueids=yes
        strictcrlpolicy=no

# connection to remote strongswan 
conn hq-to-boston
        authby=secret
        left=%defaultroute
        leftid=192.168.255.11
        leftsubnet=10.11.0.1/24
        right=192.168.255.22
        rightsubnet=10.22.0.1/24
        ike=aes256-sha2_256-modp1024!
        esp=aes256-sha2_256!
        keyingtries=0
        ikelifetime=1h
        lifetime=8h
        dpddelay=30
        dpdtimeout=120
        dpdaction=restart
        auto=start

# connection to remote Cisco ASA
conn hq-to-phoenix
        authby=secret
        left=%defaultroute
        leftid=192.168.255.11
        leftsubnet=10.11.0.1/24
        right=192.168.255.33
        rightsubnet=10.33.0.1/24
        ike=aes256-sha1-modp1536
        esp=aes256-sha1
        leftauth=psk
        rightauth=psk
        keyexchange=ikev1
        keyingtries=%forever
        ikelifetime=1h
        lifetime=8h
        dpddelay=30
        dpdtimeout=120
        dpdaction=restart
        auto=start#
```

Create keepalived notify file to make sure ipsec service is stopped on BACKUP server.

/usr/local/sbin/notify-ipsec.sh

```
#!/bin/bash
TYPE=$1
NAME=$2
STATE=$3
case $STATE in
        "MASTER") /usr/sbin/ipsec start
                  ;;
        "BACKUP") /usr/sbin/ipsec stop
                  ;;
        "FAULT")  /usr/sbin/ipsec stop
                  exit 0
                  ;;
        *)        /usr/bin/logger "ipsec unknown state"
                  exit 1
                  ;;
esac
```

ipsec start/stop/status service using following commands

```
$ ipsec start
Starting strongSwan 5.6.2 IPsec [starter]...
```

### Boston site

Install strongswan 

```
$ apt-get install strongswan
```

Configure strongswan ipsec to connect HQ site

/etc/ipsec.secrets 

```
# This file holds shared secrets or RSA private keys for authentication.

#  Source	   Destination        Key
192.168.255.22 192.168.255.11 : PSK "MyKey123"
```

/etc/ipsec.conf

```
# basic configuration
config setup
        charondebug="all"
        uniqueids=yes
        strictcrlpolicy=no

# connection to amsterdam datacenter
conn paris-to-amsterdam
        authby=secret
        left=%defaultroute
        leftid=192.168.255.22
        leftsubnet=10.22.0.1/24
        right=192.168.255.11
        rightsubnet=10.11.0.1/24
        ike=aes256-sha2_256-modp1024!
        esp=aes256-sha2_256!
        keyingtries=0
        ikelifetime=1h
        lifetime=8h
        dpddelay=30
        dpdtimeout=120
        dpdaction=restart
        auto=start
```

Start service

```
$ ipsec start
Starting strongSwan 5.6.2 IPsec [starter]...
```

Check status, As you can see TUNNEL has been ESTABLISHED

```
$ ipsec status
Security Associations (1 up, 0 connecting):
paris-to-amsterdam[1]: ESTABLISHED 6 seconds ago, 192.168.255.22[192.168.255.22]...192.168.255.11[192.168.255.11]
paris-to-amsterdam{1}:  INSTALLED, TUNNEL, reqid 1, ESP SPIs: c5c8e56b_i c81186ce_o
paris-to-amsterdam{1}:   10.22.0.0/24 === 10.11.0.0/24
```


### Phoenix Site 

Here we have Cisco ASA firewall and here is the config snippet.


```
!
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.168.255.33 255.255.255.0 
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.33.0.1 255.255.255.0 
!
object-group network local-network
 network-object 10.33.0.0 255.255.255.0
object-group network remote-network
 network-object 10.11.0.0 255.255.255.0
!
access-list asa-strongswan-vpn extended permit ip object-group local-network object-group remote-network 
!
route outside 0.0.0.0 0.0.0.0 192.168.255.1 1
!
crypto ipsec ikev1 transform-set TRANSFORM-SS-TEST esp-aes-256 esp-sha-hmac 
!
crypto map outside_map 10 match address asa-strongswan-vpn
crypto map outside_map 10 set peer 192.168.255.11 
crypto map outside_map 10 set ikev1 transform-set TRANSFORM-SS-TEST
crypto map outside_map 10 set security-association lifetime seconds 28800
crypto map outside_map interface outside
!
crypto isakmp identity address 
crypto ikev1 enable outside
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 5
 lifetime 3600
!
tunnel-group 192.168.255.11 type ipsec-l2l
tunnel-group 192.168.255.11 ipsec-attributes
 ikev1 pre-shared-key *****
!
nat (inside,outside) source static local-network local-network destination static remote-network remote-network no-proxy-arp route-lookup
!
```

Verify by pinging from host 10.33.0.10 to 10.11.0.10. As soon as you ping it will generate interesting traffic which will trigger IPsec tunnel

```
# show crypto isakmp sa

IKEv1 SAs:

   Active SA: 1
    Rekey SA: 0 (A tunnel will report 1 Active and 1 Rekey SA during rekey)
Total IKE SA: 1

1   IKE Peer: 192.168.255.11
    Type    : L2L             Role    : initiator 
    Rekey   : no              State   : MM_ACTIVE 

There are no IKEv2 SAs
```

Verify IPsec peer

```
# show crypto ipsec sa
interface: outside
    Crypto map tag: outside_map, seq num: 10, local addr: 192.168.255.33

      access-list asa-strongswan-vpn extended permit ip 10.33.0.0 255.255.255.0 10.11.0.0 255.255.255.0 
      local ident (addr/mask/prot/port): (10.33.0.0/255.255.255.0/0/0)
      remote ident (addr/mask/prot/port): (10.11.0.0/255.255.255.0/0/0)
      current_peer: 192.168.255.11


      #pkts encaps: 4, #pkts encrypt: 4, #pkts digest: 4
      #pkts decaps: 4, #pkts decrypt: 4, #pkts verify: 4
      #pkts compressed: 0, #pkts decompressed: 0
      #pkts not compressed: 4, #pkts comp failed: 0, #pkts decomp failed: 0
      #pre-frag successes: 0, #pre-frag failures: 0, #fragments created: 0
      #PMTUs sent: 0, #PMTUs rcvd: 0, #decapsulated frgs needing reassembly: 0
      #TFC rcvd: 0, #TFC sent: 0
      #Valid ICMP Errors rcvd: 0, #Invalid ICMP Errors rcvd: 0
      #send errors: 0, #recv errors: 0
```


### Keepalived failover test 

Continues ping from 10.33.0.10 to 10.11.0.10 and while ping is going on stop keepalived service on fw-1 (MASTER) node, and you will see your floating ip will move to fw-2 and notify script will trigger ipsec service to start and that will re-established VPN tunnel again. you may notice 15 to 30 second downtime but as soon as it reconnect and ping will conntinue start.




```



