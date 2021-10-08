---
title: "High Availability with Keepalived and conntrackd"
layout: post
date: 2021-10-07
image: /assets/images/2021-10-07-ha-with-keepalived-and-conntrackd/high-availability.png
headerImage: true
tag:
- Keepalived
- Conntrackd
- High Availiability
category: blog
blog: true
author: Satish Patel
description: "High Availability with Keepalived and conntrackd"

---

This post if about building High Availibility firewall using keepalived and conntrackd service which will provide connection mirroring because some application are connection sensitive which may break connection during failover if connection state not replicated to standby server. 

For this POC I'm using Cisco Modeling Labs simulator to design and simulate my network.  

### Network Setup

![<img>](/assets/images/2021-10-07-ha-with-keepalived-and-conntrackd/ha-network-setup.png){: width="500" }

### Install conntrackd 

```
$ apt-get install conntrackd
```

Primary fw-1 /etc/conntrackd/conntrackd.conf

```
Sync {
    Mode FTFW {
        DisableExternalCache Off
        CommitTimeout 180
        PurgeTimeout 5
    }

    UDP {
        # Dedicated link for connection replication
        IPv4_address 172.30.16.1
        IPv4_Destination_Address 172.30.16.2
        Port 3780
        Interface ens3
        SndSocketBuffer 1249280
        RcvSocketBuffer 1249280
        Checksum on
    }
}

General {
    Systemd on
    Nice -20
    HashSize 32768
    HashLimit 131072
    LogFile on
    Syslog on
    NetlinkOverrunResync 5
    NetlinkEventsReliable on
    PollSecs 5
    EventIterationLimit 200
    LockFile /var/lock/conntrack.lock
    UNIX {
        Path /var/run/conntrackd.ctl
        Backlog 20
    }
    NetlinkBufferSize 2097152
    NetlinkBufferSizeMaxGrowth 8388608
    Filter From Userspace {
        Protocol Accept {
            TCP
            UDP
            ICMP # This requires a Linux kernel >= 2.6.31
        }
        Address Ignore {
            IPv4_address 127.0.0.1 # loopback
            IPv4_address 10.0.0.1
            IPv4_address 10.0.0.2
            IPv4_address 10.0.0.3
            IPv4_address 192.168.255.2
            IPv4_address 192.168.255.52
            IPv4_address 192.168.255.250
        }
    }
}
```

Standby fw-2 /etc/conntrackd/conntrackd.conf

```
Sync {
    Mode FTFW {
        DisableExternalCache Off
        CommitTimeout 180
        PurgeTimeout 5
    }

    UDP {
        # Dedicated link for connection replication
        IPv4_address 172.30.16.2
        IPv4_Destination_Address 172.30.16.1
        Port 3780
        Interface ens3
        SndSocketBuffer 1249280
        RcvSocketBuffer 1249280
        Checksum on
    }
}

General {
    Systemd on
    Nice -10
    HashSize 32768
    HashLimit 131072
    LogFile on
    Syslog on
    NetlinkOverrunResync 5
    NetlinkEventsReliable on
    PollSecs 5
    EventIterationLimit 200
    LockFile /var/lock/conntrack.lock
    UNIX {
        Path /var/run/conntrackd.ctl
        Backlog 20
    }
    NetlinkBufferSize 2097152
    NetlinkBufferSizeMaxGrowth 8388608
    Filter From Userspace {
        Protocol Accept {
            TCP
            UDP
            ICMP # This requires a Linux kernel >= 2.6.31
        }
        Address Ignore {
            IPv4_address 127.0.0.1 # loopback
            IPv4_address 10.0.0.1
            IPv4_address 10.0.0.2
            IPv4_address 10.0.0.3
            IPv4_address 192.168.255.2
            IPv4_address 192.168.255.52
            IPv4_address 192.168.255.250
        }
    }
}
```

Copy primary-backup.sh script in /etc/conntrackd directory for keepalived on both servers.

```
$ cp /usr/share/doc/conntrackd/examples/sync/primary-backup.sh /etc/conntrackd/.
$ chmod 755 /etc/conntrackd/primary-backup.sh
```

Start and Enable service

```
$ systemd enable conntrackd
$ systemd start conntrackd
```

### Install Keepalived

```
$ apt-get install keepalived
```

Primary fw-1 keepalived configuration file. 

```
vrrp_sync_group G1 {
    group {
        EXT
        INT
    }
    notify_master "/etc/conntrackd/primary-backup.sh primary"
    notify_backup "/etc/conntrackd/primary-backup.sh backup"
    notify_fault "/etc/conntrackd/primary-backup.sh fault"
}

vrrp_instance INT {
    state MASTER
    interface ens4
    virtual_router_id 11
    priority 50
    advert_int 1
    unicast_src_ip 10.0.0.1
    unicast_peer {
        10.0.0.2
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3/24 dev ens4
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
    unicast_src_ip 192.168.255.11
    unicast_peer {
        192.168.255.22
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.255.250/24 dev ens2
    }
    nopreempt
    garp_master_delay 1
}
```

Standby fw-2 Keepalived configuration file.

```
vrrp_sync_group G1 {
    group {
        EXT
        INT
    }
    notify_master "/etc/conntrackd/primary-backup.sh primary"
    notify_backup "/etc/conntrackd/primary-backup.sh backup"
    notify_fault "/etc/conntrackd/primary-backup.sh fault"
}

vrrp_instance INT {
    state BACKUP
    interface ens4
    virtual_router_id 11
    priority 25
    advert_int 1
    unicast_src_ip 10.0.0.2
    unicast_peer {
        10.0.0.1
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.3/24 dev ens4
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
    unicast_src_ip 192.168.255.22
    unicast_peer {
        192.168.255.11
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.255.250/24 dev ens2
    }
    nopreempt
    garp_master_delay 1
}
```

Start and Enable service

```
$ systemd enable keepalivd
$ systemd start keepalived
```

### Verify Keepalived

If all good then you can see vip addresses on primary server

```
root@fw-1:~# ip -4 addr list ens2 | grep inet
    inet 192.168.255.11/24 brd 192.168.255.255 scope global ens2
    inet 192.168.255.250/24 scope global secondary ens2
root@fw-1:~# ip -4 addr list ens4 | grep inet
    inet 10.0.0.1/24 brd 10.0.0.255 scope global ens4
    inet 10.0.0.3/24 scope global secondary ens4
```

### Verify conntrackd 

conntrackd won't work correctly until you configure "well-formed ruleset", That means you need to configure iptables rules with connection tracking enabled, I am configuring some basic rules for example here. SNAT rule for internet access for LAN users.

```
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A FORWARD -m state --state RELATED -j ACCEPT
-A FORWARD -i ens2 -m state --state ESTABLISHED -j ACCEPT
-A FORWARD -i ens4 -p tcp -m tcp --dport 22 --tcp-flags FIN,SYN,RST,ACK SYN -m state --state NEW -j ACCEPT
-A FORWARD -i ens4 -p tcp -m state --state ESTABLISHED -j ACCEPT
-A FORWARD -m state --state INVALID -j LOG --log-prefix "invalid: "
-A POSTROUTING -s 10.0.0.0/24 -o ens2 -j SNAT --to-source 192.168.255.250
```

If all well then you can see statistics using following command.

```
root@fw-1:~# conntrackd -s
cache internal:
current active connections:                4
connections created:                      32    failed:            0
connections updated:                   34335    failed:            0
connections destroyed:                    28    failed:            0

cache external:
current active connections:                6
connections created:                      48    failed:            0
connections updated:                   21721    failed:            0
connections destroyed:                    42    failed:            0

traffic processed:
                   0 Bytes                         0 Pckts

UDP traffic (active device=ens3):
             2550092 Bytes sent              1597636 Bytes recv
               35557 Pckts sent                35619 Pckts recv
                   0 Error send                    0 Error recv

message tracking:
                   0 Malformed msgs                    0 Lost msgs
```

### Test connection replication/mirroring

I have LAN ip 10.0.0.10 which i will use to ssh 192.168.255.33 and then i will perform keepalived failover to see my ssh connection still active or not. 

Lets check fw-1 conntrackd internal cache after ssh'ing 

```
root@fw-1:~# conntrackd -i 
udp      17 src=0.0.0.0 dst=255.255.255.255 sport=68 dport=67 [UNREPLIED] src=255.255.255.255 dst=0.0.0.0 sport=67 dport=68 mark=0 [active since 32395s]
udp      17 src=172.30.16.1 dst=172.30.16.2 sport=55473 dport=3780 [UNREPLIED] src=172.30.16.2 dst=172.30.16.1 sport=3780 dport=55473 mark=0 [active since 32395s]
udp      17 src=172.30.16.2 dst=172.30.16.1 sport=50651 dport=3780 [UNREPLIED] src=172.30.16.1 dst=172.30.16.2 sport=3780 dport=50651 mark=0 [active since 32395s]
tcp      6 ESTABLISHED src=10.0.0.10 dst=192.168.255.33 sport=48070 dport=22 src=192.168.255.33 dst=192.168.255.250 sport=22 dport=48070 [ASSURED] mark=0 [active since 46s]
``` 

Lets check fw-2 internal cache, if you have noticed it doesn't have any connection info of SSH

```
root@fw-2:~# conntrackd -i
udp      17 src=0.0.0.0 dst=255.255.255.255 sport=68 dport=67 [UNREPLIED] src=255.255.255.255 dst=0.0.0.0 sport=67 dport=68 mark=0 [active since 32788s]
udp      17 src=172.30.16.1 dst=172.30.16.2 sport=55473 dport=3780 [UNREPLIED] src=172.30.16.2 dst=172.30.16.1 sport=3780 dport=55473 mark=0 [active since 32798s]
udp      17 src=172.30.16.2 dst=172.30.16.1 sport=50651 dport=3780 [UNREPLIED] src=172.30.16.1 dst=172.30.16.2 sport=3780 dport=50651 mark=0 [active since 32798s]
```

Lets check fw-2 conntrackd external cache, As you can see connection information got replicated and sitting in external cache and as soon as failover trigger it will go to internal cache.

```
root@fw-2:~# conntrackd -e
udp      17 src=0.0.0.0 dst=255.255.255.255 sport=68 dport=67 [UNREPLIED] mark=0 [active since 32533s]
udp      17 src=172.30.16.1 dst=172.30.16.2 sport=55473 dport=3780 [UNREPLIED] mark=0 [active since 32533s]
udp      17 src=172.30.16.2 dst=172.30.16.1 sport=50651 dport=3780 [UNREPLIED] mark=0 [active since 32533s]
tcp      6 ESTABLISHED src=10.0.0.10 dst=192.168.255.33 sport=48070 dport=22 [ASSURED] mark=0 [active since 185s]
```

Lets perform failover 

```
root@fw-1:~# systemd stop keepalived
```

Now check fw-2 internal cache again 

```
root@fw-2:~# conntrackd -i
udp      17 src=0.0.0.0 dst=255.255.255.255 sport=68 dport=67 [UNREPLIED] src=255.255.255.255 dst=0.0.0.0 sport=67 dport=68 mark=0 [active since 5s]
udp      17 src=172.30.16.1 dst=172.30.16.2 sport=55473 dport=3780 [UNREPLIED] src=172.30.16.2 dst=172.30.16.1 sport=3780 dport=55473 mark=0 [active since 5s]
udp      17 src=172.30.16.2 dst=172.30.16.1 sport=50651 dport=3780 [UNREPLIED] src=172.30.16.1 dst=172.30.16.2 sport=3780 dport=50651 mark=0 [active since 5s]
tcp      6 ESTABLISHED src=10.0.0.10 dst=192.168.255.33 sport=48070 dport=22 src=192.168.255.33 dst=192.168.255.250 sport=22 dport=48070 [ASSURED] mark=0 [active since 5s]
```

voila!!! if you noticed you ssh connection still alive.

Reference Doc: https://conntrack-tools.netfilter.org/manual.html
