---
title: "Maximizing Packet Capture Performance with Tcpdump"
layout: post
date: 2020-10-26
image: /assets/images/2020-10-26-maximizing-packet-capture-performance-with-tcpdump/hose-on-face.png
headerImage: true
tag:
- Tcpdump
- PF-Ring
- Linux
category: blog
blog: true
author: Satish Patel
description: "Maximizing Packet Capture Performance with Tcpdump"

---

Many times we have noticed packet drop in tcpdump output, In short tcpdump isn't meant to capture high packet per second rate because they way packets get handled by the kernel. As the packet propagates from Network Interface Controller (NIC) to the kernel and then to the userspace application, it creates some overhead on the system. Under heavy traffic conditions the percentage of the captured packets over the total number can decrease. Packet size does play a significant factor, as the smaller the packet size the higher the negative impact in the packet capture percentage.

Here i am going to show you how you can use PF_Ring technology to boots tcpdump performance to capture high packet rates with very less CPU overhead.

### What is PF_Ring?

PF_RING is a Linux kernel module and user-space framework that allows you to process packets at high-rates while providing you a consistent API for packet processing applications.

It can range from 80k pkt/sec on a 1,2GHz ARM to 15M pkt/sec and above per core on a low-end 2,5GHz Xeon. It not only enables you to capture packets faster, it also captures packets more efficiently preserving CPU cycles.

PF_Ring packet flow compare to traditional method.

![<img>](/assets/images/2020-10-26-maximizing-packet-capture-performance-with-tcpdump/pfring-diagram-small.png)

Enough, Lets get our hand dirty. First install pfring on CentOS 7.5

```
git clone https://github.com/ntop/PF_RING.git
cd PF_RING
```

Prerequisite ( make sure you download same kernel-source tree which you are running)

```
yum -y install kernel-devel
yum -y install bison flex elfutils-libelf-devel
yum -y install gcc
```

### Compile PF_Ring 

```
cd PF_RING
make
make -C kernel install
make -C userland/lib install
```

### Compile Tcpdump

```
cd userland
make all
make tcpdump/Makefile
make build_tcpdump
```

copy new tcpdump binary to system PATH with differnet name.

```
cp tcpdump/tcpdump /usr/local/sbin/tcpdump_pfring
```

Compare older tcpdump and newer tcpdump 

```
# /usr/sbin/tcpdump --version
tcpdump version 4.9.2
libpcap version 1.5.3
OpenSSL 1.0.2k-fips  26 Jan 2017
 
 
# /usr/local/sbin/tcpdump_pfring --version
tcpdump_pfring version 4.9.3
libpcap version 1.9.1 (with TPACKET_V3)
```

Load pf_ring kernel module 

```
insmod PF_RING/kernel/pf_ring.ko
```

Verify module is loaded 

```
# lsmod | grep pf_ring
pf_ring               726561  0
```

```
# cat /proc/net/pf_ring/info
PF_RING Version          : 7.7.0 (dev:9e353477f985b9709cedfa4265fcf4864825c0c6)
Total rings              : 0
 
Standard (non ZC) Options
Ring slots               : 4096
Slot version             : 17
Capture TX               : Yes [RX+TX]
IP Defragment            : No
Socket Mode              : Standard
Cluster Fragment Queue   : 0
Cluster Fragment Discard : 0
```

### Performance Test

Lets run performance test and compare old tcpdump Vs new tcpdump_pfring. I use hping3 utility to generate small size high packet per second UDP packets, my hping3 test generating 1.3 million packet per 30 second. 

Old tcpdump

```
# tcpdump -ni ens4 udp -w /var/tmp/foo.pcap -vv
tcpdump: listening on ens4, link-type EN10MB (Ethernet), capture size 262144 bytes
1313618 packets captured
1313878 packets received by filter
260 packets dropped by kernel           <------------ yikes!!!
```

New tcpdump_pfring

```
# tcpdump_pfring -ni ens4 udp -w /var/tmp/foo.pcap -vv
tcpdump_pfring: listening on ens4, link-type EN10MB (Ethernet), capture size 262144 bytes
1333521 packets captured
1333521 packets received by filter
0 packets dropped by kernel            <------------ Bravo!!!
```

Enjoy your new high performance tcpdump :) 



