---
title: "Openstack Ansible OVN deployment (external connectivity) - Part-2"
layout: post
date: 2021-04-08
image: /assets/images/2021-03-29-openstack-ansible-ovn-deployment/openstack-sdn.png
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
description: "Openstack Ansible OVN deployment (external connectivity) - Part-2"

---

In a previous post I have explained how to deploy OVN using openstack-ansible in multi-node deployment. In this post I will show you the external connectivity of your vm to the provider network. 
OVN has built-in support of L3 function which doesn't require any Linux namespace to run L3 routers. There are two way you can connect your VMs to external network 

* Non Distributed FIP (Centralized design)
* Distributed FIP (DVR design)

By default OSA deploy centralized router where they run in active-backup mode on each compute nodes, lets see through example.

This is my private network
```
$ openstack network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| b96364e8-340b-4024-97ff-2796be2acab6 | private | ea68bd5b-7d65-4204-a414-b166a8bed5d3 |
+--------------------------------------+---------+--------------------------------------+
```

Lets create public network for internet access.

```
$ openstack network create public --external --provider-network-type vlan --provider-segment 40 --provider-physical-network vlan
$ openstack subnet create --network public --subnet-range 216.163.208.0/24 --allocation-pool start=216.163.208.10,end=216.163.208.20 --no-dhcp --gateway 216.163.208.1 public-subnet
``` 

Create router name router1

```
$ openstack router create router1
```

Attach private-subnet and public network as external-gateway to router1. Here i am specifically assigning 216.163.208.2 ip address to my router1 otherwise it will pick randomly which i don't like.

```
$ openstack router add subnet router1 private-subnet
$ openstack router set --external-gateway public router1 --fixed-ip subnet=public-subnet,ip-address=216.163.208.2
```

Verify your router status and external_fixed_ips 

```
openstack router show router1 --fit-width
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                    |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                       |
| availability_zone_hints |                                                                                                                                                                          |
| availability_zones      |                                                                                                                                                                          |
| created_at              | 2021-04-08T17:20:25Z                                                                                                                                                     |
| description             |                                                                                                                                                                          |
| external_gateway_info   | {"network_id": "8e9db0f7-576c-47c4-96a3-a4c37fb9385c", "external_fixed_ips": [{"subnet_id": "0236dd5f-5680-4885-9e4c-32533cd2423e", "ip_address": "216.163.208.2"}],     |
|                         | "enable_snat": true}                                                                                                                                                     |
| flavor_id               | None                                                                                                                                                                     |
| id                      | f2db247f-1bee-478c-a5aa-aa090d74dbc7                                                                                                                                     |
| interfaces_info         | [{"port_id": "e8d9d7bf-a434-4b96-8a6c-91938f28953d", "ip_address": "172.168.0.1", "subnet_id": "ea68bd5b-7d65-4204-a414-b166a8bed5d3"}]                                  |
| name                    | router1                                                                                                                                                                  |
| project_id              | 726d0445052941eb8541d8ffeaece7b1                                                                                                                                         |
| revision_number         | 4                                                                                                                                                                        |
| routes                  |                                                                                                                                                                          |
| status                  | ACTIVE                                                                                                                                                                   |
| tags                    |                                                                                                                                                                          |
| updated_at              | 2021-04-08T17:22:27Z                                                                                                                                                     |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Now lets build two vms (vm01 and vm02) in private network

```
$ openstack server create --flavor m1.small --image cirros --nic net-id=private --security-group allow_all_traffic vm01
$ openstack server create --flavor m1.small --image cirros --nic net-id=private --security-group allow_all_traffic vm02
```

make sure both vm01 and vm02 endup on two different compute nodes, if not then delete and re-create. :) 

```
$ openstack server list --all-projects --host os-compute-1.foo.net
+--------------------------------------+------+--------+-----------------------+--------+----------+
| ID                                   | Name | Status | Networks              | Image  | Flavor   |
+--------------------------------------+------+--------+-----------------------+--------+----------+
| fb3f4b21-b21b-4368-9954-baf2a07cf20c | vm01 | ACTIVE | private=172.168.0.102 | cirros | m1.small |
+--------------------------------------+------+--------+-----------------------+--------+----------+
$ openstack server list --all-projects --host os-compute-2.foo.net
+--------------------------------------+------+--------+-----------------------+--------+----------+
| ID                                   | Name | Status | Networks              | Image  | Flavor   |
+--------------------------------------+------+--------+-----------------------+--------+----------+
| 2746e792-f2ba-4e69-918d-b7be5e724ce6 | vm02 | ACTIVE | private=172.168.0.226 | cirros | m1.small |
+--------------------------------------+------+--------+-----------------------+--------+----------+
```

Lets verify vm external connectivity, so far so good. 

```
root@os-compute-1:~# virsh console instance-00000021
Connected to domain instance-00000021
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
vm01 login: cirros
Password:
$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=112 time=14.658 ms
64 bytes from 8.8.8.8: seq=1 ttl=112 time=8.774 ms
64 bytes from 8.8.8.8: seq=2 ttl=112 time=8.167 ms
```

![<img>](/assets/images/2021-03-29-openstack-ansible-ovn-deployment/ovn-l3.png)

Lets check where OVN schedule our L3 router. Go to neutron_ovn_northd_container and run `ovn-nbctl list Gateway_Chassis` command and pay attention to chassis_name and priority flags. as you can see our router1 scheduled on both computer nodes but computer-1 has lower priority so currently that router is in active mode and second one in backup. you can verify chassis_name UUID running `ovn-sbctl show` command.

```
root@os-infra-1:~# lxc-attach -n os-infra-1_neutron_ovn_northd_container-24eea9c2

root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-nbctl list Gateway_Chassis
_uuid               : 2962c864-be70-4570-a30d-291a466f97ec
chassis_name        : "57c973ee-464a-40df-8436-1d046a69a671"
external_ids        : {}
name                : lrp-9d32dec2-cae7-4544-bbae-c17840deaea5_57c973ee-464a-40df-8436-1d046a69a671
options             : {}
priority            : 1

_uuid               : d00c0e0f-0672-42ae-9b6b-55e7b10ac6d1
chassis_name        : "fb3ec9a5-4be5-40d8-8b8c-b5c9eff03fc4"
external_ids        : {}
name                : lrp-9d32dec2-cae7-4544-bbae-c17840deaea5_fb3ec9a5-4be5-40d8-8b8c-b5c9eff03fc4
options             : {}
priority            : 2
```

You can run following experiment, go to vm02 and run continue ping to 8.8.8.8 while ping is running try to shutdown br-vxlan (overlay) interface on compute-1 and you will see 8.8.8.8 ping will stopped, that means compute-2 vms using compute-1 l3 router for internet connectivity, just like legacy deployment where your network node is SNAT gateway for all traffic. 

Lets attach FIP to vm02 and see how traffic is flowing.

```
$ openstack floating ip create public
$ openstack server add floating ip vm02 216.163.208.15
```

Let verify dnat_and_snat tables in OVN, This is very important when external_mac field is empty that means your traffic going to use centralized gateway for routing. 

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-nbctl find NAT type=dnat_and_snat
_uuid               : d157cf58-beb1-43de-a8e2-7d58eb5a52ee
external_ids        : {"neutron:fip_external_mac"="fa:16:3e:9a:62:c1", "neutron:fip_id"="9a134be8-41e7-41f7-9d31-53a8dac2bda1", "neutron:fip_network_id"="8e9db0f7-576c-47c4-96a3-a4c37fb9385c", "neutron:fip_port_id"="e3dc6cb7-1ba7-448f-a069-ac51f72b1cda", "neutron:revision_number"="2", "neutron:router_name"=neutron-f2db247f-1bee-478c-a5aa-aa090d74dbc7}
external_ip         : "216.163.208.15"
external_mac        : []
logical_ip          : "172.168.0.226"
logical_port        : "e3dc6cb7-1ba7-448f-a069-ac51f72b1cda"
options             : {}
type                : dnat_and_snat
```

As you can see following NAT table where vm02 has EXTERNAL_IP for static NAT 216.163.208.15 but it doesn't have EXTERNAL_MAC that means your all traffic will flow via compute-1 which is active gateway router 

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-nbctl lr-nat-list neutron-f2db247f-1bee-478c-a5aa-aa090d74dbc7
TYPE             EXTERNAL_IP        LOGICAL_IP            EXTERNAL_MAC         LOGICAL_PORT
dnat_and_snat    216.163.208.15     172.168.0.226
snat             216.163.208.2      172.168.0.0/24
```

Now ping continue ping 216.163.208.15 from public network and try to shutdown br-vxlan interface on compute-1 and you will noticed you will loose connectivity because your router is active on compute-1 node. if you stop ovn-controller service on compute-1 then you will notice computer-2 will get active and start service external traffic.

```
root@border-gateway:~# ping 216.163.208.15
PING 216.163.208.15 (216.163.208.15) 56(84) bytes of data.
64 bytes from 216.163.208.15: icmp_seq=1 ttl=63 time=2.92 ms
64 bytes from 216.163.208.15: icmp_seq=2 ttl=63 time=1.07 ms
64 bytes from 216.163.208.15: icmp_seq=3 ttl=63 time=0.570 ms
64 bytes from 216.163.208.15: icmp_seq=4 ttl=63 time=0.525 ms
```

![<img>](/assets/images/2021-03-29-openstack-ansible-ovn-deployment/non-distributed-fip.png)


NOTES: This solution is great and provide redendency but not scalable for network performance because your compute-1 will be bottlneck for your network traffic.

### Distributed Gateway (a.k.a DVR)

OVN does support DVR and for that you have to configure following option on your neutron-server (neutron.conf file)

```
[ovn]
enable_distributed_floating_ip = True
```

After making above change lets remove floating ip from vm02 and re-add floating ip again.

```
$ openstack server remove floating ip vm02 216.163.208.15
$ openstack server add floating ip vm02 216.163.208.15
```

Lets verify dnat_and_snat table again. Voila!!! look at EXTERNAL_MAC field. 

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovn-nbctl lr-nat-list neutron-f2db247f-1bee-478c-a5aa-aa090d74dbc7
TYPE             EXTERNAL_IP        LOGICAL_IP            EXTERNAL_MAC         LOGICAL_PORT
dnat_and_snat    216.163.208.15     172.168.0.226         fa:16:3e:9a:62:c1    e3dc6cb7-1ba7-448f-a069-ac51f72b1cda
snat             216.163.208.2      172.168.0.0/24
```

Now ping 216.163.208.15 from outside world and try to shutdown br-vxlan interface on compute-1 node and you will see no impact because now your traffic going in/out using local compute node like true DVR, at this point all compute nodes are service active router role without single point of failure and you will get full network bandwidth for your vms without bottleneck. 

![<img>](/assets/images/2021-03-29-openstack-ansible-ovn-deployment/distributed-fip.png)
