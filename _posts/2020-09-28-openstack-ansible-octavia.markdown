---
title: "Octavia deployment using openstack-ansible"
layout: post
date: 2020-09-13
image: /assets/images/2020-09-28-openstack-ansible-octavia/load-balancer-main.png
headerImage: true
tag:
- openstack-ansible
- openstack
- octavia
category: blog
blog: true
author: Satish Patel
description: "Octavia deployment using openstack-ansible"

---

Octavia is an OpenStack project which provides operator-grade Load Balancing. In this blog i am going to show you how did i configure octavia networking with 3 infra nodes controllers with compute and how did i wire up br-lbaas to talked to amphora running on compute nodes. 

Follow official document here to setup octavia using [openstack-ansible](https://docs.openstack.org/openstack-ansible-os_octavia/latest/configure-octavia.html#openstack-ansible-deployment). 

/etc/openstack_deploy/openstack_user_config.yml

```
cidr_networks:
  lbaas: 172.27.40.0/24
...
...
- network:
   container_bridge: "br-lbaas"
   container_type: "veth"
   container_interface: "eth14"
   host_bind_override: "eth14"
   ip_from_q: "lbaas"
   type: "raw"
   net_name: "lbaas"
   group_binds:
     - neutron_linuxbridge_agent
     - octavia-worker
     - octavia-housekeeping
     - octavia-health-manager
```

/etc/openstack_deploy/user_variables.yml

```
## Octavia
neutron_lbaas_octavia: true
octavia_ssh_enabled: true
octavia_management_net_subnet_allocation_pools: 172.27.40.200-172.27.40.250
octavia_management_net_subnet_cidr: 172.27.40.0/24
octavia_loadbalancer_topology: SINGLE
#octavia_loadbalancer_topology: ACTIVE_STANDBY
#octavia_enable_anti_affinity: True
octavia_provider_network_name: vlan
octavia_provider_segmentation_id: 27
octavia_provider_network_type: vlan
octavia_container_network_name: lbaas_address
## Octavia UI Panel
horizon_enable_octavia_ui: False
```

/etc/openstack_deploy/conf.d/octavia.yml

```
# The controller host that the octavia control plane will be run on
octavia-infra_hosts:
  os-infra-1:
    ip: 172.28.40.101
  os-infra-2:
    ip: 172.28.40.102
  os-infra-3:
    ip: 172.28.40.103
```

Next run openstack-ansible playbooks to deploy octavia on infra nodes.

```
$ openstack-ansible setup-hosts.yml
$ openstack-ansible os-octavia-install.yml 

```

The following diagram represents how br-lbaas wire-up with br-vlan to talk to amphora VMs (Notes: Just create br-lbaas empty bridge without attaching to any interface, also you don't need to configure ipaddress also)

![<img>](/assets/images/2020-09-28-openstack-ansible-octavia/octavia-lbaas-network-diagram.png)

In my case i didn't created any dedicated br-lbaas interface and attached to NIC or bond0 interface. I am using dedicated VLAN 27 for lbaas-mgmt control plane so my octavia lxc containers running on controller nodes can talk to amphora using VLAN 27. To make that magic i am using following script on contoller nodes to wire up VLAN 27 to talk to lxc container using br-vlan interface. (on compute nodes you don't need to do anything)

Adding following script in /etc/rc.local so at start up it will get activate. 

/usr/local/bin/setup-lbaas-veth.sh

```
#!/bin/bash
VLAN_ID=27

# This sets up the link
ip link add v-br-vlan type veth peer name v-br-lbaas
ip link add link v-br-lbaas name v-br-lbaas.${VLAN_ID} type vlan id ${VLAN_ID}
ip link set v-br-vlan up
ip link set v-br-lbaas up
ip link set v-br-lbaas.${VLAN_ID} up
brctl addif br-lbaas v-br-lbaas.${VLAN_ID}
brctl addif br-vlan v-br-vlan

```

Lets verify 

```
[root@os-infra-1 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
br-host		8000.000c2903d158	no		ens192
br-lbaas		8000.7a2832183e75	no		37eafb13_eth14
							v-br-lbaas.27
br-mgmt		8000.000c2903d162	no		0e85a963_eth1
							3107b36d_eth1
							37eafb13_eth1
							51bbff29_eth1
							631caea7_eth1
							84118d26_eth1
							935c9c66_eth1
							95e8e592_eth1
							9c815c04_eth1
							ab238cb6_eth1
							b59f47ae_eth1
							c319b509_eth1
							cc708a80_eth1
							e139058e_eth1
							e7b7f7c1_eth1
							ens224
br-vlan		8000.000c2903d176	no		ens161
							v-br-vlan
br-vxlan		8000.000c2903d16c	no		ens256
lxcbr0		8000.fe22fd96764c	no		0e85a963_eth0
							3107b36d_eth0
							37eafb13_eth0
							51bbff29_eth0
							631caea7_eth0
							84118d26_eth0
							935c9c66_eth0
							95e8e592_eth0
							9c815c04_eth0
							ab238cb6_eth0
							b59f47ae_eth0
							c319b509_eth0
							cc708a80_eth0
							e139058e_eth0
							e7b7f7c1_eth0
```

From here you can following regular octavia installation guide for next step to create amphora image and create load-balance. Thank you. 
