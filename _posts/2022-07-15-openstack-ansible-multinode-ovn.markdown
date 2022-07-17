---
title: "Openstack-Ansible Multi-Node OVN Deployment"
layout: post
date: 2022-07-15
image: /assets/images/2022-07-15-openstack-ansible-multinode-ovn/ovn-sdn-logo.png
headerImage: true
tag:
- Openstack-Ansible
- OVN
- Openstack
category: blog
blog: true
author: Satish Patel
description: "Openstack-Ansible Multi-Node OVN Deployment"

---

In this blog i am going to show you how to deploy ovn based networking using openstack-ansible openstack deployment tool in multi-node environment for high availability. 

#### Lab 

I am using VMware ESXi host to run 5 vms which include 3 infra nodes and 2 compute nodes.

![<img>](/assets/images/2022-07-15-openstack-ansible-multinode-ovn/ovn-lab.png){: width="600" }

#### Network configuration. 

ovn-infra-1 netplan (2 other nodes has similar config except ipaddr)

```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens192:
      dhcp4: no
    ens224:
      dhcp4: no
    ens161:
      dhcp4: no
    ens256:
      dhcp4: no
    ens193:
      dhcp4: no

  bridges:
    br-host:
      interfaces: [ ens192 ]
      addresses: [ 10.64.7.11/21 ]
      gateway4: 10.64.0.1
      nameservers:
        addresses: [ 10.64.0.10, 10.64.0.11 ]
        search: [ v1v0x.net, vivox.com ]

  bridges:
    br-mgmt:
      interfaces: [ ens224 ]
      addresses: [ 10.99.7.11/24 ]

  bridges:
    br-vxlan:
      interfaces: [ ens161 ]
      addresses: [ 10.100.7.11/24 ]

  bridges:
    br-storage:
      interfaces: [ ens256 ]
      addresses: [ 10.101.7.11/24 ]

  bridges:
    br-vlan:
      interfaces: [ ens193 ]
```

#### Openstack-Ansible config

/etc/openstack_deploy/openstack_user_config.yml

```
---
cidr_networks:
  container: 10.99.7.0/24
  tunnel: 10.100.7.0/24

used_ips:
  - "10.99.7.0,10.99.7.100"
  - "10.100.7.0,10.100.7.100"

global_overrides:
  internal_lb_vip_address: 10.99.7.5
  #
  # The below domain name must resolve to an IP address
  # in the CIDR specified in haproxy_keepalived_external_vip_cidr.
  # If using different protocols (https/http) for the public/internal
  # endpoints the two addresses must be different.
  #
  external_lb_vip_address: openstack-phx-lab.vivox.com
  tunnel_bridge: "br-vxlan"
  management_bridge: "br-mgmt"
  provider_networks:
    - network:
        container_bridge: "br-mgmt"
        container_type: "veth"
        container_interface: "eth1"
        ip_from_q: "container"
        type: "raw"
        group_binds:
          - all_containers
          - hosts
        is_container_address: true
        is_ssh_address: true
    - network:
        container_bridge: "br-vxlan"
        container_type: "veth"
        container_interface: "eth10"
        ip_from_q: "tunnel"
        type: "geneve"
        range: "1:1000"
        net_name: "geneve"
        group_binds:
          - neutron_ovn_controller
    - network:
        container_bridge: br-provider
        container_type: veth
        net_name: vlan
        network_interface: br-vlan
        range: 101:200,301:400
        type: vlan
        group_binds:
          - neutron_ovn_controller

##################################################
###            Infrastructure                  ###
##################################################

## galera, memcache, rabbitmq, utility
shared-infra_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

## repository (apt cache, python packages, etc)
repo-infra_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

## load balancer
haproxy_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12

## openstack roles
image_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

orchestration_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

dashboard_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

identity_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

network_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

compute-infra_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

placement-infra_hosts:
  ovn-infra-1:
    ip: 10.99.7.11
  ovn-infra-2:
    ip: 10.99.7.12
  ovn-infra-3:
    ip: 10.99.7.13

compute_hosts:
  ovn-compute-1:
    ip: 10.99.7.21
  ovn-compute-2:
    ip: 10.99.7.22

```

/etc/openstack_deploy/user_variables.yml

```
---
## Debug and Verbose options.
debug: false

## Disable security hardening
apply_security_hardening: false

# repositories to install OpenStack using distribution packages
install_method: source

## Region
service_region: lab

dhcp_domain: example.com

rabbitmq_monitoring_userid: monitoring
rabbitmq_upgrade: false

haproxy_keepalived_external_vip_cidr: "10.64.0.5/32"
haproxy_keepalived_internal_vip_cidr: "{{internal_lb_vip_address}}/32"
haproxy_keepalived_external_interface: br-host
haproxy_keepalived_internal_interface: br-mgmt

########### Horizon settings ###########
horizon_launch_instance_defaults:
  create_volume: False
horizon_keystone_multidomain_support: True
horizon_show_keystone_v2_rc: True

########### Galera settings ############
galera_cluster_name: openstack_galera_cluster
galera_max_connections: 3200

########### Neutron settings ############
neutron_metadata_checksum_fix: False
neutron_dns_domain: example.com
neutron_vxlan_group: "239.0.0.1"
neutron_rpc_workers: 1

########### Nova settings ###############
nova_console_type: novnc
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

########### Heat setting ###########
heat_wsgi_processes: 1
heat_api_threads: 1
heat_engine_workers: 1


########## OVN ##############
neutron_plugin_type: ml2.ovn
neutron_plugin_base:
  - neutron.services.ovn_l3.plugin.OVNL3RouterPlugin
neutron_ml2_drivers_type: "vlan,local,geneve"
```

/etc/openstack_deploy/env.d/neutron.yml

```
component_skel:
  neutron_ovn_controller:
    belongs_to:
      - neutron_all
  neutron_ovn_northd:
    belongs_to:
      - neutron_all

container_skel:
  neutron_agents_container:
    contains: {}
  neutron_ovn_northd_container:
    belongs_to:
      - network_containers
    contains:
      - neutron_ovn_northd
```

/etc/openstack_deploy/env.d/nova.yml

```
container_skel:
  nova_compute_container:
    belongs_to:
      - compute_containers
      - kvm-compute_containers
      - lxd-compute_containers
      - qemu-compute_containers
    contains:
      - neutron_ovn_controller
      - nova_compute
    properties:
      is_metal: true
```

/etc/openstack_deploy/group_vars/network_hosts

```
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH"
```

#### OVN Architecture

![<img>](/assets/images/2022-07-15-openstack-ansible-multinode-ovn/ovn-architecture.png){: width="600" }

#### OVN clustering

Go to ovn-northd container on any infra node and run following command. Its telling you status of Northbound database and clustering status. 

In following output you can see ovn clustering status like (Role: leader). Thats means following node (ovn-infra-1) is leader and other 2 nodes are follower and they sync database from leader. 
```
root@ovn-infra-1-neutron-ovn-northd-container-f0375a6d:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
ff50
Name: OVN_Northbound
Cluster ID: 8223 (8223c67c-7f12-452e-8d4b-1f64d6f11799)
Server ID: ff50 (ff50b7c0-aeec-484b-b9e4-5a8446491b68)
Address: tcp:10.99.7.152:6643
Status: cluster member
Role: leader
Term: 34
Leader: self
Vote: self

Last Election started 80016285 ms ago, reason: leadership_transfer
Last Election won: 80016245 ms ago
Election timer: 1000
Log: [424, 444]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: ->88dc <-88dc ->2381 <-2381
Disconnections: 4
Servers:
    ff50 (ff50 at tcp:10.99.7.152:6643) (self) next_index=425 match_index=443
    88dc (88dc at tcp:10.99.7.153:6643) next_index=444 match_index=443 last msg 113 ms ago
    2381 (2381 at tcp:10.99.7.207:6643) next_index=444 match_index=443 last msg 113 ms ago
```

Check cluster status of Southbound DB. Sometime you can find Southbound DB leader on other nodes. Its not required both North and South DB leader on same node. 

```
root@ovn-infra-1-neutron-ovn-northd-container-f0375a6d:~# ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
4a1c
Name: OVN_Southbound
Cluster ID: 783a (783afb94-7ff5-4e1b-8826-ef2dc9c572e4)
Server ID: 4a1c (4a1c793c-f1c4-4fc6-b8e0-1a515fb9d4a5)
Address: tcp:10.99.7.152:6644
Status: cluster member
Role: leader
Term: 34
Leader: self
Vote: self

Last Election started 80392243 ms ago, reason: leadership_transfer
Last Election won: 80392219 ms ago
Election timer: 1000
Log: [413, 435]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: ->1f5e <-1f5e ->419b <-419b
Disconnections: 4
Servers:
    4a1c (4a1c at tcp:10.99.7.152:6644) (self) next_index=414 match_index=434
    1f5e (1f5e at tcp:10.99.7.153:6644) next_index=435 match_index=434 last msg 323 ms ago
    419b (419b at tcp:10.99.7.207:6644) next_index=435 match_index=434 last msg 323 ms ago
```

Check ovn-infra-2 node ovn cluster status. As you can see its Role is follower and when you loose leader node then election happened and which will decided which follower node will be next leader. 


```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
88dc
Name: OVN_Northbound
Cluster ID: 8223 (8223c67c-7f12-452e-8d4b-1f64d6f11799)
Server ID: 88dc (88dcb194-5fc4-4132-9275-91a46261c55c)
Address: tcp:10.99.7.153:6643
Status: cluster member
Role: follower
Term: 34
Leader: ff50
Vote: ff50

Last Election started 80860368 ms ago, reason: leadership_transfer
Last Election won: 80860324 ms ago
Election timer: 1000
Log: [425, 444]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: ->ff50 <-ff50 ->2381 <-2381
Disconnections: 1
Servers:
    ff50 (ff50 at tcp:10.99.7.152:6643) last msg 79 ms ago
    88dc (88dc at tcp:10.99.7.153:6643) (self)
    2381 (2381 at tcp:10.99.7.207:6643) last msg 80634326 ms ago
```

For experiment lets restart ovn-infra-1 node ovn-northd container and see what happened. 

```
root@ovn-infra-1:~# lxc-stop -n ovn-infra-1_neutron_ovn_northd_container-f0375a6d
root@ovn-infra-1:~# lxc-start -n ovn-infra-1_neutron_ovn_northd_container-f0375a6d
```

Check ovn-infra-1 node ovn cluster role. As you can see its nolonger leader now but follower after reboot.

```
root@ovn-infra-1-neutron-ovn-northd-container-f0375a6d:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep Role
Role: follower
```

ovn-infra-2 is now leader

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep Role
Role: leader
```

Lets explore northbound db using ovn-nbctl utility.  

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-nbctl show
switch 1414bf74-fe98-47e7-bca3-a1adf9eb8992 (neutron-732a4b87-2589-4c54-9e54-5b383d2275a4) (aka private1)
    port 6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69
        type: localport
        addresses: ["fa:16:3e:14:94:57 10.1.1.10"]
    port bb603782-dab0-42ea-a004-b4b3869fb6a5
        type: router
        router-port: lrp-bb603782-dab0-42ea-a004-b4b3869fb6a5
switch 91930dfd-fa3a-4181-a5cb-72ceea977690 (neutron-1335afde-03ef-4158-b4d8-f5fc2c9d3118) (aka public1)
    port e7ea6e1b-e77f-4d01-854f-65d52a43b6d4
        type: localport
        addresses: ["fa:16:3e:4c:25:4c"]
    port c1ff73df-5bac-40c0-9ba6-f792dea9a57f
        type: router
        router-port: lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
    port provnet-c4dc39f7-ee70-479e-a9c9-db8bd572faa0
        type: localnet
        tag: 100
        addresses: ["unknown"]
router befa5e54-86e8-42c8-97e3-e98d88bc6b1f (neutron-92edd610-1baa-491f-9324-e261c95cead2) (aka router1)
    port lrp-bb603782-dab0-42ea-a004-b4b3869fb6a5
        mac: "fa:16:3e:92:af:22"
        networks: ["10.1.1.1/24"]
    port lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
        mac: "fa:16:3e:78:62:13"
        networks: ["69.125.124.194/23"]
        gateway chassis: [8a3ada6a-e062-49c2-abb7-0221a29b3ce4 ed74f491-7389-4730-af3b-645874a660a8]
    nat c53c5ae2-2013-4017-8fd2-bbe050566769
        external ip: "69.125.124.194"
        logical ip: "10.1.1.0/24"
        type: "snat"
```

ovn-nbctl only works on Leader node and if you are running that command on non-leader node then you will get following error. ovn-infra-1 is not leader. If you want to run command on non-leader then using (ovn-nbctl --no-leader-only show) 

```
root@ovn-infra-1-neutron-ovn-northd-container-f0375a6d:~# ovn-nbctl show
ovn-nbctl: unix:/var/run/ovn/ovnnb_db.sock: database connection failed ()
```

Check Southbound db using ovn-sbctl utility. Southbound container information of hypervisors and port binding to hypervisors.  

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-sbctl show
Chassis "ed74f491-7389-4730-af3b-645874a660a8"
    hostname: ovn-compute-1.example.com
    Encap vxlan
        ip: "10.100.7.21"
        options: {csum="true"}
    Encap geneve
        ip: "10.100.7.21"
        options: {csum="true"}
Chassis "8a3ada6a-e062-49c2-abb7-0221a29b3ce4"
    hostname: ovn-compute-2.example.com
    Encap geneve
        ip: "10.100.7.22"
        options: {csum="true"}
    Encap vxlan
        ip: "10.100.7.22"
        options: {csum="true"}
    Port_Binding "6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69"
    Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
```

#### OVN Port Binding of VM

Lets spin up one vm and see how it looks in North and South DB


```
root@ovn-infra-1:~# openstack server create --flavor m1.small --image cirros --nic net-id=private1 vm1

root@ovn-infra-1:~# nova list --host ovn-compute-2.example.com
+--------------------------------------+------+--------+------------+-------------+--------------------+
| ID                                   | Name | Status | Task State | Power State | Networks           |
+--------------------------------------+------+--------+------------+-------------+--------------------+
| ca55a0f1-ecd7-44ec-aac6-da8fafed9eab | vm1  | ACTIVE | -          | Running     | private1=10.1.1.97 |
+--------------------------------------+------+--------+------------+-------------+--------------------+

```

Lets check Northbound DB. You can see vm1 port attached to private1 switch. In ovn terminology switch is network. 

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-nbctl show
switch 1414bf74-fe98-47e7-bca3-a1adf9eb8992 (neutron-732a4b87-2589-4c54-9e54-5b383d2275a4) (aka private1)
    port 6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69
        type: localport
        addresses: ["fa:16:3e:14:94:57 10.1.1.10"]
    port bb603782-dab0-42ea-a004-b4b3869fb6a5
        type: router
        router-port: lrp-bb603782-dab0-42ea-a004-b4b3869fb6a5
    port 967b3f42-a881-4ac3-957f-dbf26756b2f2
        addresses: ["fa:16:3e:8a:8c:f3 10.1.1.97"]
...
...
```

check Southbound DB after vm creation. (Port_Binding "967b3f42-a881-4ac3-957f-dbf26756b2f2" is attached to compute-2, same port ID you can see in Northbound DB in above output)

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-sbctl show
Chassis "ed74f491-7389-4730-af3b-645874a660a8"
    hostname: ovn-compute-1.example.com
    Encap vxlan
        ip: "10.100.7.21"
        options: {csum="true"}
    Encap geneve
        ip: "10.100.7.21"
        options: {csum="true"}
Chassis "8a3ada6a-e062-49c2-abb7-0221a29b3ce4"
    hostname: ovn-compute-2.example.com
    Encap geneve
        ip: "10.100.7.22"
        options: {csum="true"}
    Encap vxlan
        ip: "10.100.7.22"
        options: {csum="true"}
    Port_Binding "6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69"
    Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
    Port_Binding "967b3f42-a881-4ac3-957f-dbf26756b2f2"
```

Attach floating IP and see how Northbound DB looks

```
root@ovn-infra-1:~# openstack server add floating ip vm1 69.125.124.198
```

Check northbound DB. As you can see in router/nat section vm1 ip associated with public floating ip. 

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-nbctl show 
...
...
router befa5e54-86e8-42c8-97e3-e98d88bc6b1f (neutron-92edd610-1baa-491f-9324-e261c95cead2) (aka router1)
    port lrp-bb603782-dab0-42ea-a004-b4b3869fb6a5
        mac: "fa:16:3e:92:af:22"
        networks: ["10.1.1.1/24"]
    port lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
        mac: "fa:16:3e:78:62:13"
        networks: ["69.125.124.194/23"]
        gateway chassis: [8a3ada6a-e062-49c2-abb7-0221a29b3ce4 ed74f491-7389-4730-af3b-645874a660a8]
    nat 61c83906-97ca-4e62-9896-c9f4267c292b
        external ip: "69.125.124.198"
        logical ip: "10.1.1.97"
        type: "dnat_and_snat"
    nat c53c5ae2-2013-4017-8fd2-bbe050566769
        external ip: "69.125.124.194"
        logical ip: "10.1.1.0/24"
        type: "snat"
```

#### OVN Gateway Chassis

Check who is the gateway chassis and responsible for extranal connectivity. In following output ovn-compute-1 and ovn-compute-2 are gateway chassis. The chassis with the highest priority will be hosting the gateway router port and all vm send traffic via that gateway port. 

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-nbctl list gateway_chassis
_uuid               : 577780c8-4729-47f9-b691-4d4e1eda8694
chassis_name        : "ed74f491-7389-4730-af3b-645874a660a8"
external_ids        : {}
name                : lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f_ed74f491-7389-4730-af3b-645874a660a8
options             : {}
priority            : 1

_uuid               : 2486f2f3-6bfb-4557-a37b-441ecf5549bc
chassis_name        : "8a3ada6a-e062-49c2-abb7-0221a29b3ce4"
external_ids        : {}
name                : lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f_8a3ada6a-e062-49c2-abb7-0221a29b3ce4
options             : {}
priority            : 2
```

You can see southbound db output to check outer port binding on ovn-compute-2 chassis because it has higher priority which is 2 (Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f)

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-sbctl show
Chassis "ed74f491-7389-4730-af3b-645874a660a8"
    hostname: ovn-compute-1.v1v0x.net
    Encap vxlan
        ip: "10.100.7.21"
        options: {csum="true"}
    Encap geneve
        ip: "10.100.7.21"
        options: {csum="true"}
Chassis "8a3ada6a-e062-49c2-abb7-0221a29b3ce4"
    hostname: ovn-compute-2.v1v0x.net
    Encap geneve
        ip: "10.100.7.22"
        options: {csum="true"}
    Encap vxlan
        ip: "10.100.7.22"
        options: {csum="true"}
    Port_Binding "6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69"
    Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
    Port_Binding "967b3f42-a881-4ac3-957f-dbf26756b2f2"
```

Lets set manualy higher priority to ovn-compute-1 to make it active gateway and then you will see (Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f) will move to ovn-compute-1

Setting ovn-compute-1 priority to 10

```
ovn-nbctl lrp-set-gateway-chassis lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f ed74f491-7389-4730-af3b-645874a660a8 10
```

Lets check Southbound db again. you can see i used --no-leader-only option because i found intra-2 is nolonger leader for southbound DB. You will noticed in following output Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f moved from ovn-compute-2 to ovn-compute-1 after setting up higher priority. 

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovn-sbctl --no-leader-only show
Chassis "ed74f491-7389-4730-af3b-645874a660a8"
    hostname: ovn-compute-1.v1v0x.net
    Encap vxlan
        ip: "10.100.7.21"
        options: {csum="true"}
    Encap geneve
        ip: "10.100.7.21"
        options: {csum="true"}
    Port_Binding cr-lrp-c1ff73df-5bac-40c0-9ba6-f792dea9a57f
Chassis "8a3ada6a-e062-49c2-abb7-0221a29b3ce4"
    hostname: ovn-compute-2.v1v0x.net
    Encap geneve
        ip: "10.100.7.22"
        options: {csum="true"}
    Encap vxlan
        ip: "10.100.7.22"
        options: {csum="true"}
    Port_Binding "6f23c1a2-5bf4-4fe6-8cb6-df206d2efe69"
    Port_Binding "967b3f42-a881-4ac3-957f-dbf26756b2f2"
```

Lets check infra-2 cluster role. I mentioned earlier that both DB clustering independent and one can be leader then possible other could be follower which you can see in following output. 

```
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound | grep Role
Role: leader
root@ovn-infra-2-neutron-ovn-northd-container-218bed1b:~# ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound | grep Role
Role: follower
```

#### OVN DVR or Distributed Gateway Routing

In above example active gateway schedule on one of chassis that means other chassis act like standby gateway in that design active gateway chassis could be bottleneck for network traffic. OVN support built in DVR support which you can enable with following snippet in user_variables.yml file and re-run neutron-playbook to deploy changes.

```
# DVR/Distributed L3 routing support
neutron_neutron_conf_overrides:
  ovn:
    enable_distributed_floating_ip: True
```

Thank you and enjoy!!!
