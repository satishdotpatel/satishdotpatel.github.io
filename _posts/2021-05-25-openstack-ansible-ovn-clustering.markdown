---
title: "Openstack Ansible OVN Clustering - Part-3"
layout: post
date: 2021-04-08
image: /assets/images/2021-03-29-openstack-ansible-ovn-deployment/openstack-sdn.png
headerImage: true
tag:
- openstack
- openstack-ansible
- ovn
- clustering
- SDN
- openvswitch
category: blog
blog: true
author: Satish Patel
description: "Openstack Ansible OVN Clustering - Part-3"

---

In a previous post I have explained how to deploy OVN using openstack-ansible in multi-node deployment. In this post we will talk about how to create high-availability clustering using OVN.  

### Prerequisite

* 3 Infra Nodes 
* 1 Compute Node

### What is Raft

OVN north/south bound use ovsdb-server for a database to cluster ovsdb-server it use Raft Consensus Algorithm to create 3 node or 5 node cluster. more information: https://raft.github.io/. In Raft cluster nodes has 3 following states.

* Follower   - This is first state when node start, start timer 
* Candidate  - After timeout it declare itself candidate for votes
* Leader     - If get enough vote then it become leader and all other nodes followers

### Create cluster

Notes: At present openstack-ansible doesn't offer working playbook to build ovn cluster. I'm building this cluster manually but soon we will have working playbook to support automatic deployment of ovn cluster.

I have 3 neutron_ovn_northd containers in 3 infra nodes.

```
root@os-osa:/opt/openstack-ansible/playbooks# /usr/bin/python3 /usr/local/bin/inventory-manage.py -l | grep ovn
| os-infra-1_neutron_ovn_northd_container-24eea9c2 | None     | neutron_ovn_northd | os-infra-1    | None           | 172.30.40.93  | None                         |
| os-infra-2_neutron_ovn_northd_container-bdb3912a | None     | neutron_ovn_northd | os-infra-2    | None           | 172.30.40.25  | None                         |
| os-infra-3_neutron_ovn_northd_container-24e31668 | None     | neutron_ovn_northd | os-infra-3    | None           | 172.30.40.177 | None                         |
```

Currently all 3 nodes not in a cluster and acting up like a standalone nodes. First we will pick os-infra-1 to start cluster and then other nodes to connect to os-infra-1

#### os-infra-1

Go to os-infra-1 and make sure ovn-northd service is stopped. 

```
root@os-infra-1:~# lxc-attach -n os-infra-1_neutron_ovn_northd_container-24eea9c2
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~#
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# systemctl stop ovn-northd
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ps aux | grep ovn-northd
root        2215  0.0  0.0   6432   672 ?        S+   18:21   0:00 grep --color=auto ovn-northd
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~#
```

Remove all the files in /var/lib/ovn also hiden lock files otherwise it won't let you create cluster database

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# rm -rf /var/lib/ovn/*
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# rm -rf /var/lib/ovn/.ovn*
```

Start northd with following command. 

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# /usr/share/ovn/scripts/ovn-ctl --db-nb-addr=172.30.40.93 \
> --db-nb-create-insecure-remote=yes \
> --db-sb-addr=172.30.40.93 \
> --db-sb-create-insecure-remote=yes \
> --db-nb-cluster-local-addr=172.30.40.93 \
> --db-sb-cluster-local-addr=172.30.40.93 \
> --ovn-northd-nb-db=tcp:172.30.40.93:6641,tcp:172.30.40.25:6641,tcp:172.30.40.177:6641 \
> --ovn-northd-sb-db=tcp:172.30.40.93:6642,tcp:172.30.40.25:6642,tcp:172.30.40.177:6642 \
> start_northd

 * Backing up database to /var/lib/ovn/ovnnb_db.db.backup5.20.0-987891875
 * Creating cluster database /var/lib/ovn/ovnnb_db.db from existing one
2021-05-25T18:23:39Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connecting...
2021-05-25T18:23:39Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connected
 * Waiting for OVN_Northbound to come up
 * Backing up database to /var/lib/ovn/ovnsb_db.db.backup2.7.0-4286723485
 * Creating cluster database /var/lib/ovn/ovnsb_db.db from existing one
2021-05-25T18:23:39Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connecting...
2021-05-25T18:23:39Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connected
 * Waiting for OVN_Southbound to come up
 * Starting ovn-northd
```

If all good then you will see following result when you run cluster/status command. As you can see this is a first node and vote itself to make itself leader, now when we bring up other nodes they will be followers and start syncing database with leader.

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
3b93
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: 3b93 (3b932313-41dd-4e8a-9097-1b0f54ae983e)
Address: tcp:172.30.40.93:6643
Status: cluster member
Role: leader
Term: 1
Leader: self
Vote: self

Election timer: 1000
Log: [2, 3]
Entries not yet committed: 0
Entries not yet applied: 0
Connections:
Servers:
    3b93 (3b93 at tcp:172.30.40.93:6643) (self) next_index=2 match_index=2
```

#### os-infra-2

Lets bring up second node and join it to os-infra-1

clean up db directoy first

```
root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# rm -rf /var/lib/ovn/*
root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# rm -rf /var/lib/ovn/.ovn*
```

Join cluster using following command remote-addr will be your os-infra-1 node

```
root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# /usr/share/ovn/scripts/ovn-ctl --db-nb-addr=172.30.40.25 \
> --db-nb-create-insecure-remote=yes \
> --db-sb-addr=172.30.40.25 \
> --db-sb-create-insecure-remote=yes \
> --db-nb-cluster-local-addr=172.30.40.25 \
> --db-sb-cluster-local-addr=172.30.40.25 \
> --db-nb-cluster-remote-addr=172.30.40.93 \
> --db-sb-cluster-remote-addr=172.30.40.93 \
> start_northd

2021-05-25T18:56:19Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connecting...
2021-05-25T18:56:19Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connected
 * Waiting for OVN_Northbound to come up
2021-05-25T18:56:20Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connecting...
2021-05-25T18:56:20Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connected
 * Waiting for OVN_Southbound to come up
 * Starting ovn-northd
 ```

 Verify cluster status, you can see os-infra-2 is follower because leader was already presented and vote is unknow because it didn't get chance to vote. 

 ```
 root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
d68c
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: d68c (d68c3219-43c4-454a-8bdd-d28f4be27a2a)
Address: tcp:172.30.40.25:6643
Status: cluster member
Role: follower
Term: 1
Leader: 3b93
Vote: unknown

Election timer: 1000
Log: [2, 4]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: ->0000 <-3b93
Servers:
    d68c (d68c at tcp:172.30.40.25:6643) (self)
    3b93 (3b93 at tcp:172.30.40.93:6643)
```

Lets check status on os-infra-1, as you can see in Servers: section we have two nodes and Connections: showing it can talk both direction in and out <-d68c ->d68c

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
3b93
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: 3b93 (3b932313-41dd-4e8a-9097-1b0f54ae983e)
Address: tcp:172.30.40.93:6643
Status: cluster member
Role: leader
Term: 1
Leader: self
Vote: self

Election timer: 1000
Log: [2, 4]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: <-d68c ->d68c
Servers:
    d68c (d68c at tcp:172.30.40.25:6643) next_index=4 match_index=3
    3b93 (3b93 at tcp:172.30.40.93:6643) (self) next_index=2 match_index=3
```

#### os-infra-3

Clean up db files

```
root@os-infra-3-neutron-ovn-northd-container-24e31668:~# rm -rf /var/lib/ovn/*
root@os-infra-3-neutron-ovn-northd-container-24e31668:~# rm -rf /var/lib/ovn/.ovn*
```

Join cluster using following command remote-addr will be your os-infra-1 node

```
root@os-infra-3-neutron-ovn-northd-container-24e31668:~# /usr/share/ovn/scripts/ovn-ctl  --db-nb-addr=172.30.40.177 \
> --db-nb-create-insecure-remote=yes \
> --db-nb-cluster-local-addr=172.30.40.177 \
> --db-sb-addr=172.30.40.177 \
> --db-sb-create-insecure-remote=yes \
> --db-sb-cluster-local-addr=172.30.40.177 \
> --db-nb-cluster-remote-addr=172.30.40.93 \
> --db-sb-cluster-remote-addr=172.30.40.93 \
> start_northd

2021-05-25T19:16:50Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connecting...
2021-05-25T19:16:50Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnnb_db.sock: connected
 * Waiting for OVN_Northbound to come up
2021-05-25T19:16:50Z|00001|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connecting...
2021-05-25T19:16:50Z|00002|reconnect|INFO|unix:/var/run/ovn/ovnsb_db.sock: connected
 * Waiting for OVN_Southbound to come up
 * Starting ovn-northd
```

Verify cluster status, as you can see its follower and vote: is unknown because it didn't participate in voting process. 

```
root@os-infra-3-neutron-ovn-northd-container-24e31668:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
7b0e
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: 7b0e (7b0e82a3-7127-4eae-8851-5bd60fc05937)
Address: tcp:172.30.40.177:6643
Status: cluster member
Role: follower
Term: 1
Leader: 3b93
Vote: unknown

Election timer: 1000
Log: [2, 5]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: ->0000 ->d68c <-3b93 <-d68c
Servers:
    d68c (d68c at tcp:172.30.40.25:6643)
    7b0e (7b0e at tcp:172.30.40.177:6643) (self)
    3b93 (3b93 at tcp:172.30.40.93:6643)
```

Lets verify on os-infra-1, Check Servers: and Connections: section and you will see we have 3 nodes.

```
root@os-infra-1-neutron-ovn-northd-container-24eea9c2:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
3b93
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: 3b93 (3b932313-41dd-4e8a-9097-1b0f54ae983e)
Address: tcp:172.30.40.93:6643
Status: cluster member
Role: leader
Term: 1
Leader: self
Vote: self

Election timer: 1000
Log: [2, 5]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: <-d68c ->d68c <-7b0e ->7b0e
Servers:
    d68c (d68c at tcp:172.30.40.25:6643) next_index=5 match_index=4
    7b0e (7b0e at tcp:172.30.40.177:6643) next_index=5 match_index=4
    3b93 (3b93 at tcp:172.30.40.93:6643) (self) next_index=2 match_index=4
```

#### Cluster failover testing 


Lets shutdown os-infra-1 node which is leader and see what happen. 

```
root@os-infra-1:~# lxc-stop -n os-infra-1_neutron_ovn_northd_container-24eea9c2
```

Voila!  As you can see os-infra-2 is new leader now, Pay attention to Term: section whenever node participate in election process Term: get increment by 1

```
root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound
d68c
Name: OVN_Northbound
Cluster ID: eb3b (eb3b01bb-fab5-42b4-8095-2e479ad9cabc)
Server ID: d68c (d68c3219-43c4-454a-8bdd-d28f4be27a2a)
Address: tcp:172.30.40.25:6643
Status: cluster member
Role: leader
Term: 2
Leader: self
Vote: self

Election timer: 1000
Log: [2, 6]
Entries not yet committed: 0
Entries not yet applied: 0
Connections: (->0000) <-7b0e ->7b0e
Servers:
    d68c (d68c at tcp:172.30.40.25:6643) (self) next_index=5 match_index=5
    7b0e (7b0e at tcp:172.30.40.177:6643) next_index=6 match_index=5
    3b93 (3b93 at tcp:172.30.40.93:6643) next_index=6 match_index=0
```

#### Recovery from complete database failure 

Lets say something happened and you lost whole cluster and backup also in that case use following tool to rebuild your northd database


Go to neutron-server container and run neutron-ovn-db-sync-util command to rebuild db. 

```
root@os-infra-1-neutron-server-container-49dba0f4:~# /openstack/venvs/neutron-22.0.0.0rc2.dev111/bin/neutron-ovn-db-sync-util --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --ovn-neutron_sync_mode repair

```

To verify north database following command from leader node (on non-leader node use this this command "ovn-nbctl --no-leader-only show")

```
root@os-infra-2-neutron-ovn-northd-container-bdb3912a:~# ovn-nbctl show
switch 1753e314-7f28-4662-a0fa-76831341cb34 (neutron-b96364e8-340b-4024-97ff-2796be2acab6) (aka private)
    port 6bfc7d1c-bedc-41d8-b55e-aa40d1544abb
        type: localport
        addresses: ["fa:16:3e:dc:fa:0e 172.168.0.2"]
    port e8d9d7bf-a434-4b96-8a6c-91938f28953d
        type: router
        router-port: lrp-e8d9d7bf-a434-4b96-8a6c-91938f28953d
    port 049b06a8-934d-4785-9c03-751fe3016b0e
        addresses: ["fa:16:3e:84:c3:72 172.168.0.121"]
switch 12bfed9d-d952-471f-804c-d3459af8f5ee (neutron-8e9db0f7-576c-47c4-96a3-a4c37fb9385c) (aka public)
    port 9d32dec2-cae7-4544-bbae-c17840deaea5
        type: router
        router-port: lrp-9d32dec2-cae7-4544-bbae-c17840deaea5
    port provnet-aca91150-412d-4cf0-b76e-3f983907c6a6
        type: localnet
        tag: 40
        addresses: ["unknown"]
    port d0afa78c-d50d-43a9-b518-311c8dd236aa
        type: localport
        addresses: ["fa:16:3e:5e:4f:a8 216.163.208.11"]
router c0722814-0472-4cbc-8efb-de2b454896c6 (neutron-f2db247f-1bee-478c-a5aa-aa090d74dbc7) (aka router1)
    port lrp-e8d9d7bf-a434-4b96-8a6c-91938f28953d
        mac: "fa:16:3e:cb:50:de"
        networks: ["172.168.0.1/24"]
    port lrp-9d32dec2-cae7-4544-bbae-c17840deaea5
        mac: "fa:16:3e:92:66:48"
        networks: ["216.163.208.2/24"]
        gateway chassis: [57c973ee-464a-40df-8436-1d046a69a671]
    nat 9462251f-0610-46fe-be8c-b200be8891f7
        external ip: "216.163.208.2"
        logical ip: "172.168.0.0/24"
        type: "snat"
```
