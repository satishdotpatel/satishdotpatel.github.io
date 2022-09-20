---
title: "Deploy Ceph Using Cephadm"
layout: post
date: 2022-09-06
image: /assets/images/2022-09-06-deploy-ceph-storage-using-cephadm/ceph-logo.png
headerImage: true
tag:
- Openstack
- Ceph
- Cephadm
- Storage
category: blog
blog: true
author: Satish Patel
description: "Deploy Ceph Using Cephadm"

---

### What is cephadm?

Cephadm uses SSH connections to interact with hosts and deploys services using standard container images. There are no dependencies on outside tools like ansible and salt. Cluster is built simply by downloading a binary and running a bootstrap sequence. In short, Cephadm is the easiest way yet to get a new Ceph cluster up and running quickly. 

#### Lab 

Two servers running Ubuntu 20.04, following roles we are going to run on them. Later i will add 3rd node during that time i will spin up 3 mon daemon. 

* ceph1 - 10.73.0.191 (mon,mgr,osd)
* ceph2 - 10.73.0.192 (mon,mgr,osd)
* ceph3 - 10.73.0.193 (mon)

#### Prerequisite 

* Docker or Podman ( In my case its docker https://docs.docker.com/engine/install/ubuntu/)
* Python3
* LVM


### Install cephadm

You can install cephadm using apt or yum command provided by disto or use curl to download latest script.

```
root@ceph1:~# curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
root@ceph1:~# ls
cephadm
root@ceph1:~# chmod +x cephadm
```

#### Add repo 

```
root@ceph1:~# ./cephadm add-repo --release pacific
root@ceph1:~# ./cephadm install
root@ceph1:~# which cephadm
/usr/sbin/cephadm
```

#### Bootstrap a new cluster

The first step in creating a new Ceph cluster is running the cephadm bootstrap command on the Ceph cluster’s first host. The act of running the cephadm bootstrap command on the Ceph cluster’s first host creates the Ceph cluster’s first “monitor daemon”, and that monitor daemon needs an IP address. You must pass the IP address of the Ceph cluster’s first host to the ceph bootstrap command, so you’ll need to know the IP address of that host.

End of following command it will spit out username/password of dashborad. 

```
root@ceph1:~# cephadm bootstrap --mon-ip 10.73.0.191 --allow-fqdn-hostname
```

You can see, your first node is ready.

```
root@ceph1:~# docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED        STATUS        PORTS     NAMES
f948bbd65858   quay.io/ceph/ceph-grafana:8.3.5           "/bin/sh -c 'grafana…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-grafana-ceph1
da9940de0261   quay.io/prometheus/alertmanager:v0.23.0   "/bin/alertmanager -…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-alertmanager-ceph1
5d64f507e598   quay.io/prometheus/prometheus:v2.33.4     "/bin/prometheus --c…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-prometheus-ceph1
189ba16afeef   quay.io/prometheus/node-exporter:v1.3.1   "/bin/node_exporter …"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-node-exporter-ceph1
6dc14e163713   quay.io/ceph/ceph                         "/usr/bin/ceph-crash…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-crash-ceph1
8f2887215bf4   quay.io/ceph/ceph                         "/usr/bin/ceph-mon -…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-mon-ceph1
d555bddb6bcc   quay.io/ceph/ceph                         "/usr/bin/ceph-mgr -…"   6 minutes ago   Up 6 minutes             ceph-e0ed5b04-2d51-11ed-99fd-4124623d1806-mgr-ceph1-mmsoeo
```

#### Enable ceph CLI

Cephadm does not require any Ceph packages to be installed on the host. However, it recommends enabling easy access to the ceph command. cephadm shell spawn container to execute ceph commands.

```
root@ceph1:~# cephadm shell -- ceph -s
Inferring fsid e0ed5b04-2d51-11ed-99fd-4124623d1806
Using recent ceph image quay.io/ceph/ceph@sha256:43f6e905f3e34abe4adbc9042b9d6f6b625dee8fa8d93c2bae53fa9b61c3df1a
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

You can install the ceph-common package, which contains all of the ceph commands, including ceph, rbd, mount.ceph (for mounting CephFS file systems), etc.:


```
root@ceph1:~# cephadm install ceph-common
```

Now you can run ceph command native.

```
root@ceph1:~# ceph health
HEALTH_OK
```

#### Adding additional hosts to the cluster

NOTES: Before adding new node to cluster i would like to set unmanage on mon role. Otherwise cephadm will auto deploy mon on ceph2. (For quorom we just need single mon)

```
root@ceph1:~# ceph orch apply mon --unmanaged
```

To add each new host to the cluster, perform two steps:

1. Install the cluster’s public SSH key in the new host’s root user’s authorized_keys file:

```
root@ceph1:~# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
```

2. Tell Ceph that the new node is part of the cluster:

```
root@ceph1:~# ceph orch host add ceph2 10.73.0.192 --labels _admin
```

Wait for sometime and then you will see two mgr node in cluster. 

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

#### Adding OSDs

This is lab so i am using loop device to create multiple OSDs to mimic fake disks. (Don't use loop device in production) 

Create LVM disk on both ceph1 and ceph2 nodes.

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

We have two 199GB LVM disk per nodes so total 4 OSDs we can add in ceph. 

```
root@ceph1:~# ceph orch daemon add osd ceph1:/dev/ceph-ssd-vg/ceph-ssd-lv-0
root@ceph1:~# ceph orch daemon add osd ceph1:/dev/ceph-ssd-vg/ceph-ssd-lv-0

root@ceph1:~# ceph orch daemon add osd ceph2:/dev/ceph-ssd-vg/ceph-ssd-lv-0
root@ceph1:~# ceph orch daemon add osd ceph2:/dev/ceph-ssd-vg/ceph-ssd-lv-0
```

Wait for sometime and then you can check status

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

### Scale mon daemons 

Currently we have only 2 node in ceph cluster that is why we have single mon node in cluster. I am going to add 3rd node in ceph cluster and add mon service on all 3 node for better redendency. 

Current status of cluster

```
root@ceph1:~# ceph orch host ls
HOST   ADDR         LABELS  STATUS
ceph1  10.73.0.191  _admin
ceph2  10.73.0.192  _admin
2 hosts in cluster
```

#### Add new ceph3 node in cluster 

Pre-requisite to install docker-ce on newhost (ceph3). After that copy ceph key to newhost.

```
root@ceph1:~# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
```

Tell Ceph that the new node is part of the cluster:

```
root@ceph1:~# ceph orch host add ceph3 10.73.0.193 --labels _admin
```

Few minutes later

```
root@ceph1:~# ceph orch host ls
HOST   ADDR         LABELS  STATUS
ceph1  10.73.0.191  _admin
ceph2  10.73.0.192  _admin
ceph3  10.73.0.193  _admin
3 hosts in cluster
```

Now lets tell cephadm to add 2 more mon daemon on ceph2/ceph3

```
root@ceph1:~# ceph orch daemon add mon ceph2:10.73.0.192
root@ceph1:~# ceph orch daemon add mon ceph3:10.73.0.193
```

Now, enable automatic placement of Daemons

```
root@ceph1:~# ceph orch apply mon --placement="ceph1,ceph2,ceph3" --dry-run
root@ceph1:~# ceph orch apply mon --placement="ceph1,ceph2,ceph3"
```

After few minutes you will see we have 3 mon daemon 

```
root@ceph1:~# ceph mon stat
e12: 3 mons at {ceph1=[v2:10.73.0.191:3300/0,v1:10.73.0.191:6789/0],ceph2=[v2:10.73.0.192:3300/0,v1:10.73.0.192:6789/0],ceph3=[v2:10.73.0.193:3300/0,v1:10.73.0.193:6789/0]}, election epoch 48, leader 0 ceph1, quorum 0,1,2 ceph1,ceph3,ceph2
```

In more details

```
root@ceph1:~# ceph orch ps
NAME                 HOST   PORTS        STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
alertmanager.ceph1   ceph1  *:9093,9094  running (13d)     5m ago   2w    50.9M        -           ba2b418f427c  da9940de0261
crash.ceph1          ceph1               running (13d)     5m ago   2w    8584k        -  17.2.3   0912465dcea5  6dc14e163713
crash.ceph2          ceph2               running (13d)   110s ago   2w    7568k        -  17.2.3   0912465dcea5  046663e7d825
crash.ceph3          ceph3               running (28m)    44s ago  28m    9092k        -  17.2.3   0912465dcea5  068bcd9a1b0c
grafana.ceph1        ceph1  *:3000       running (13d)     5m ago   2w     130M        -  8.3.5    dad864ee21e9  f948bbd65858
mgr.ceph1.mmsoeo     ceph1  *:8443,9283  running (13d)     5m ago   2w     688M        -  17.2.3   0912465dcea5  d555bddb6bcc
mgr.ceph2.ewfwsb     ceph2  *:8443,9283  running (13d)   110s ago   2w     436M        -  17.2.3   0912465dcea5  17c25c7ac9d7
mon.ceph1            ceph1               running (13d)     5m ago   2w     464M    2048M  17.2.3   0912465dcea5  8f2887215bf4
mon.ceph2            ceph2               running (12m)   110s ago  12m    55.1M    2048M  17.2.3   0912465dcea5  f93695536d9e
mon.ceph3            ceph3               running (21m)    44s ago  21m    84.8M    2048M  17.2.3   0912465dcea5  2532ddaed999
node-exporter.ceph1  ceph1  *:9100       running (13d)     5m ago   2w    67.7M        -           1dbe0e931976  189ba16afeef
node-exporter.ceph2  ceph2  *:9100       running (13d)   110s ago   2w    67.5M        -           1dbe0e931976  f87b1ec6f349
node-exporter.ceph3  ceph3  *:9100       running (28m)    44s ago  28m    46.3M        -           1dbe0e931976  283c6d21ea9c
osd.0                ceph1               running (13d)     5m ago   2w    1390M    17.2G  17.2.3   0912465dcea5  3456c126e322
osd.1                ceph1               running (13d)     5m ago   2w    1373M    17.2G  17.2.3   0912465dcea5  7c1fa2662443
osd.2                ceph2               running (13d)   110s ago   2w    1534M    18.7G  17.2.3   0912465dcea5  336e424bbbb2
osd.3                ceph2               running (13d)   110s ago   2w    1506M    18.7G  17.2.3   0912465dcea5  874a811e1f3b
prometheus.ceph1     ceph1  *:9095       running (28m)     5m ago   2w     122M        -           514e6a882f6e  93972e9bdfa9
```

In next post I will cover how to integrate ceph with kolla-ansible deployment. Enjoy! 

