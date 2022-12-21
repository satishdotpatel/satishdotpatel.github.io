---
title: "Openstack Kolla deploy Octavia Loadbalancer"
layout: post
date: 2022-12-20
image: /assets/images/2022-12-20-openstack-kolla-deploy-octavia/lb1.png
headerImage: true
tag:
- Openstack
- Kolla
- Octavia
- Loadbalancer
category: blog
blog: true
author: Satish Patel
description: "Openstack Kolla deploy Octavia Loadbalancer"

---

Octavia is an open source, operator-scale load balancing solution designed to work with OpenStack. I will show quick way to setup octavia LB using openstack kolla. I'm using tenant base network for octavia heath manager which is not recommonded for production deployment. Please use this setup for dev/test environment. 

### Enable Octavia 

In global.yml 

```
enable_octavia: "yes"
octavia_network_type: "tenant"
```

Run playbook to deploy. Following playbooks will create octavia certs, network, flavor, security group and ssh key.

```
$ kolla-ansible octavia-certificates
$ kolla-ansible -i multinode deploy --tags octavia,common,horizon
```

Run post-deploy playbook to create octavia-openrc.sh file

```
$ kolla-ansible post-deploy 
```

Load ovtavia-openrc.sh

```
$ . /etc/kolla/octavia-openrc.sh
```

### Verify deployment

Verify network/flavor/keys

```
$ openstack network list
$ openstack flavor list
$ openstack keypair list
$ openstack security group list
```

Verify o-hm0 interface on controller nodes. This port attach to br-int bridge.

```
$ ip a show dev o-hm0
35: o-hm0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1442 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:16:3e:27:dd:0d brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.177/24 brd 10.1.0.255 scope global dynamic o-hm0
       valid_lft 39146sec preferred_lft 39146sec
    inet6 fe80::f816:3eff:fe27:dd0d/64 scope link
       valid_lft forever preferred_lft forever
```

Neutron port 

```
$ openstack port list | grep octavia-listen-port
| b26cf9d5-c0b3-4977-89b4-52bf6479593e | octavia-listen-port-kolla-infra-1.example.com        | fa:16:3e:27:dd:0d | ip_address='10.1.0.177', subnet_id='29f3c75b-029d-4a81-af3f-b536dc29a123'  | ACTIVE |
```

Let's Download amphora image and import to glance.

```
$ wget https://minio.services.osism.tech/openstack-octavia-amphora-image/octavia-amphora-haproxy-wallaby.qcow2
$
$ openstack image create amphora-x64-haproxy --container-format bare --disk-format qcow2 --private --tag amphora --file octavia-amphora-haproxy-wallaby.qcow2 --property hw_architecture='x86_64' --property hw_rng_model=virtio
```

### Creating a TCP load balancer

Create a load balancer (lb1) on the public subnet (demo-subnet).

```
$ openstack loadbalancer create --name lb1 --vip-subnet-id demo-subnet
```

Verify the state of the load balancer, make sure status are ONLINE & ACTIVE

```
$ 	openstack loadbalancer show lb1
```

Create a TCP listener (listener1) on the specified port (80) 

```
$ openstack loadbalancer listener create --name listener1 --protocol TCP --protocol-port 80 lb1
```

Create a pool (pool1) and make it the default pool for the listener.

```
$ openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol TCP
```

Create a health monitor on the pool (pool1) that connects to the back-end servers and probes the TCP service port.

```
$ openstack loadbalancer healthmonitor create --delay 15 --max-retries 4 --timeout 10 --type TCP pool1
```

Add the back-end servers (10.0.0.10 and 10.0.0.35) on the private subnet (demo-subnet) to the pool.

```
$ openstack loadbalancer member create --subnet-id demo-subnet --address 10.0.0.10 --protocol-port 80 pool1
$ openstack loadbalancer member create --subnet-id demo-subnet --address 10.0.0.35 --protocol-port 80 pool1
```

Check status of members

```
$ openstack loadbalancer member list pool1
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| id                                   | name | project_id                       | provisioning_status | address   | protocol_port | operating_status | weight |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| cf58f699-9be3-49c4-ac82-a688cce02e6f |      | 670dea6393824b3ca32426474848f0e1 | ACTIVE              | 10.0.0.10 |            80 | ONLINE           |      1 |
| 5d25c3ef-21e7-499d-b50a-3beab8eed60a |      | 670dea6393824b3ca32426474848f0e1 | ACTIVE              | 10.0.0.35 |            80 | ONLINE           |      1 |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
```

### Attach floating ip to lb1

Create floating ip

```
$ openstack floating ip create public1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2022-12-21T04:24:47Z                 |
| description         |                                      |
| dns_domain          |                                      |
| dns_name            |                                      |
| fixed_ip_address    | None                                 |
| floating_ip_address | 10.73.1.168                          |
| floating_network_id | 769f5034-0729-492e-b22e-400ab1c6991b |
| id                  | c79726a7-68ad-44e4-b5f1-a4a443768ac4 |
| name                | 10.73.1.168                          |
```

Find vip_port_id of lb1

```
$ openstack loadbalancer show lb1 | grep vip_port_id
| vip_port_id         | b8a1ce2d-09d1-4109-8f9b-53ffc8318bcb |
```

Attach floating ip ID to vip_port_id (Example: openstack floating ip set --port <load_balancer_vip_port_id> <floating_ip_id> )

```
$ openstack floating ip set --port b8a1ce2d-09d1-4109-8f9b-53ffc8318bcb c79726a7-68ad-44e4-b5f1-a4a443768ac4
```

Now, you can access LB using http://10.73.1.168

Check stats of lb1

```
$ openstack loadbalancer stats show lb1
+--------------------+-------+
| Field              | Value |
+--------------------+-------+
| active_connections | 0     |
| bytes_in           | 525   |
| bytes_out          | 78211 |
| request_errors     | 0     |
| total_connections  | 7     |
+--------------------+-------+
```

Enjoy! 
