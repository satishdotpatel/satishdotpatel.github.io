---
title: "Openstack-Ansible add compute node using OpenvSwitch + DPDK"
layout: post
date: 2020-11-18
image: /assets/images/2020-11-18-openstack-ansible-add-compute-node-using-openvswitch-dpdk/openstack-ansible.png
headerImage: true
tag:
- Openstack
- Openstack-Ansible
- CentOS
- OpenvSwitch
- DPDK
category: blog
blog: true
author: Satish Patel
description: "Openstack-Ansible add compute node using OpenvSwitch + DPDK"

---

In previous post i have added OpenvSwitch based compute nodes to Openstack. When you need a high performance network and pps rate for your vms then OVS+DPDK deployment comes in picture. There are planty of articles on internet regarding ovs+dpdk so i am not going to waste a single minutes here. 

scope: I am adding *compute-lxb-3* in existing cloud for ovs+dpdk support.

![<img>](/assets/images/2020-11-18-openstack-ansible-add-compute-node-using-openvswitch-dpdk/osa-ovs-dpdk.png)

## Prepare compute Node

Notes: 
Not all NIC card support DPDK so please make sure your nic is part of this list: https://core.dpdk.org/supported/nics/ following is my nic model, Also you need latest firmware/drive of nic card otherwise you will see some unknown error when you trying to attach nic to dpdk

```
[root@compute-lxb-3 ~]# lspci | grep -i ethernet
06:00.0 Ethernet controller: Intel(R) 82599 10 Gigabit Dual Port Backplane Connection (rev 01)
06:00.1 Ethernet controller: Intel(R) 82599 10 Gigabit Dual Port Backplane Connection (rev 01)
```

Create basic network bridge interface/bridge using linuxbridge. (If you noticed i didn't create br-vlan here because we will do that in OVS)

/etc/sysconfig/network-scripts/ifcfg-bond0
```
# Bond0 Interface
NAME=bond0
DEVICE=bond0
BOOTPROTO=none
ONBOOT=yes
BONDING_OPTS="mode=1 miimon=500 downdelay=1000 primary="eno49" primary_reselect=always"
```
/etc/sysconfig/network-scripts/ifcfg-bond0.64
```
# VLAN64 for br-host Interface 
NAME=bond0.64
DEVICE=bond0.64
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-host
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-bond0/ifcfg-bond0.65
```
# VLAN65 for br-mgmt Interface
NAME=bond0.65
DEVICE=bond0.65
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-mgmt
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-host
```
# br-host Bridge
DEVICE=br-host
NAME=br-host
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.64.0.114
NETMASK=255.255.248.0
GATEWAY=10.64.0.1
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-mgmt
```
# br-mgmt Bridge
DEVICE=br-mgmt
NAME=br-mgmt
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.65.0.114
NETMASK=255.255.248.0
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
Restart NetworkManager service 

```
[root@infra-lxb-1 ~]# systemctl restart NetworkManager
```

Enable IOMMU and HugePage in grub.conf (I have 64GB memory on compute node)

```
GRUB_CMDLINE_LINUX="vmalloc=384M crashkernel=auto rd.lvm.lv=rootvg01/lv01 console=ttyS1,118200 rhgb quiet intel_iommu=on iommu=pt default_hugepagesz=1GB hugepagesz=1G hugepages=60 transparent_hugepage=never"
```

### Configure controller node

Edit /etc/openstack_deploy/user_variables.yml

```
neutron_plugin_types:
  - ml2.ovs
```

Create compute node file inside /etc/openstack_deploy/host_vars/compute-lxb-3.yml

```
# Ensure the openvswitch kernel module is loaded
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH="
    group: "network_hosts"

# Neutron specific config
neutron_plugin_type: ml2.ovs

#
neutron_provider_networks:
  network_types: "vlan"
  network_vlan_ranges: "vlan:66:68"
  network_mappings: "vlan:br-vlan"


# Enable DPDK support
ovs_dpdk_support: True

# NIC PCI bus, PMD cpu core & socket memory parameters
ovs_dpdk_pci_addresses: "0000:06:00.1"
ovs_dpdk_lcore_mask: 101000101
ovs_dpdk_pmd_cpu_mask: 202000202
ovs_dpdk_socket_mem: "1024,1024"
```

Add compute node in /etc/openstack_deploy/openstack_user_config.yml 

```
compute_hosts:
  compute-lxb-1:
    ip: 10.65.0.112
  compute-lxb-2:
    ip: 10.65.0.113 
  compute-lxb-3:
    ip: 10.65.0.114  # New OVS+DPDK compute node
```

Run playbooks

```
[root@infra-lxb-1 ~]# cd /opt/openstack-ansible/playbooks/
[root@infra-lxb-1 ~]# openstack-ansible setup-hosts.yml os-nova-install.yml os-neutron-install.yml
```

Verify 

```
[root@infra-lxb-1 ~]# lxc-attach -n infra-lxb-1_utility_container-085107e1
[root@infra-lxb-1-utility-container-085107e1 ~]#source /root/openrc
[root@infra-lxb-1-utility-container-085107e1 ~]# openstack hypervisor list
+----+-------------------------+-----------------+-------------+-------+
| ID | Hypervisor Hostname     | Hypervisor Type | Host IP     | State |
+----+-------------------------+-----------------+-------------+-------+
|  1 | compute-lxb-1.v1v0x.net | QEMU            | 10.65.0.112 | up    |
|  2 | compute-lxb-2.v1v0x.net | QEMU            | 10.65.0.113 | up    |
|  3 | compute-lxb-3.v1v0x.net | QEMU            | 10.65.0.114 | up    |
+----+-------------------------+-----------------+-------------+-------+
```

Go back to compute nodes and bind dpdk to your nic port (in my case its 0000:06:00.1 which i want to assign to dpdk)

```
[root@compute-lxb-3 ~]# driverctl -v list-devices | grep ixgbe
0000:06:00.0 ixgbe (82599 10 Gigabit Dual Port Backplane Connection (Ethernet 10Gb 2-port 560FLB Adapter))
0000:06:00.1 ixgbe (82599 10 Gigabit Dual Port Backplane Connection (Ethernet 10Gb 2-port 560FLB Adapter)) 
```

assign 06:00.1 to dpdk

```
driverctl set-override 0000:06:00.1 vfio-pci
```

verify assignment 

```
[root@compute-lxb-3 ~]# dpdk-devbind --status

Network devices using DPDK-compatible driver
============================================
0000:06:00.1 '82599 10 Gigabit Dual Port Backplane Connection 10f8' drv=vfio-pci unused=ixgbe

Network devices using kernel driver
===================================
0000:06:00.0 '82599 10 Gigabit Dual Port Backplane Connection 10f8' if=eno49 drv=ixgbe unused=vfio-pci
```

Attach dpdk port to br-vlan

```
[root@compute-lxb-3 ~]# ovs-vsctl add-port br-vlan dpdk-1 -- set Interface dpdk-1 type=dpdk options:dpdk-devargs=0000:06:00.1
```

Verify 

```
[root@compute-lxb-3 ~]# ovs-vsctl show
    ...
    Bridge br-vlan
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: netdev
        Port dpdk-1
            Interface dpdk-1
                type: dpdk
                options: {dpdk-devargs="0000:06:00.1"}
        Port br-vlan
            Interface br-vlan
                type: internal
        Port phy-br-vlan
            Interface phy-br-vlan
                type: patch
                options: {peer=int-br-vlan}
    ...
```
You will see CPU usage of ovs-vswitchd always 100% but that is normal because PMD constantly polling for packets from your NIC even you don't have any traffic.

```
top - 00:41:44 up 10 days, 23:00,  1 user,  load average: 1.00, 1.00, 1.00
Tasks: 529 total,   1 running, 528 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.1 us,  0.0 sy,  0.0 ni, 97.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  64091.5 total,     96.6 free,  62760.3 used,   1234.6 buff/cache
MiB Swap:   4096.0 total,   4056.2 free,     39.8 used.   1256.8 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 250023 openvsw+  10 -10  256.9g 141208  34872 S 100.0   0.2   5707:18 ovs-vswitchd
 251077 neutron   20   0  321068 130764  16404 S   0.7   0.2  13:15.53 /openstack/venv
   1571 root      20   0  390928  13764  11568 S   0.3   0.0   0:15.40 NetworkManager
```

Confirm that the receive queues for the instance are being serviced by a poll mode driver (PMD)
```
[root@compute-lxb-3 ~]# ovs-appctl dpif-netdev/pmd-rxq-show
pmd thread numa_id 0 core_id 1:
  isolated : false
pmd thread numa_id 0 core_id 9:
  isolated : false
pmd thread numa_id 0 core_id 25:
  isolated : false
pmd thread numa_id 0 core_id 33:
  isolated : false
  port: dpdk-1            queue-id:  0 (enabled)   pmd usage:  0 %
```
Use the following command to show statistics for the PMDs:
```
[root@compute-lxb-3 ~]# ovs-appctl dpif-netdev/pmd-stats-show
pmd thread numa_id 0 core_id 1:
  packets received: 0
  packet recirculations: 0
  avg. datapath passes per packet: 0.00
  emc hits: 0
  smc hits: 0
  megaflow hits: 0
  avg. subtable lookups per megaflow hit: 0.00
  miss with success upcall: 0
  miss with failed upcall: 0
  avg. packets per output batch: 0.00
pmd thread numa_id 0 core_id 9:
  packets received: 0
  packet recirculations: 0
  avg. datapath passes per packet: 0.00
  emc hits: 0
  smc hits: 0
  megaflow hits: 0
  avg. subtable lookups per megaflow hit: 0.00
  miss with success upcall: 0
  miss with failed upcall: 0
  avg. packets per output batch: 0.00
pmd thread numa_id 0 core_id 25:
  packets received: 0
  packet recirculations: 0
  avg. datapath passes per packet: 0.00
  emc hits: 0
  smc hits: 0
  megaflow hits: 0
  avg. subtable lookups per megaflow hit: 0.00
  miss with success upcall: 0
  miss with failed upcall: 0
  avg. packets per output batch: 0.00
pmd thread numa_id 0 core_id 33:
  packets received: 2682021
  packet recirculations: 0
  avg. datapath passes per packet: 1.00
  emc hits: 688731
  smc hits: 0
  megaflow hits: 1956691
  avg. subtable lookups per megaflow hit: 1.26
  miss with success upcall: 36599
  miss with failed upcall: 0
  avg. packets per output batch: 1.01
  idle cycles: 853974252204841 (100.00%)
  processing cycles: 12979169723 (0.00%)
  avg cycles per packet: 318411836.21 (853987231374564/2682021)
  avg processing cycles per packet: 4839.32 (12979169723/2682021)
main thread:
  packets received: 0
  packet recirculations: 0
  avg. datapath passes per packet: 0.00
  emc hits: 0
  smc hits: 0
  megaflow hits: 0
  avg. subtable lookups per megaflow hit: 0.00
  miss with success upcall: 0
  miss with failed upcall: 0
  avg. packets per output batch: 0.00
```
Use the following command to reset the PMD statistics:
```
ovs-appctl dpif-netdev/pmd-stats-clear
```
Use the following command to see interface statistics to find out error/drops etc.
```
[root@compute-lxb-3 ~]# ovs-vsctl --column statistics list interface dpdk-1
statistics          : {flow_director_filter_add_errors=0, flow_director_filter_remove_errors=0, mac_local_errors=16, mac_remote_errors=1, ovs_rx_qos_drops=0, ovs_tx_failure_drops=0, ovs_tx_invalid_hwol_drops=0, ovs_tx_mtu_exceeded_drops=0, ovs_tx_qos_drops=0, rx_128_to_255_packets=204, rx_1_to_64_packets=5407, rx_256_to_511_packets=11643, rx_512_to_1023_packets=20, rx_65_to_127_packets=2667887, rx_broadcast_packets=95799, rx_bytes=191850348, rx_crc_errors=0, rx_dropped=0, rx_errors=0, rx_fcoe_crc_errors=0, rx_fcoe_dropped=0, rx_fcoe_mbuf_allocation_errors=0, rx_fragment_errors=0, rx_illegal_byte_errors=0, rx_jabber_errors=0, rx_length_errors=0, rx_mac_short_packet_dropped=0, rx_management_dropped=0, rx_management_packets=0, rx_mbuf_allocation_errors=0, rx_missed_errors=0, rx_oversize_errors=0, rx_packets=2685161, rx_priority0_dropped=0, rx_priority0_mbuf_allocation_errors=0, rx_priority1_dropped=0, rx_priority1_mbuf_allocation_errors=0, rx_priority2_dropped=0, rx_priority2_mbuf_allocation_errors=0, rx_priority3_dropped=0, rx_priority3_mbuf_allocation_errors=0, rx_priority4_dropped=0, rx_priority4_mbuf_allocation_errors=0, rx_priority5_dropped=0, rx_priority5_mbuf_allocation_errors=0, rx_priority6_dropped=0, rx_priority6_mbuf_allocation_errors=0, rx_priority7_dropped=0, rx_priority7_mbuf_allocation_errors=0, rx_undersize_errors=0, tx_128_to_255_packets=0, tx_1_to_64_packets=0, tx_256_to_511_packets=0, tx_512_to_1023_packets=0, tx_65_to_127_packets=0, tx_broadcast_packets=0, tx_bytes=0, tx_dropped=0, tx_errors=0, tx_management_packets=0, tx_multicast_packets=0, tx_packets=0}
```

Enjoy!!
