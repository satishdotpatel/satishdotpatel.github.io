---
title: "Openstack Kolla Deploy OVN Provider Driver For Octavia"
layout: post
date: 2022-12-21
image: /assets/images/2022-12-21-openstack-kolla-deploy-octavia-ovn/rope-walk.png
headerImage: true
tag:
- Openstack
- Kolla
- Octavia
- Loadbalancer
- OVN
category: blog
blog: true
author: Satish Patel
description: "Openstack Kolla Deploy OVN Provider Driver For Octavia"

---

Octavia is an open source, operator-scale load balancing solution designed to work with OpenStack. Octavia has integrated support for provider drivers where any third party Load Balancer driver can be integrated with Octavia. Functionality related to this has been developed in OVN and now OVN can now be supported as a provider driver for Octavia.

In previous blog I've deployed Amphora based loadbalancer which deploy haproxy LB as a VM instance. Using OVN based driver deploy LB without VMs.

Limitations of the OVN Provider Driver

### OVN provider - Advantages

The OVN Provider driver has a few advantages when used as a provider driver for Octavia over Amphora, like:

* OVN can be deployed without VMs, so there is no additional overhead as is required currently in Octavia when using the default Amphora driver.

* OVN Load Balancers can be deployed faster than default Load Balancers in Octavia (which use Amphora currently) because of no additional deployment requirement.

### OVN provider - Limitations

* OVN currently supports TCP and UDP, so Layer-7 based load balancing is not possible with the OVN provider driver.

* While Health Checks are now available in OVN, they are not currently implemented in OVN’s Provider Driver for Octavia.

* Currently, the OVN Provider driver supports a 1:1 protocol mapping between Listeners and associated Pools, i.e. a Listener which can handle TCP protocols can only be used with pools associated to the TCP protocol. Pools handling UDP protocols cannot be linked with TCP based Listeners.

* This limitation will be handled in an upcoming core OVN release.

* Mixed IPv4 and IPv6 members are not supported.

* Only the 'SOURCE_IP_PORT' load balancing algorithm is supported, others like 'ROUND_ROBIN' and 'LEAST_CONNECTIONS' are not currently supported.

* Octavia flavors are not supported.


### Enable OVN Privider Driver For Octavia

In global.yml (Default its enabled if you deployed OVN for neutron) 

```
octavia_provider_drivers: "ovn:OVN provider"
octavia_provider_agents: "ovn"
```

### Create Loabalancer using OVN Provider

Create loadbalancer (ovn-lb1 using option --provider ovn)

```
$ openstack loadbalancer create --vip-network-id demo-net --provider ovn --name ovn-lb1
$
$ openstack loadbalancer list
+--------------------------------------+---------+----------------------------------+-------------+---------------------+------------------+----------+
| id                                   | name    | project_id                       | vip_address | provisioning_status | operating_status | provider |
+--------------------------------------+---------+----------------------------------+-------------+---------------------+------------------+----------+
| 4a8b01ed-5dd1-4aee-bde8-b0568241e4eb | lb1     | 670dea6393824b3ca32426474848f0e1 | 10.0.0.178  | ACTIVE              | DEGRADED         | amphora  |
| f8e447fb-0474-4d17-b35a-2b1753fffbd9 | ovn-lb1 | 670dea6393824b3ca32426474848f0e1 | 10.0.0.113  | ACTIVE              | ONLINE           | ovn      |
+--------------------------------------+---------+----------------------------------+-------------+---------------------+------------------+----------+
```

Create loadbalancer listener (ovn-listener-lb1)

```
$ openstack loadbalancer listener create --protocol TCP --protocol-port 80 ovn-lb1 --name ovn-listener-lb1
$
$ openstack loadbalancer listener list
+--------------------------------------+--------------------------------------+------------------+----------------------------------+----------+---------------+----------------+
| id                                   | default_pool_id                      | name             | project_id                       | protocol | protocol_port | admin_state_up |
+--------------------------------------+--------------------------------------+------------------+----------------------------------+----------+---------------+----------------+
| 1f34c808-e229-4f5c-b362-1e4bdeb87907 | 3f7151fd-f3e5-4d15-8e7c-9d56e638326c | listener1        | 670dea6393824b3ca32426474848f0e1 | TCP      |            80 | True           |
| 199c73bf-2480-40d3-82a3-b920530e5155 | None                                 | ovn-listener-lb1 | 670dea6393824b3ca32426474848f0e1 | TCP      |            80 | True           |
+--------------------------------------+--------------------------------------+------------------+----------------------------------+----------+---------------+----------------+
```

Create pool (ovn-pool-lb1)

```
$ openstack loadbalancer pool create --protocol TCP --lb-algorithm SOURCE_IP_PORT --listener ovn-listener-lb1 --name ovn-pool-lb1
$
$ openstack loadbalancer pool list
+--------------------------------------+--------------+----------------------------------+---------------------+----------+----------------+----------------+
| id                                   | name         | project_id                       | provisioning_status | protocol | lb_algorithm   | admin_state_up |
+--------------------------------------+--------------+----------------------------------+---------------------+----------+----------------+----------------+
| 3f7151fd-f3e5-4d15-8e7c-9d56e638326c | pool1        | 670dea6393824b3ca32426474848f0e1 | ACTIVE              | TCP      | ROUND_ROBIN    | True           |
| c41ed8ba-7505-4006-8725-bca3f0971748 | ovn-pool-lb1 | 670dea6393824b3ca32426474848f0e1 | ACTIVE              | TCP      | SOURCE_IP_PORT | True           |
+--------------------------------------+--------------+----------------------------------+---------------------+----------+----------------+----------------+
```

Create or add members in pool (10.0.0.10 is webserver1)

```
$ openstack loadbalancer member create --address 10.0.0.10 --protocol-port 80 ovn-pool-lb1
$
$ openstack loadbalancer member list ovn-pool-lb1
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| id                                   | name | project_id                       | provisioning_status | address   | protocol_port | operating_status | weight |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| 61aaf325-e42f-4d83-b416-af0945129b36 |      | 670dea6393824b3ca32426474848f0e1 | ACTIVE              | 10.0.0.10 |            80 | NO_MONITOR       |      1 |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
```

Attach floating ip so we can access vip from outside. 

```
$ VIP_PORT_ID=`openstack loadbalancer show ovn-lb1 -c vip_port_id -f value`
$ FLOATING_IP_ID="984588f5-fdc5-435c-894e-b38a143f1178"
$
$ openstack floating ip set --port $VIP_PORT_ID $FLOATING_IP_ID
```

### Verify

In ovn db you can verify status of LB. 

```
$ ovn-nbctl list load_balancer
_uuid               : d37651dc-e0af-4dde-b819-a5a819faacc3
external_ids        : {enabled=True, listener_199c73bf-2480-40d3-82a3-b920530e5155="80:pool_c41ed8ba-7505-4006-8725-bca3f0971748", lr_ref=neutron-363f91d0-9d42-41a3-9bfb-3343f3f3489f, ls_refs="{\"neutron-bc801598-7eee-481e-bcc0-465179e4c4d6\": 2}", "neutron:member_status"="{\"61aaf325-e42f-4d83-b416-af0945129b36\": \"NO_MONITOR\"}", "neutron:vip"="10.0.0.113", "neutron:vip_fip"="10.73.1.171", "neutron:vip_port_id"="d9107bbf-56fb-499a-a7d5-9b2d4f44ba70", pool_c41ed8ba-7505-4006-8725-bca3f0971748="member_61aaf325-e42f-4d83-b416-af0945129b36_10.0.0.10:80_280864fc-4bc2-4b8a-85c8-318a6f8a46db"}
health_check        : []
ip_port_mappings    : {}
name                : "f8e447fb-0474-4d17-b35a-2b1753fffbd9"
protocol            : tcp
selection_fields    : [ip_dst, ip_src, tp_dst, tp_src]
vips                : {"10.0.0.113:80"="10.0.0.10:80", "10.73.1.171:80"="10.0.0.10:80"}
```

Verify, You can access web1 service using floating ip

```
$ curl http://10.73.1.171
```

There is no need for flavors (no VM is created), failovers (no need to recover a VM), or HA (no need to create extra VMs as in the ovn-octavia case the flows are injected in all the nodes, i.e., it is HA by default).

Enjoy!! 
