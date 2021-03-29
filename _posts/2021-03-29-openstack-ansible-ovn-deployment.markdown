---
title: "Openstack Ansible OVN deployment"
layout: post
date: 2021-03-19
image: /assets/images/2021-03-19-openstack-ansible-ovn-deployment/openstack-sdn.png
headerImage: true
tag:
- openstack
- openstack-ansible
- ovn
- SDN
- openvswitch
category: blog
blog: true
author: Satish Patel
description: "Openstack Ansible OVN deployment"

---

Open Virtual Network (OVN) is an Open vSwitch-based software-defined networking (SDN) solution for supplying network services to instances. I am hearing lots of good stuff about OVN (open virtual network) so thought let me give it a try on openstack-ansible and document whole process of deployment. In following lab i am going to use VMware virtual machines to run my OSA. I'm running following vms and using Ubuntu focal for my OS version. 

Assuming you already knows about how to deploy openstack using openstack-ansible and if you are new to openstack-ansible then check out my previous post https://satishdotpatel.github.io/build-openstack-cloud-using-openstack-ansible/

### Environment

* osa - Deployment host
* os-infra-1 - Single controller node
* os-compute-N - Two compute nodes 

### Network Setup

On VMware ESXi host machine i have created 4 vSwitch for br-host, br-mgmt, br-vxlan and br-vlan for my deployment to mimic production design.

os-infra-1 interface configuration

```
root@os-infra-1:~# cat /etc/netplan/00-network-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens192:
      dhcp4: no
    ens224:
      dhcp4: no
    ens256:
      dhcp4: no
    ens161:
      dhcp4: no

  bridges:
    br-host:
      interfaces: [ ens192 ]
      addresses: [ 10.30.40.2/16 ]
      gateway4: 10.30.0.1
      nameservers:
        addresses: [ 10.30.0.10, 10.30.0.11 ]
        search: [ example.com ]
    br-mgmt:
      interfaces: [ ens224 ]
      addresses: [ 172.30.40.2/24 ]
    br-vxlan:
      interfaces: [ ens256 ]
      addresses: [ 192.168.40.2/24 ]
    br-vlan:
      interfaces: [ ens161 ]
```

os-compute-1 interface configuration

```
root@os-compute-1:~# cat /etc/netplan/00-network-config.yaml
# This is the network config written by kickstart
network:
  version: 2
  renderer: networkd
  ethernets:
    ens192:
      dhcp4: no
    ens224:
      dhcp4: no
    ens256:
      dhcp4: no
    ens161:
      dhcp4: no

  bridges:
    br-host:
      interfaces: [ ens192 ]
      addresses: [ 10.30.40.3/16 ]
      gateway4: 10.30.0.1
      nameservers:
        addresses: [ 10.30.0.10, 10.30.0.11 ]
        search: [ example.com ]
    br-mgmt:
      interfaces: [ ens224 ]
      addresses: [ 172.30.40.3/24 ]
    br-vxlan:
      interfaces: [ ens256 ]
      addresses: [ 192.168.40.3/24 ]
    br-vlan:
      interfaces: [ ens161 ]
```

### Openstack-Ansible configuration 

/etc/openstack_deploy/openstack_user_config.yml

```
---
cidr_networks:
  container: 172.30.40.0/24
  tunnel: 192.168.40.0/24

used_ips:
  - "172.30.40.1"
  - "172.30.40.2"
  - "172.30.40.3"
  - "172.30.40.4"
  - "172.30.40.5"
  - "192.168.40.1"
  - "192.168.40.2"
  - "192.168.40.3"
  - "192.168.40.4"
  - "192.168.40.5"

global_overrides:
  external_lb_vip_address: 10.30.40.2
  internal_lb_vip_address: 172.30.40.2
  management_bridge: br-mgmt
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
        container_bridge: br-vxlan
        container_interface: eth10
        container_type: veth
        group_binds:
          - neutron_ovn_controller
        ip_from_q: tunnel
        net_name: geneve
        range: 1:1000
        type: geneve
    - network:
        container_bridge: br-provider
        container_type: veth
        group_binds:
          - neutron_ovn_controller
        net_name: vlan
        network_interface: br-vlan
        range: 101:200,301:400
        type: vlan

shared-infra_hosts:
  os-infra-1:
    ip: 172.30.40.2
repo-infra_hosts:
  os-infra-1:
    ip: 172.30.40.2
```

/etc/openstack_deploy/user_variables.yml

```
---
debug: false
apply_security_hardening: false
install_method: source
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

### Installation

Run following playbooks 

```
$ openstack-ansible setup-hosts.yml
$ openstack-ansible setup-infrastructure.yml
$ openstack-ansible setup-openstack.yml
```

### Validation

After successfully run of all playbook you will see following list containers on os-infra-1 node

```
root@os-infra-1:~# lxc-ls -f
NAME                                             STATE   AUTOSTART GROUPS            IPV4                      IPV6 UNPRIVILEGED
os-infra-1_galera_container-30b742a6             RUNNING 1         onboot, openstack 10.0.3.168, 172.30.40.57  -    false
os-infra-1_glance_container-4d0d43b4             RUNNING 1         onboot, openstack 10.0.3.125, 172.30.40.138 -    false
os-infra-1_horizon_container-dbd6f260            RUNNING 1         onboot, openstack 10.0.3.129, 172.30.40.28  -    false
os-infra-1_keystone_container-1c2acf25           RUNNING 1         onboot, openstack 10.0.3.225, 172.30.40.220 -    false
os-infra-1_memcached_container-fd52739c          RUNNING 1         onboot, openstack 10.0.3.163, 172.30.40.121 -    false
os-infra-1_neutron_ovn_northd_container-24eea9c2 RUNNING 1         onboot, openstack 10.0.3.240, 172.30.40.93  -    false
os-infra-1_neutron_server_container-49dba0f4     RUNNING 1         onboot, openstack 10.0.3.74, 172.30.40.12   -    false
os-infra-1_nova_api_container-11f9cc79           RUNNING 1         onboot, openstack 10.0.3.35, 172.30.40.209  -    false
os-infra-1_placement_container-4467b60b          RUNNING 1         onboot, openstack 10.0.3.215, 172.30.40.26  -    false
os-infra-1_rabbit_mq_container-95798546          RUNNING 1         onboot, openstack 10.0.3.133, 172.30.40.238 -    false
os-infra-1_repo_container-4dff68f5               RUNNING 1         onboot, openstack 10.0.3.140, 172.30.40.227 -    false
os-infra-1_utility_container-73ccc49d            RUNNING 1         onboot, openstack 10.0.3.239, 172.30.40.139 -    false
```

![<img>](/assets/images/2021-03-29-openstack-ansible-ovn-deployment/ovn-flow.png)

As you can see we have ovn_northd_container which containe northd services and north/south ovsdb databases, lets verify. 

```
root@os-infra-1:~# lxc-attach -n os-infra-1_neutron_ovn_northd_container-24eea9c2
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# systemctl | grep ovn
  ovn-central.service                                    loaded active exited    Open Virtual Network central components
  ovn-northd.service                                     loaded active running   Open Virtual Network central control daemon
  ovn-ovsdb-server-nb.service                            loaded active running   Open vSwitch database server for OVN Northbound database
  ovn-ovsdb-server-sb.service                            loaded active running   Open vSwitch database server for OVN Southbound database
```

You can view north and south database using ovn-nbctl and ovn-sbctl commands.

northbound, currently we don't have any router/switch/port that is why ovn-nbctl return nothing.

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-nbctl show
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# 
```

southbound, you can see two compute nodes as known as Chassis.  

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-sbctl show
Chassis "fb3ec9a5-4be5-40d8-8b8c-b5c9eff03fc4"
    hostname: os-compute-2.example.com
    Encap geneve
        ip: "192.168.40.4"
        options: {csum="true"}
    Encap vxlan
        ip: "192.168.40.4"
        options: {csum="true"}
Chassis "57c973ee-464a-40df-8436-1d046a69a671"
    hostname: os-compute-1.example.com
    Encap geneve
        ip: "192.168.40.3"
        options: {csum="true"}
    Encap vxlan
        ip: "192.168.40.3"
        options: {csum="true"}
```

Lets create routers/network and vms to see how does OVN handle them. 

Creating router
```
$ openstack router create router1
```
Creating network/subnet 
```
$ neutron net-create net98 --shared --provider:physical_network vlan --provider:network_type vlan --provider:segmentation_id 98
$ neutron subnet-create net98 192.168.1.0/24 --name sub98 --allocation-pool start=192.168.1.10,end=192.168.1.20 --gateway=192.168.1.1
$ openstack router add subnet router1 sub98
```
Creating flavor
```
openstack flavor create --id 1 --ram 1048  --vcpus 1 --disk 1  m1.small
```
Creating Security Groups
```
openstack security group create allow_all_traffic --description "Allow all traffic"
openstack security group rule create --proto tcp allow_all_traffic
openstack security group rule create --proto udp allow_all_traffic
openstack security group rule create --proto icmp allow_all_traffic
```
Download and upload cirros image
```
$ wget https://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img
$ glance image-create --name cirros --disk-format raw --container-format bare --property hw_vif_multiqueue_enabled=true --property hw_scsi_model=virtio-scsi --property hw_disk_bus=scsi --visibility public --file cirros-0.5.1-x86_64-disk.img --progress
```
Creating VMs
```
$ openstack server create --flavor m1.small --image cirros --nic net-id=net98 --security-group allow_all_traffic vm01
+-------------------------------------+-----------------------------------------------+
| Field                               | Value                                         |
+-------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                        |
| OS-EXT-AZ:availability_zone         |                                               |
| OS-EXT-SRV-ATTR:host                | None                                          |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                          |
| OS-EXT-SRV-ATTR:instance_name       |                                               |
| OS-EXT-STS:power_state              | NOSTATE                                       |
| OS-EXT-STS:task_state               | scheduling                                    |
| OS-EXT-STS:vm_state                 | building                                      |
| OS-SRV-USG:launched_at              | None                                          |
| OS-SRV-USG:terminated_at            | None                                          |
| accessIPv4                          |                                               |
| accessIPv6                          |                                               |
| addresses                           |                                               |
| adminPass                           | QEd8pYb63ojT                                  |
| config_drive                        |                                               |
| created                             | 2021-03-29T20:59:12Z                          |
| flavor                              | m1.small (1)                                  |
| hostId                              |                                               |
| id                                  | 4cd96b51-9ca4-4274-a6a5-18f88dca76f7          |
| image                               | cirros (1f4ee41a-3019-4bbb-a9e6-8b903d2675e0) |
| key_name                            | None                                          |
| name                                | vm01                                          |
| progress                            | 0                                             |
| project_id                          | 726d0445052941eb8541d8ffeaece7b1              |
| properties                          |                                               |
| security_groups                     | name='93812fbf-e660-43ce-bef0-46d24cd9d154'   |
| status                              | BUILD                                         |
| updated                             | 2021-03-29T20:59:12Z                          |
| user_id                             | 1929a3840a70454f8819b50121fdb25b              |
| volumes_attached                    |                                               |
+-------------------------------------+-----------------------------------------------+
```
It works!!!  
```
$ nova list
+--------------------------------------+------+--------+------------+-------------+--------------------+
| ID                                   | Name | Status | Task State | Power State | Networks           |
+--------------------------------------+------+--------+------------+-------------+--------------------+
| 4cd96b51-9ca4-4274-a6a5-18f88dca76f7 | vm01 | ACTIVE | -          | Running     | net98=192.168.1.13 |
+--------------------------------------+------+--------+------------+-------------+--------------------+
```
Lets verify northbound database after creating router/switch/port/vms. You can see in following output all the logical information of neutron network.

```
$ ovn-nbctl show
switch 6ffcc845-c1e8-4584-ac12-9faf3c40bef9 (neutron-218c7fd7-09bc-4340-bc24-0a90bcc2d45c) (aka net98)
    port dcabfa92-37dc-4f6c-9692-55de56a9c183
        type: router
        router-port: lrp-dcabfa92-37dc-4f6c-9692-55de56a9c183
    port provnet-c34cd1f3-94cd-49af-8d44-bcff771743e7
        type: localnet
        tag: 98
        addresses: ["unknown"]
    port 5de8dfba-5f48-4d8a-8d70-a21a8c542207
        type: localport
        addresses: ["fa:16:3e:d4:04:51 192.168.1.10"]
    port 594b06b0-c78f-4f9a-860e-f7724db9eeec
        addresses: ["fa:16:3e:34:ac:23 192.168.1.13"]
router 55927ae5-7625-4bc8-8289-0a7f0cd76003 (neutron-c4438f62-7b63-47d7-8da2-425700f4f89f) (aka router1)
    port lrp-dcabfa92-37dc-4f6c-9692-55de56a9c183
        mac: "fa:16:3e:bd:5e:31"
        networks: ["192.168.1.1/24"]
```

### L3 Routing

OVN support L3 routing so you don't need special Linux namespace or iptables for L3 agents. Lets verify, I am going to create new network net99 and will attach to router1 

```
$ neutron net-create net99 --shared --provider:physical_network vlan --provider:network_type vlan --provider:segmentation_id 99
$ neutron subnet-create net99 192.168.2.0/24 --name sub99 --allocation-pool start=192.168.2.10,end=192.168.2.20 --gateway=192.168.2.1
$ openstack router add subnet router1 sub99
```

Lets spin up new vm02 in net99 vlan 

```
$ openstack server create --flavor m1.small --image cirros --nic net-id=net99 --security-group allow_all_traffic vm02
```

This is what our network topology looks, Two networks connected using router1

![<img>](/assets/images/2021-03-29-openstack-ansible-ovn-deployment/network-topo.png)

Lets verify ping from vm01 to vm02 to check L3 routing

```
login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
vm01 login: cirros
Password:
$ hostname
vm01
$ ping 192.168.2.19 -c 3
PING 192.168.2.19 (192.168.2.19): 56 data bytes
64 bytes from 192.168.2.19: seq=0 ttl=63 time=1.811 ms
64 bytes from 192.168.2.19: seq=1 ttl=63 time=1.233 ms
64 bytes from 192.168.2.19: seq=2 ttl=63 time=0.669 ms
```  

Enjoy your openstack SDN network!!! Next i am planning to create 3 node controller for high redendency. 
