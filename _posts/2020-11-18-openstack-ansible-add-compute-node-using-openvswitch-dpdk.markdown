---
title: "Openstack-Ansible Add compute node using OpenvSwitch + DPDK"
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
description: "Openstack-Ansible Add compute node using OpenvSwitch + DPDK"

---

In previous post i have added OpenvSwitch based compute nodes to Openstack. When you need a high performance network and pps rate for your vms then OVS+DPDK deployment comes in picture. There are planty of articles on internet regarding ovs+dpdk so i am not going to waste a single minutes here. 

![<img>](/assets/images/2020-11-18-openstack-ansible-add-compute-node-using-openvswitch-dpdk/osa-ovs-dpdk.png)

## Prepare compute Node

###Notes: 
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

Enjoy!!
