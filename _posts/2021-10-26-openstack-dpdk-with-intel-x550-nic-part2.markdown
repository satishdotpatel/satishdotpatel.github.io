---
title: "Openstack OVS-DPDK with Intel X550T NIC - Part2"
layout: post
date: 2021-10-26
image: /assets/images/2021-10-22-openstack-dpdk-with-intel-x550t2-nic/fastlane.png
headerImage: true
tag:
- Openstack
- DPDK
- Openstack-Ansible
- X550T
- OpenvSwitch
- Ubuntu
category: blog
blog: true
author: Satish Patel
description: "Openstack OVS-DPDK with Intel X550T NIC - Part2"

---

In a previous post I have explained how to configure DPDK with Intel X550 nic. In this post we will look at what we can do to optimize DPDK performance. 

#### CPU 

```
# lscpu
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   46 bits physical, 48 bits virtual
CPU(s):                          48
On-line CPU(s) list:             0-47
Thread(s) per core:              2
Core(s) per socket:              12
Socket(s):                       2
NUMA node(s):                    2
Vendor ID:                       GenuineIntel
CPU family:                      6
Model:                           85
Model name:                      Intel(R) Xeon(R) Silver 4214 CPU @ 2.20GHz
Stepping:                        7
CPU MHz:                         1000.005
BogoMIPS:                        4400.00
Virtualization:                  VT-x
L1d cache:                       768 KiB
L1i cache:                       768 KiB
L2 cache:                        24 MiB
L3 cache:                        33 MiB
NUMA node0 CPU(s):               0,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30,32,34,36,38,40,42,44,46
NUMA node1 CPU(s):               1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31,33,35,37,39,41,43,45,47
```

#### NUMA

Let's take a look at NUMA layout. 

```
$ numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30 32 34 36 38 40 42 44 46
node 0 size: 47816 MB
node 0 free: 111 MB
node 1 cpus: 1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31 33 35 37 39 41 43 45 47
node 1 size: 48353 MB
node 1 free: 51 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

#### NIC 

If you noticed both DPDK nic port attached to NUMA0 and NUMA1. 

![<img>](/assets/images/2021-10-22-openstack-dpdk-with-intel-x550t2-nic/vpn-network-setup.png){: width="800" }


#### CPU isolation

I'm going to assigned dedicated core for HOST OS and DPDK PMD thread and rest for VMs. 

![<img>](/assets/images/2021-10-22-openstack-dpdk-with-intel-x550t2-nic/dpdk-numa.png){: width="800" }

* HOST OS cores - 0,24,1,25
* DPDK PMD cores - 2,26,3,27
* VMs cores - 4-23,28-47 

We have to configure following in grub to isolate CPU cores. I have used 2-23,26-47 that will include DPDK-PMD cores and VMs cores. 

```
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=0 rd.auto intel_iommu=on iommu=pt default_hugepagesz=1GB hugepagesz=1G hugepages=88 transparent_hugepage=never isolcpus=2-23,26-47 nohz_full=2-23,26-47 rcu_nocbs=2-23,26-47"
```

After reboot machine you can validate using following command. It shows HOST OS has only 4 cores available and rest isolated.

```
$ taskset -cp 1
pid 1's current affinity list: 0,1,24,25
```

Now i am going to define my VMs cores in nova.conf. as you noticed i didn't include DPDK PMD cores. 

```
[DEFAULT]
# vCPU Set
vcpu_pin_set = 4-23,28-47
```



#### DPDK PMD CPU MASK

Once we have isolated HOST OS cores and VM's cores now i am going to tell OpenvSwitch about DPDK PMD core. This is little tricky because you have to use bitmask instead of core numbers. Here is the script which will help you to generate bitmask https://gist.github.com/satishdotpatel/d28ea78e8731363b510ab8a4f3c09098

```
$ ./dpdk-pmd-mask.py 2,26,3,27
c00000c
```

Run following command to configure PMD-CPU-MASK

```
$ ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=c00000c
```

Validation

```
ovs-appctl dpif-netdev/pmd-rxq-show
pmd thread numa_id 0 core_id 2:
  isolated : false
pmd thread numa_id 1 core_id 3:
  isolated : false
  port: dpdk1             queue-id:  0 (enabled)   pmd usage: NOT AVAIL
pmd thread numa_id 0 core_id 26:
  isolated : false
  port: dpdk0             queue-id:  0 (enabled)   pmd usage: NOT AVAIL
pmd thread numa_id 1 core_id 27:
  isolated : false
```

Lets create VM and validate CPU pinning. 

First create flavor with property mem_page_size='large' without hugepage DPDK won't work. You can spin up vm but not ping. 

```
$ openstack flavor create --id 4 --ram 8192  --vcpus 8 --disk 100 --swap 4096 --property hw:mem_page_size='large' --property hw:cpu_policy='dedicated' --property hw:cpu_sockets='2' --property hw:cpu_threads='2' dpdk.medium
```

After successfully spun up VM you can see following vcpu layout. It will shows VCPU mapping with physical CORE. 

```
root@ostack-ams-comp-dpdk-5:~# virsh vcpuinfo instance-0000007f
VCPU:           0
CPU:            10
State:          running
CPU time:       8.2s
CPU Affinity:   ----------y-------------------------------------

VCPU:           1
CPU:            34
State:          running
CPU time:       2.2s
CPU Affinity:   ----------------------------------y-------------

VCPU:           2
CPU:            12
State:          running
CPU time:       2.0s
CPU Affinity:   ------------y-----------------------------------

VCPU:           3
CPU:            36
State:          running
CPU time:       3.8s
CPU Affinity:   ------------------------------------y-----------

VCPU:           4
CPU:            18
State:          running
CPU time:       2.8s
CPU Affinity:   ------------------y-----------------------------

VCPU:           5
CPU:            42
State:          running
CPU time:       2.2s
CPU Affinity:   ------------------------------------------y-----

VCPU:           6
CPU:            6
State:          running
CPU time:       1.5s
CPU Affinity:   ------y-----------------------------------------

VCPU:           7
CPU:            30
State:          running
CPU time:       12.7s
CPU Affinity:   ------------------------------y-----------------
```











