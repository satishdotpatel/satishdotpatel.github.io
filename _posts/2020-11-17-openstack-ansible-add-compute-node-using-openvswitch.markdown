---
title: "Openstack-Ansible Add compute node using OpenvSwitch"
layout: post
date: 2020-11-08
image: /assets/images/2020-11-17-openstack-ansible-add-compute-node-using-openvswitch/openstack-ansible.png
headerImage: true
tag:
- Openstack
- Openstack-Ansible
- CentOS
- OpenvSwitch
category: blog
blog: true
author: Satish Patel
description: "Openstack-Ansible Add compute node using OpenvSwitch"

---

In previous post i have showed you how to deploy openstack using openstack-ansible (OSA) using LinuxBridge. In this post i will show you how to new add compute node but use OpenvSwitch for br-vlan instance (vms) traffic. In short LinuxBridge for control plane and vm traffic will use OVS. 

![<img>](/assets/images/2020-11-17-openstack-ansible-add-compute-node-using-openvswitch/osa-ovs.png)

## Prepare compute Node

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
IPADDR=10.64.0.113
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
IPADDR=10.65.0.113
NETMASK=255.255.248.0
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
Restart NetworkManager service 

```
[root@infra-lxb-1 ~]# systemctl restart NetworkManager
```

### Configure controller node

Edit /etc/openstack_deploy/user_variables.yml

```
neutron_plugin_types:
  - ml2.ovs
```

Create compute node file inside /etc/openstack_deploy/host_vars/compute-lxb-2.yml

```
# Ensure the openvswitch kernel module is loaded
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH="
    group: "network_hosts"

# Neutron agent plugin
neutron_plugin_type: ml2.ovs

# Neutron provider network
neutron_provider_networks:
  network_types: "vlan"
  network_vlan_ranges: "vlan:66:68"
  network_mappings: "vlan:br-vlan"
```

Add compute node in /etc/openstack_deploy/openstack_user_config.yml 

```
compute_hosts:
  compute-lxb-1:
    ip: 10.65.0.112
  compute-lxb-2:
    ip: 10.65.0.113  # New OVS compute node
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
+----+-------------------------+-----------------+-------------+-------+
```

Go to compute nodes and bind br-vlan interface to bond0 on compute node. (This step is manual but you can write small playbook to do that)

```
[root@compute-lxb-2 ~]# ovs-vsctl add-port br-vlan bond0
```

Lets verify LinuxBridge, as you can see br-host and br-mgmt only using LinuxBridge.

```
[root@compute-lxb-2 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
br-host		8000.1402ec85694c	no		bond0.64
br-mgmt		8000.1402ec85694c	no		bond0.65
lxcbr0		8000.000000000000	no
```

Lets verify OpenvSwitch (you can see br-vlan is part of bond0, which vms will use for data traffic)

```
[root@compute-lxb-2 ~]# ovs-vsctl show
c48809ab-d8d4-470e-a7ba-05398bf08d55
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-vlan
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port phy-br-vlan
            Interface phy-br-vlan
                type: patch
                options: {peer=int-br-vlan}
        Port br-vlan
            Interface br-vlan
                type: internal
        Port bond0
            Interface bond0
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
        Port int-br-vlan
            Interface int-br-vlan
                type: patch
                options: {peer=phy-br-vlan}
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port vxlan-0a410072
            Interface vxlan-0a410072
                type: vxlan
                options: {df_default="true", egress_pkt_mark="0", in_key=flow, local_ip="10.65.0.113", out_key=flow, remote_ip="10.65.0.114"}
        Port br-tun
            Interface br-tun
                type: internal
    ovs_version: "2.13.0"
```

I would prefer using LinuxBridge for openstack controller and use ovs for just vms traffic on compute nodes. It will get little complicated if you try to deploy everything using openvswitch. 




