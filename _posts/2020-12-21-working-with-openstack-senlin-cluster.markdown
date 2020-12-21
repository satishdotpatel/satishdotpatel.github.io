---
title: "Working with Openstack Senlin clustering"
layout: post
date: 2020-11-21
image: /assets/images/2020-12-21-working-with-openstack-senlin-cluster/cluster.png
headerImage: true
tag:
- senlin
- openstack-ansible
- autoscaling
- openstack
category: blog
blog: true
author: Satish Patel
description: "Working with Openstack Senlin clustering"

---

Senlin is a clustering service for OpenStack clouds. It creates and operates clusters of homogeneous objects exposed by other OpenStack services. The goal is to make orchestration of collections of similar objects easier. In earlier post i have convered basic of senlin. [openstack cloud autoscaling with senline](https://satishdotpatel.github.io/openstack-senlin-autoscaling/)

For this post i have used openstack-ansible to deploy senlin clustering services. You can check out here [Openstack-Ansible Documentation](https://docs.openstack.org/openstack-ansible/latest/) 

Lets assume senlin clustering services is up and running at this point. 

### Getting start

Verify senlin is functional using following command.

```
$ openstack cluster build info
+--------+---------------------+
| Field  | Value               |
+--------+---------------------+
| api    | {                   |
|        |   "revision": "1.0" |
|        | }                   |
| engine | {                   |
|        |   "revision": "1.0" |
|        | }                   |
+--------+---------------------+
```

Lets create first cluster before that we need to create server profile call my-profile.yml.

```
$ cat my-profile.yml
type: os.nova.server
version: 1.0
properties:
  name: senlin-vm
  flavor: m1.small
  image: "cirros"
  key_name: osa-key
  networks:
   - network: net_vlan69
     security_groups:
       - allow_all_traffic
  metadata:
    test_key: test_value
  user_data: |
    #!/bin/sh
    echo 'hello, world' > /tmp/test_file
```

Create first server profile 

```
$ openstack cluster profile create --spec-file my-profile.yml server-profile
$
$ openstack cluster profile list
+----------+----------------+--------------------+----------------------+
| id       | name           | type               | created_at           |
+----------+----------------+--------------------+----------------------+
| 5190dd3a | server-profile | os.nova.server-1.0 | 2020-12-21T21:06:23Z |
+----------+----------------+--------------------+----------------------+
```

Create cluster my-cluster using server-profile

* desired-capacity: Spin up number of vms and maintain that count.
* min-size: minimum number of vms. when you scale down cluster this number come in play
* max-size: maximum number of vms you can scale up in this cluster. 

Example:
![<img>](/assets/images/2020-12-21-working-with-openstack-senlin-cluster/asg-diagram.png)

```
$ openstack cluster create --profile server-profile --desired-capacity 2 --min-size 1 --max-size 3 my-cluster
$
$ openstack cluster list
+----------+--------------+----------+----------------------+----------------------+
| id       | name         | status   | created_at           | updated_at           |
+----------+--------------+----------+----------------------+----------------------+
| d67fa276 | my-cluster   | ACTIVE   | 2020-12-21T21:09:19Z | 2020-12-21T21:09:19Z |
+----------+--------------+----------+----------------------+----------------------+
```

Verify cluster vms (as you can see cluster spin up desired-capacity count) 

```
$ nova list
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| ID                                   | Name      | Status | Task State | Power State | Networks               |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.254 |
| 6f689f96-9b6e-448f-b3b3-82e08ef78013 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.150 |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
```

### Cluster auto-healing 

auto-healing is really a good feature where it will auto create vms if any reason they die. for that first we need to create auto-healing policy and attached to our cluster. 

```
$ cat policy.yml
type: senlin.policy.health
version: 1.1
description: A policy for maintaining node health from a cluster.
properties:
  detection:
    # Number of seconds between two adjacent checking
    interval: 60

    detection_modes:
      # Type for health checking, valid values include:
      # NODE_STATUS_POLLING, NODE_STATUS_POLL_URL, LIFECYCLE_EVENTS
      - type: NODE_STATUS_POLLING

  recovery:
    # Action that can be retried on a failed node, will improve to
    # support multiple actions in the future. Valid values include:
    # REBOOT, REBUILD, RECREATE
    actions:
      - name: RECREATE
```

Create policy 

```
$ openstack cluster policy create --spec-file policy.yml my-policy
$
$ openstack cluster policy list
+----------+-----------+--------------------------+----------------------------+
| id       | name      | type                     | created_at                 |
+----------+-----------+--------------------------+----------------------------+
| 66f6a12f | my-policy | senlin.policy.health-1.1 | 2020-12-21T21:22:38.000000 |
+----------+-----------+--------------------------+----------------------------+
```

Attach policy to my-cluster 

```
$ openstack cluster policy attach --policy my-policy my-cluster
Request accepted by action: 25e1a4fc-90aa-4d1c-b4dc-d3a029bebc94
$
```

Verify policy binding with my-cluster

```
$ openstack cluster policy binding show --policy my-policy my-cluster
```

Lets try to delete vms to see if cluster auto-healing instances or not. 

```
$ nova list
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| ID                                   | Name      | Status | Task State | Power State | Networks               |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.254 |
| 6f689f96-9b6e-448f-b3b3-82e08ef78013 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.150 |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
$
$ nova delete 6f689f96-9b6e-448f-b3b3-82e08ef78013
Request to delete server 6f689f96-9b6e-448f-b3b3-82e08ef78013 has been accepted.
```

Verify cluster status after deleting vm, we have single vm in cluster and desire capacity set to two. 

```
$ nova list
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| ID                                   | Name      | Status | Task State | Power State | Networks               |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.254 |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
```

verify cluster members status, you can see both member showing ACTIVE because health check service still think they are good.

```
$ openstack cluster members list my-cluster
+----------+---------------+-------+--------+-------------+----------------------+
| id       | name          | index | status | physical_id | created_at           |
+----------+---------------+-------+--------+-------------+----------------------+
| efe2c16d | node-YUuUu1di |     1 | ACTIVE | 6f689f96    | 2020-12-21T21:09:19Z |
| efe74d2b | node-z6reqO0s |     2 | ACTIVE | 3e365a7c    | 2020-12-21T21:09:17Z |
+----------+---------------+-------+--------+-------------+----------------------+
```

If you re-run member list command after 45 second you will see cluster status get RECOVERING

```
$ openstack cluster members list my-cluster
+----------+---------------+-------+------------+-------------+----------------------+
| id       | name          | index | status     | physical_id | created_at           |
+----------+---------------+-------+------------+-------------+----------------------+
| efe2c16d | node-YUuUu1di |     1 | RECOVERING | 6f689f96    | 2020-12-21T21:09:19Z |
| efe74d2b | node-z6reqO0s |     2 | ACTIVE     | 3e365a7c    | 2020-12-21T21:09:17Z |
+----------+---------------+-------+------------+-------------+----------------------+
```

Same time you will see nova building new vm

```
$ nova list
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| ID                                   | Name      | Status | Task State | Power State | Networks               |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.254 |
| 4a012717-d2c3-47cb-b91f-d4b2dfac7908 | senlin-vm | BUILD  | scheduling | NOSTATE     |                        |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
```

### Cluster scaling up and down 

Expanding cluster size

```
$ openstack cluster expand my-cluster
```

Verify, It has started create new instance so now we have 3 instances in cluster ( our max cluster size is 3 so we can't add 4th instance) 

```
$ nova list
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| ID                                   | Name      | Status | Task State | Power State | Networks               |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
| 39b211e8-4794-4bb6-841f-d9f83d7a60a0 | senlin-vm | BUILD  | spawning   | NOSTATE     |                        |
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.254 |
| 4a012717-d2c3-47cb-b91f-d4b2dfac7908 | senlin-vm | ACTIVE | -          | Running     | net_vlan69=10.69.1.176 |
+--------------------------------------+-----------+--------+------------+-------------+------------------------+
```

Shrinking cluster size

```
$ openstack cluster shrink my-cluster
```

Verify shrinking, as you can see it has started deleting freshly created vms. 

```
$ nova list
+--------------------------------------+-----------+---------+------------+-------------+------------------------+
| ID                                   | Name      | Status  | Task State | Power State | Networks               |
+--------------------------------------+-----------+---------+------------+-------------+------------------------+
| 39b211e8-4794-4bb6-841f-d9f83d7a60a0 | senlin-vm | DELETED | -          | NOSTATE     |                        |
| 3e365a7c-d4e4-46e1-a54d-155ead06c000 | senlin-vm | ACTIVE  | -          | Running     | net_vlan69=10.69.1.254 |
| 4a012717-d2c3-47cb-b91f-d4b2dfac7908 | senlin-vm | ACTIVE  | -          | Running     | net_vlan69=10.69.1.176 |
+--------------------------------------+-----------+---------+------------+-------------+------------------------+
```

You should be using webhooks for scaling clsuter up/down, I have covered that topic here in details [senline webhooks](https://satishdotpatel.github.io/openstack-senlin-autoscaling/)
