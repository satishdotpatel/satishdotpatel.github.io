---
title: "Openstack-Ansible add SR-IOV compute node"
layout: post
date: 2020-12-19
image: /assets/images/2020-12-19-openstack-ansible-add-sriov-compute-node/openstack-ansible.png
headerImage: true
tag:
- Openstack
- Openstack-Ansible
- CentOS
- SR-IOV
category: blog
blog: true
author: Satish Patel
description: "Openstack-Ansible add SR-IOV compute node"

---

In previous post i have added OpenvSwitch + DPDK based compute nodes to Openstack. In this port i am going to show you how to add SR-IOV compute nodo for wire-speed network performance. 

scope: I'm adding *compute-lxb-4* in existing openstack cloud for SR-IOV supported compute node.

![<img>](/assets/images/2020-12-19-openstack-ansible-add-sriov-compute-node/osa-sriov.png)

## Prepare compute Node

Check vendor documentation for SR-IOV supported Ethernet NIC

```
[root@compute-lxb-4 ~]# lspci | grep -i ethernet
06:00.0 Ethernet controller: Intel(R) 82599 10 Gigabit Dual Port Backplane Connection (rev 01)
06:00.1 Ethernet controller: Intel(R) 82599 10 Gigabit Dual Port Backplane Connection (rev 01)
[root@compute-lxb-4 ~]#
[root@compute-lxb-4 ~]# lspci -vv | grep -i SR-IOV
	Capabilities: [160 v1] Single Root I/O Virtualization (SR-IOV)
	Capabilities: [160 v1] Single Root I/O Virtualization (SR-IOV)
```
My NIC firmware/driver version, make sure running latest firmware.
```
[root@compute-lxb-4 ~]# ethtool -i eno49
drver: ixgbe
version: 5.9.4
firmware-version: 0x800008fb, 1.2028.0
```

Create basic network bridge interfaces. (If you noticed i am not using nic LACP Bonding because SR-IOV doesn't support bonding)

/etc/sysconfig/network-scripts/ifcfg-eno49
```
# ostack mgmt traffic
NAME="eno49"
DEVICE="eno49"
ONBOOT=yes
NETBOOT=yes
IPV6INIT=no
BOOTPROTO=none
TYPE=Ethernet
```
/etc/sysconfig/network-scripts/ifcfg-eno49.64
```
# br-host
NAME=eno49.64
DEVICE=eno49.64
ONPARENT=yes
BOOTPROTO=static
VLAN=yes
BRIDGE=br-host
```
/etc/sysconfig/network-scripts/ifcfg-eno49.65
```
# br-mgmt
NAME=eno49.65
DEVICE=eno49.65
ONPARENT=yes
BOOTPROTO=static
VLAN=yes
BRIDGE=br-mgmt
```
/etc/sysconfig/network-scripts/ifcfg-br-host
```
# Openstack host Interface
DEVICE=br-host
NAME=br-host
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.64.0.115
NETMASK=255.255.248.0
GATEWAY=10.64.0.1
```
/etc/sysconfig/network-scripts/ifcfg-br-mgmt
```
# Openstack Managment Interface
DEVICE=br-mgmt
NAME=br-mgmt
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.65.0.115
NETMASK=255.255.248.0
```
/etc/sysconfig/network-scripts/ifcfg-eno50 ( This is our SR-IOV interface)
```
# SR-IOV Interface for VF
TYPE=Ethernet
BOOTPROTO=none
IPV6INIT=no
NAME=eno50
DEVICE=eno50
ONBOOT=yes
```
Restart NetworkManager service 

```
[root@compute-lxb-4 ~]# systemctl restart NetworkManager
```

Enable IOMMU and HugePage in grub.conf (I have 64GB memory on compute node)

```
GRUB_CMDLINE_LINUX="vmalloc=384M crashkernel=auto rd.lvm.lv=rootvg01/lv01 console=ttyS1,118200 rhgb quiet intel_iommu=on iommu=pt default_hugepagesz=1GB hugepagesz=1G hugepages=60 transparent_hugepage=never"
```
Create Virtual Function on eno50 interface. (VF interface will attached directly to your vm interface, if you create 12 VF then you can only have 12 vm nic)
Add following command in /etc/rc.local so at startup it will create VF.

```
[root@compute-lxb-4 ~]# echo '12' > /sys/class/net/eno50/device/sriov_numvfs
```
Verify VF
```
[root@compute-lxb-4 ~]# ip a | grep eno50
3: eno50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
9: eno50v0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
10: eno50v3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
11: eno50v1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
13: eno50v2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
14: eno50v5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
15: eno50v6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
16: eno50v7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
18: eno50v9: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
19: eno50v10: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
20: eno50v11: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
70: eno50v8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
71: eno50v4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
```

### Configure controller node

Create file /etc/openstack_deploy/host_vars/compute-4.yml ( nova_pci_passthrough_whitelist: is your SR-IOV nic port )

```
[root@infra-lxb-1 ~]# cat /etc/openstack_deploy/host_vars/compute-4.yml
# Select Interface for SR-IOV
nova_pci_passthrough_whitelist: '{ "physical_network":"vlan", "devname":"eno50" }'
# Install neutron sriov agent
neutron_plugin_types:
  - ml2.sriov
neutron_linuxbridge_agent_ini_overrides:
  FDB:
    shared_physical_device_mappings: "vlan:eno49,vlan:eno50"
  linux_bridge:
    physical_interface_mappings: vlan:br-vlan
neutron_sriov_nic_agent_ini_overrides:
  sriov_nic:
    physical_device_mappings: "vlan:eno49,vlan:eno50"
```
Add PciPassthroughFilter and NUMATopologyFilter scheduler filter in user_variables.yml file for nova
```
## NOVA Scheduler Setting
nova_scheduler_default_filters:
  - AvailabilityZoneFilter
  - ComputeFilter
  - AggregateInstanceExtraSpecsFilter
  - AggregateNumInstancesFilter
  - AggregateIoOpsFilter
  - ComputeCapabilitiesFilter
  - ImagePropertiesFilter
  - ServerGroupAntiAffinityFilter
  - ServerGroupAffinityFilter
  - NUMATopologyFilter
  - PciPassthroughFilter
```
Add compute node in /etc/openstack_deploy/openstack_user_config.yml 
```
compute_hosts:
  compute-lxb-1:
    ip: 10.65.0.112
  compute-lxb-2:
    ip: 10.65.0.113 
  compute-lxb-3:
    ip: 10.65.0.114  
  compute-lxb-4:
    ip: 10.65.0.115  # New SRIOV compute node
```
Run playbooks:
```
[root@infra-lxb-1 ~]# cd /opt/openstack-ansible/playbooks/
[root@infra-lxb-1 ~]# openstack-ansible setup-hosts.yml os-nova-install.yml os-neutron-install.yml
```
Validate:
```
[root@infra-lxb-1 ~]# lxc-attach -n infra-lxb-1_utility_container-085107e1
[root@infra-lxb-1-utility-container-085107e1 ~]#source /root/openrc
[root@infra-lxb-1-utility-container-085107e1 ~]# openstack hypervisor list
+----+-------------------------+-----------------+-------------+-------+
| ID | Hypervisor Hostname     | Hypervisor Type | Host IP     | State |
+----+-------------------------+-----------------+-------------+-------+
|  1 | compute-lxb-1.spatel.net | QEMU            | 10.65.0.112 | up    |
|  2 | compute-lxb-2.spatel.net | QEMU            | 10.65.0.113 | up    |
|  3 | compute-lxb-3.spatel.net | QEMU            | 10.65.0.114 | up    |
|  4 | compute-lxb-4.spatel.net | QEMU            | 10.65.0.115 | up    |
+----+-------------------------+-----------------+-------------+-------+
```
Validate SR-IOV neutron agent service
```
[root@infra-lxb-1-utility-container-085107e1 ~ (lab-keystone)]> neutron agent-list --agent_type 'NIC Switch agent'
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+------------------+-------------------------+-------------------+-------+----------------+-------------------------+
| id                                   | agent_type       | host                    | availability_zone | alive | admin_state_up | binary                  |
+--------------------------------------+------------------+-------------------------+-------------------+-------+----------------+-------------------------+
| 23b59590-95d6-4bef-a8a6-8d7ab275aae5 | NIC Switch agent | compute-lxb-4.spatel.net |                   | :-)   | True           | neutron-sriov-nic-agent |
+--------------------------------------+------------------+-------------------------+-------------------+-------+----------------+-------------------------+
```

### Launch sriov nic supported instance

First you need to create neutron sriov port

```
[root@infra-lxb-1-utility-container-085107e1 ~]# openstack port create --network vlan66 --vnic-type=direct sriov-port1
```
Launch VM 

```
[root@infra-lxb-1-utility-container-085107e1 ~]# openstack server create --flavor m1.small \
  --image centos-8 \
  --nic port-id=`openstack port list | grep sriov-port1 | awk '{print $2}'` \
  vm1
```
Verify the instance boots with the SRIOV port. Verify VF assignment by running dmesg on the compute node where the instance was placed.
```
[root@compute-lxb-4 ~]# dmesg | grep VF
...
[1344115.509695] ixgbevf 0000:06:11.1: Intel(R) 82599 Virtual Function
[1344217.798027] ixgbe 0000:06:00.1: setting MAC fa:16:3e:26:14:f5 on VF 8
[1344217.885108] ixgbe 0000:06:00.1: Reload the VF driver to make this change effective.
[1344217.988273] ixgbe 0000:06:00.1: Setting VLAN 66, QOS 0x0 on VF 8
...
```
Enjoy!!
