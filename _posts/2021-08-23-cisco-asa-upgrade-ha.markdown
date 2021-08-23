---
title: "Upgrade Cisco ASA Active-Standby Mode"
layout: post
date: 2021-08-23
image: /assets/images/2021-08-23-cisco-asa-upgrade-ha/cisco-asa.png
headerImage: true
tag:
- Cisco
- ASA
- Firewall
category: blog
blog: true
author: Satish Patel
description: "Upgrade Cisco ASA Active-Standby Mode"

---

When you have two Cisco ASA fireall deployed in Active/Standby failover configuration, As you already have a high availability solution you do not want any downtime. 


### What Primary/Secondary Vs Active/Standby 

The Primary and Secondary firwalls are physical firewalls roles. Primary will always primary and Secondary will always be the secondary, which you configure during failover configuration.
The Active firwall will be firwall that's passing traffic and in operation, and the Standby firwall is sat waiting to take over, each physical firewall can be either active or standby. 

Following output telling you this firewall is Primary (pri) but currently standby (stby).

```
pri/stby/asa-1# show run | grep (primary|secondary)
failover lan unit primary
```

Following output telling you this firewall is Secondary (sec) but currently active (act) and serving live traffic.

```
sec/act/asa-1# show run | grep (primary|secondary)
failover lan unit secondary
```

![<img>](/assets/images/2021-08-23-cisco-asa-upgrade-ha/asa-diagram.png){: width="850" }


### Let copy new cisco asa image on both primary and secondary devices 

asa-1 (active)

```
pri/act/asa-1# copy tftp: disk0:
Address or name of remote host [192.168.255.11]? 
Source filename [asa634-45-smp-k8.bin]? asa964-45-smp-k8.bin
Destination filename [asa964-45-smp-k8.bin]? 
Accessing tftp://192.168.255.11/asa964-45-smp-k8.bin...!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```

asa-2 (standby)

```
sec/stby/asa-1# copy tftp: disk0:
Address or name of remote host [192.168.255.11]? 
Source filename [asa634-45-smp-k8.bin]? asa964-45-smp-k8.bin
Destination filename [asa964-45-smp-k8.bin]? 
Accessing tftp://192.168.255.11/asa964-45-smp-k8.bin...!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```

Verify

```
pri/act/asa-1# dir

Directory of disk0:/

7      -rwx  0            13:56:32 Jun 27 2019  use_ttyS0
16     drwx  4096         02:55:20 Aug 20 2021  smart-log
8      drwx  4096         03:34:22 Aug 23 2021  log
11     drwx  4096         02:55:24 Aug 20 2021  coredumpinfo
81     -rwx  82796544     03:01:40 Aug 20 2021  asa964-36-smp-k8.bin
82     drwx  4096         03:34:50 Aug 23 2021  tmp
83     -rwx  82814976     13:15:36 Aug 23 2021  asa964-45-smp-k8.bin
```

### Set boot system image 

Current running version is 9.6(4).36

```
pri/act/asa-1# show version 

Cisco Adaptive Security Appliance Software Version 9.6(4)36 
Device Manager Version 7.12(2)

Compiled on Thu 07-Nov-19 13:37 PST by builders
System image file is "(hd0,1)/asa964-36-smp-k8.bin"
Config file at boot was "startup-config"
```

Show bootvar 

```
pri/act/asa-1# show bootvar 

BOOT variable = (hd0,1)/asa964-36-smp-k8.bin
Current BOOT variable = disk0:/asa964-36-smp-k8.bin
CONFIG_FILE variable = 
Current CONFIG_FILE variable = 
```

Set new image for next boot 

```
pri/act/asa-1# conf t
pri/act/asa-1(config)#boot system disk0:/asa964-45-smp-k8.bin
```

Remove older image from boot and save config

```
pri/act/asa-1(config)# no boot system disk0:/asa964-36-smp-k8.bin
pri/act/asa-1(config)# wr
Building configuration...
Cryptochecksum: ce770b8f 9a7ce38e fc682db4 81d2bf45 

8009 bytes copied in 0.290 secs
[OK]
```

Verify 

```
pri/act/asa-1(config)# show bootvar 

BOOT variable = (hd0,1)/asa964-45-smp-k8.bin
Current BOOT variable = disk0:/asa964-45-smp-k8.bin
CONFIG_FILE variable = 
Current CONFIG_FILE variable = 
```


### Reload Standby device ( Whilst still on the primary active firewall)

```
pri/act/asa-1(config)# failover reload-standby 
```

After above command you will see following output on standby device and it will start reloading.

```
sec/stby/asa-1# 


***
*** --- SHUTDOWN NOW ---
***
*** Message to all terminals:
***
***   requested by active unit
```

After successful reload of Standby device you will see following output on console of Active device, that version's are not identical because we yet to reboot primary device. 

```
pri/act/asa-1(config)# 
************WARNING****WARNING****WARNING********************************
   Mate version 9.6(4)45 is not identical with ours 9.6(4)36
************WARNING****WARNING****WARNING********************************
Beginning configuration replication: Sending to mate.
End Configuration Replication to mate
```

At this point lets wait for little bit until both device synchornized, you can verify that in following command, (Standby Ready)

```
pri/act/asa-1(config)# show failover 
Failover On 
Failover unit Primary
Failover LAN Interface: LANFAIL GigabitEthernet0/6 (up)
Reconnect timeout 0:00:00
Unit Poll frequency 1 seconds, holdtime 15 seconds
Interface Poll frequency 5 seconds, holdtime 25 seconds
Interface Policy 1
Monitored Interfaces 2 of 61 maximum
MAC Address Move Notification Interval not set
Version: Ours 9.6(4)36, Mate 9.6(4)45
Serial Number: Ours 9AD21GCRV4D, Mate 9A3BGKSQT5B
Last Failover at: 15:56:01 UTC Aug 23 2021
        This host: Primary - Active 
                Active time: 1088 (sec)
                slot 0: empty
                  Interface outside (192.168.255.100): Normal (Monitored)
                  Interface inside (10.10.10.1): Normal (Monitored)
        Other host: Secondary - Standby Ready 
                Active time: 0 (sec)
                  Interface outside (192.168.255.101): Normal (Monitored)
                  Interface inside (10.10.10.2): Normal (Monitored)
```

### Switch Primary/Active to Primary/Standby.

At this point we have succesfully upgraded standby firewall without disturbing live production traffic. Now we need to reload Active device which is serving live traffic so let's do failover and shift live traffic to standby device which is running on latest software. 

Followig command will make Active device Standby and shift traffic to Secondary device.   

```
pri/act/asa-1(config)# no failover active 
pri/act/asa-1(config)# 
        Switching to Standby
```

If you notice closely you prompt has been changed (pri/stby), that means you are primary firewall but in standby mode so its safe to reload. 

```
pri/stby/asa-1(config)# 
```

Reload primary device

```
pri/stby/asa-1(config)# reload 
Proceed with reload? [confirm] 
```

After successfully reload you can see both device running on latest code and you finally upgraded both device without any downtime. Enjoy!






