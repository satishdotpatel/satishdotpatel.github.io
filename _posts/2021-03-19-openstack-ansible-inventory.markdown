---
title: "Openstack Ansible Manage Inventory"
layout: post
date: 2021-03-19
image: /assets/images/2021-03-19-openstack-ansible-inventory/openstack-ansible.png
headerImage: true
tag:
- openstack
- openstack-ansible
category: blog
blog: true
author: Satish Patel
description: "Openstack Ansible Manage Inventory"

---

Openstack-ansible has cool way to manage compute inventory so i am going to share what i am using to group compute hosts by type in basic example. 

We have two type or compute nodes.

* General computes
* SR-IOV computes

Lets two files inside /etc/openstack_deplpoy/env.d/ directoy

Group name gen_hosts for general computes
```
[root@ostack-osa ~]# cat /etc/openstack_deploy/env.d/gen_hosts.yml
# custom inventory group to traget general computes
gen_hosts:
  belongs_to:
    - hosts
``` 
Group name sriov_hosts for SR-IOV computes
```
[root@ostack-osa ~]# cat /etc/openstack_deploy/env.d/sriov_hosts.yml
# custom inventory group to traget sriov computes
sriov_hosts:
  belongs_to:
    - hosts
```

Lets add compute nodes in each respective group in /etc/openstack_deploy/conf.d/nova.yaml file.

```
[root@ostack-osa ~]# cat /etc/openstack_deploy/conf.d/nova.yml
---
# nova api, conductor, scheduler
compute-infra_hosts:
  ostack-api-1-1:
    ip: 10.65.0.21
  ostack-api-1-2:
    ip: 10.65.0.22
  ostack-api-1-3:
    ip: 10.65.0.23

# General compute group
gen_hosts:
  ostack-comp-gen-1-1: &_ostack-comp-gen-1-1_
    ip: 10.65.1.1
  ostack-comp-gen-1-2: &_ostack-comp-gen-1-2_
    ip: 10.65.1.2
  ostack-comp-gen-1-3: &_ostack-comp-gen-1-3_
    ip: 10.65.1.3
  ostack-comp-gen-1-4: &_ostack-comp-gen-1-4_
    ip: 10.65.1.4
  ostack-comp-gen-1-5: &_ostack-comp-gen-1-5_
    ip: 10.65.1.5
  ostack-comp-gen-1-6: &_ostack-comp-gen-1-6_
    ip: 10.65.1.6
  ostack-comp-gen-1-7: &_ostack-comp-gen-1-7_
    ip: 10.65.1.7
  ostack-comp-gen-1-8: &_ostack-comp-gen-1-8_
    ip: 10.65.1.8
  ostack-comp-gen-1-9: &_ostack-comp-gen-1-9_
    ip: 10.65.1.9
  ostack-comp-gen-1-10: &_ostack-comp-gen-1-10_
    ip: 10.65.1.10
  ostack-comp-gen-1-11: &_ostack-comp-gen-1-11_
    ip: 10.65.1.11
  ostack-comp-gen-1-12: &_ostack-comp-gen-1-12_
    ip: 10.65.1.12


# SR-IOV compute group
sriov_hosts:
  ostack-comp-sriov-1-1: &_ostack-comp-sriov-1-1_
    ip: 10.65.3.1
  ostack-comp-sriov-1-2: &_ostack-comp-sriov-1-2_
    ip: 10.65.3.2
  ostack-comp-sriov-1-3: &_ostack-comp-sriov-1-3_
    ip: 10.65.3.3
  ostack-comp-sriov-1-4: &_ostack-comp-sriov-1-4_
    ip: 10.65.3.4
  ostack-comp-sriov-1-5: &_ostack-comp-sriov-1-5_
    ip: 10.65.3.5
  ostack-comp-sriov-1-6: &_ostack-comp-sriov-1-6_
    ip: 10.65.3.6
  ostack-comp-sriov-1-7: &_ostack-comp-sriov-1-7_
    ip: 10.65.3.7
  ostack-comp-sriov-1-8: &_ostack-comp-sriov-1-8_
    ip: 10.65.3.8
  ostack-comp-sriov-1-9: &_ostack-comp-sriov-1-9_
    ip: 10.65.3.9
  ostack-comp-sriov-1-10: &_ostack-comp-sriov-1-10_
    ip: 10.65.3.10

# All compute nodes
compute_hosts: &_compute_all_
  # General compute nodes
  ostack-comp-gen-1-1: *_ostack-comp-gen-1-1_
  ostack-comp-gen-1-2: *_ostack-comp-gen-1-2_
  ostack-comp-gen-1-3: *_ostack-comp-gen-1-3_
  ostack-comp-gen-1-4: *_ostack-comp-gen-1-4_
  ostack-comp-gen-1-5: *_ostack-comp-gen-1-5_
  ostack-comp-gen-1-6: *_ostack-comp-gen-1-6_
  ostack-comp-gen-1-7: *_ostack-comp-gen-1-7_
  ostack-comp-gen-1-8: *_ostack-comp-gen-1-8_
  ostack-comp-gen-1-9: *_ostack-comp-gen-1-9_
  ostack-comp-gen-1-10: *_ostack-comp-gen-1-10_
  ostack-comp-gen-1-11: *_ostack-comp-gen-1-11_
  ostack-comp-gen-1-12: *_ostack-comp-gen-1-12_

  # SR-IOV compute nodes
  ostack-comp-sriov-1-1: *_ostack-comp-sriov-1-1_
  ostack-comp-sriov-1-2: *_ostack-comp-sriov-1-2_
  ostack-comp-sriov-1-3: *_ostack-comp-sriov-1-3_
  ostack-comp-sriov-1-4: *_ostack-comp-sriov-1-4_
  ostack-comp-sriov-1-5: *_ostack-comp-sriov-1-5_
  ostack-comp-sriov-1-6: *_ostack-comp-sriov-1-6_
  ostack-comp-sriov-1-7: *_ostack-comp-sriov-1-7_
  ostack-comp-sriov-1-8: *_ostack-comp-sriov-1-8_
  ostack-comp-sriov-1-9: *_ostack-comp-sriov-1-9_
  ostack-comp-sriov-1-10: *_ostack-comp-sriov-1-10_
```

Now regenerate inventory or when running an Ansible command (such as ansible, ansible-playbook or openstack-ansible) Ansible automatically executes the dynamic_inventory.py script and use its output as inventory

```
# from the root folder of cloned OpenStack-Ansible repository
inventory/dynamic_inventory.py --config /etc/openstack_deploy/
```

You can verify your update inventory using following command.

```
...output ommitted...
| gen_hosts                            | ostack-phx-comp-gen-1-1                              |
|                                      | ostack-phx-comp-gen-1-2                              |
|                                      | ostack-phx-comp-gen-1-3                              |
|                                      | ostack-phx-comp-gen-1-4                              |
|                                      | ostack-phx-comp-gen-1-5                              |
|                                      | ostack-phx-comp-gen-1-6                              |
|                                      | ostack-phx-comp-gen-1-7                              |
|                                      | ostack-phx-comp-gen-1-8                              |
|                                      | ostack-phx-comp-gen-1-9                              |
|                                      | ostack-phx-comp-gen-1-10                             |
|                                      | ostack-phx-comp-gen-1-11                             |
|                                      | ostack-phx-comp-gen-1-12                             |
| sriov_hosts                          | ostack-phx-comp-sriov-1-1                            |
|                                      | ostack-phx-comp-sriov-1-2                            |
|                                      | ostack-phx-comp-sriov-1-3                            |
|                                      | ostack-phx-comp-sriov-1-4                            |
|                                      | ostack-phx-comp-sriov-1-5                            |
|                                      | ostack-phx-comp-sriov-1-6                            |
|                                      | ostack-phx-comp-sriov-1-7                            |
|                                      | ostack-phx-comp-sriov-1-8                            |
|                                      | ostack-phx-comp-sriov-1-9                            |
|                                      | ostack-phx-comp-sriov-1-10                           |
...output ommitted...
```

Now you can apply or compute type configuration to target group to computes nodes like in following example i have applied following configuration to only gen_hosts group.

```
[root@ostack-osa ~]# cat /etc/openstack_deploy/group_vars/gen_hosts.yml
---
## No overcommit CPU
nova_cpu_allocation_ratio: 1.0
## custom setting of nova
nova_nova_conf_overrides:
  DEFAULT:
    heal_instance_info_cache_interval: 300
    rpc_response_timeout: 120
  libvirt:
    cpu_mode: host-passthrough
  oslo_messaging_rabbit:
    rabbit_retry_interval: 20
    rabbit_retry_backoff: 3
    rabbit_interval_max: 60
    rabbit_transient_queues_ttl: 300
    rabbit_qos_prefetch_count: 100
    rpc_conn_pool_size: 300
  oslo_messaging_notifications:
    driver: noop
## neutron
neutron_linuxbridge_agent_ini_overrides:
  linux_bridge:
    physical_interface_mappings: vlan:br-vlan
```

You can run ad-hoc ansible command to target specific group also like following example

```
$ ansible gen_hosts -m shell -a "hostname" 
``` 

Enjoy! Your awesome inventory \o/ 

