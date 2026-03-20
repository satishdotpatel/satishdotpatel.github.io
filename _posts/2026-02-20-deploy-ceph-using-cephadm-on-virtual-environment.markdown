---
title: "Deploy Ceph Using Cephadm on Virtual Environment"
layout: post
date: 2026-02-20
tag:
- Ceph
- Cephadm
- Virtualization
- Storage
- Linux
category: blog
blog: true
author: Satish Patel
description: "Deploy Ceph Using Cephadm on Virtual Environment"

---

Cephadm uses SSH connections to interact with hosts and deploys services using standard container images. There are no dependencies on outside tools like ansible and salt.

The cluster can be built by downloading a binary and running a bootstrap sequence, making Cephadm one of the fastest approaches for getting a new Ceph cluster operational.

## Lab Setup

Three virtual servers with the following roles will be deployed:

- ceph1 - 10.73.0.191 (mon,mgr,osd)
- ceph2 - 10.73.0.192 (mon,mgr,osd)
- ceph3 - 10.73.0.193 (mon)

## Prerequisite

- Docker or Podman (https://docs.docker.com/engine/install/ubuntu/)
- Python3
- LVM

## Install cephadm

You can install cephadm using apt or yum command provided by your distribution or use curl to download the latest script.

```
root@ceph1:~# curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
root@ceph1:~# ls
cephadm
root@ceph1:~# chmod +x cephadm
```

### Add repo

```
root@ceph1:~# ./cephadm add-repo --release pacific
root@ceph1:~# ./cephadm install
root@ceph1:~# which cephadm
/usr/sbin/cephadm
```

### Bootstrap a new cluster

The first step in creating a new Ceph cluster involves running the cephadm bootstrap command on the Ceph cluster's first host. The bootstrap command on the initial host creates the Ceph cluster's first monitor daemon, which requires an IP address. You must pass the IP address of the Ceph cluster's first host to the bootstrap command.

Following command completion outputs the dashboard username and password.

```
root@ceph1:~# cephadm bootstrap --mon-ip 10.73.0.191 --allow-fqdn-hostname
```

The first node is now ready:

```
root@ceph1:~# docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED        STATUS        PORTS     NAMES
f948bbd65858   quay.io/ceph/ceph-grafana:8.3.5           "/bin/sh -c 'grafana…"   6 minutes ago   Up 6 minutes             ceph-...-grafana-ceph1
da9940de0261   quay.io/prometheus/alertmanager:v0.23.0   "/alertmanager -…"        6 minutes ago   Up 6 minutes             ceph-...-alertmanager-ceph1
5d64f507e598   quay.io/prometheus/prometheus:v2.33.4     "/bin/prometheus --c…"    6 minutes ago   Up 6 minutes             ceph-...-prometheus-ceph1
189ba16afeef   quay.io/prometheus/node-exporter:v1.3.1   "/bin/node_exporter …"   6 minutes ago   Up 6 minutes             ceph-...-node-exporter-ceph1
6dc14e163713   quay.io/ceph/ceph                         "/usr/bin/ceph-crash…"   6 minutes ago   Up 6 minutes             ceph-...-crash-ceph1
8f2887215bf4   quay.io/ceph/ceph                         "/usr/bin/ceph-mon -…"   6 minutes ago   Up 6 minutes             ceph-...-mon-ceph1
d555bddb6bcc   quay.io/ceph/ceph                         "/usr/bin/ceph-mgr -…"   6 minutes ago   Up 6 minutes             ceph-...-mgr-ceph1-mmsoeo
```

### Enable ceph CLI

Cephadm does not require any Ceph packages on the host. However, enabling easy access to the ceph command is recommended. The cephadm shell command spawns a container to execute ceph commands.

```
root@ceph1:~# cephadm shell -- ceph -s
Inferring fsid e0ed5b04-2d51-11ed-99fd-4124623d1806
Using recent ceph image quay.io/ceph/ceph@sha256:43f6e905f...
  cluster:
    id:     e0ed5b04-2d51-11ed-99fd-4124623d1806
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum ceph1 (age 10m)
    mgr: ceph1.mmsoeo(active, since 10m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

Install the ceph-common package, which contains all ceph commands, including ceph, rbd, mount.ceph (for mounting CephFS file systems), etc.:

```
root@ceph1:~# cephadm install ceph-common
```

Now run ceph commands natively:

```
root@ceph1:~# ceph health
HEALTH_OK
```

### Adding additional hosts to the cluster

Before adding new nodes, set the mon role to unmanaged. Otherwise, cephadm will auto-deploy monitors on ceph2. For quorum, only a single monitor is required initially:

```
root@ceph1:~# ceph orch apply mon --unmanaged
```

Adding new hosts involves two steps:

1. Install the cluster's public SSH key in the new host's root user's authorized_keys file:

```
root@ceph1:~# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
```

2. Tell Ceph that the new node is part of the cluster:

```
root@ceph1:~# ceph orch host add ceph2 10.73.0.192 --labels _admin
```

Within minutes, you will see two manager nodes in the cluster:

```
root@ceph1:~# ceph -s
  cluster:
    id:     e0ed5b04-2d51-11ed-99fd-4124623d1806
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum ceph1 (age 20m)
    mgr: ceph1.mmsoeo(active, since 20m), standbys: ceph2.ewfwsb
```

### Adding OSDs

For this lab, loop devices create multiple OSDs to simulate fake disks. (Do not use loop devices in production environments)

Create LVM disk on both ceph1 and ceph2 nodes:

```
$ fallocate -l 200G 200GB-SSD-0.img
$ fallocate -l 200G 200GB-SSD-1.img

$ losetup -fP 200GB-SSD-0.img
$ losetup -fP 200GB-SSD-1.img

$ pvcreate /dev/loop0
$ pvcreate /dev/loop1

$ vgcreate ceph-ssd-vg /dev/loop0 /dev/loop1

$ lvcreate --size 199G --name ceph-ssd-lv-0 ceph-ssd-vg
$ lvcreate --size 199G --name ceph-ssd-lv-1 ceph-ssd-vg
```

Two 199GB LVM disks per node enables adding four OSDs total to ceph:

```
root@ceph1:~# ceph orch daemon add osd ceph1:/dev/ceph-ssd-vg/ceph-ssd-lv-0
root@ceph1:~# ceph orch daemon add osd ceph1:/dev/ceph-ssd-vg/ceph-ssd-lv-1

root@ceph1:~# ceph orch daemon add osd ceph2:/dev/ceph-ssd-vg/ceph-ssd-lv-0
root@ceph1:~# ceph orch daemon add osd ceph2:/dev/ceph-ssd-vg/ceph-ssd-lv-1
```

Check status after waiting:

```
root@ceph1:~# ceph osd stat
4 osds: 4 up (since 2h), 4 in (since 5h); epoch: e103

root@ceph1:~# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.77716  root default
-3         0.38858      host ceph1
 0    ssd  0.19429          osd.0       up   1.00000  1.00000
 1    ssd  0.19429          osd.1       up   1.00000  1.00000
-5         0.38858      host ceph2
 2    ssd  0.19429          osd.2       up   1.00000  1.00000
 3    ssd  0.19429          osd.3       up   1.00000  1.00000
```

## Scale mon daemons

Currently with two nodes, the cluster has a single monitor. Adding a third node and monitors on all three nodes improves redundancy.

### Add new ceph3 node in cluster

Prerequisites include docker-ce installation on the new host (ceph3). After that, copy the ceph key to the new host:

```
root@ceph1:~# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
```

Tell Ceph that the new node is part of the cluster:

```
root@ceph1:~# ceph orch host add ceph3 10.73.0.193 --labels _admin
```

Few minutes later:

```
root@ceph1:~# ceph orch host ls
HOST   ADDR         LABELS  STATUS
ceph1  10.73.0.191  _admin
ceph2  10.73.0.192  _admin
ceph3  10.73.0.193  _admin
3 hosts in cluster
```

Now direct cephadm to add additional monitor daemons on ceph2 and ceph3:

```
root@ceph1:~# ceph orch daemon add mon ceph2:10.73.0.192
root@ceph1:~# ceph orch daemon add mon ceph3:10.73.0.193
```

Enable automatic daemon placement:

```
root@ceph1:~# ceph orch apply mon --placement="ceph1,ceph2,ceph3" --dry-run
root@ceph1:~# ceph orch apply mon --placement="ceph1,ceph2,ceph3"
```

After several minutes, three monitor daemons will be present:

```
root@ceph1:~# ceph mon stat
e12: 3 mons at {ceph1=[v2:10.73.0.191:3300/0,v1:10.73.0.191:6789/0],ceph2=[v2:10.73.0.192:3300/0,v1:10.73.0.192:6789/0],ceph3=[v2:10.73.0.193:3300/0,v1:10.73.0.193:6789/0]}, election epoch 48, leader 0 ceph1, quorum 0,1,2 ceph1,ceph3,ceph2
```

## Ceph Maintenance Options

For performing maintenance on OSD nodes, use these flags:

```
ceph osd set noout
ceph osd set norebalance
ceph osd set norecover
```

To remove maintenance mode:

```
ceph osd unset noout
ceph osd unset norebalance
ceph osd unset norecover
```
