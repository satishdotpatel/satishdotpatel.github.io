---
title: "Openstack-Ansible Glance With Glusterfs"
layout: post
date: 2022-03-15
image: /assets/images/2022-03-16-openstack-ansible-glance-with-glusterfs/data-replication.png
headerImage: true
tag:
- Glance
- GlusterFS
- Openstack-Ansible
category: blog
blog: true
author: Satish Patel
description: "Openstack-Ansible Glance With Glusterfs"

---

In some small deployment of openstack when you don't have ceph storage in that case you need a some kind of shared storage for your glance to sync images across all infra nodes. You have couple of choice like use some kind of centralized NFS or deploye swift s3 style object storage. In my case i am using glusterfs which i am going to show you how i did.

### Prerequisit 

* 3 x Infra nodes (openstack-ansible based deployment)

This is how it will looks from 3000ft. 

![<img>](/assets/images/2022-03-16-openstack-ansible-glance-with-glusterfs/osa-glance-glusterfs.png){: width="400" }

#### Install/Setup GlusterFS server

Install glusterfs on all 3 infra nodes. 

```
$ apt-get update
$ apt-get install glusterfs
$ apt -y install glusterfs-server
$ systemctl enable --now glusterd
$ service glusterd start
```

Create directory for gluster replication volume on 3 nodes

```
## os-infra-1
$ mkdir -p /gluster/bricks/1/brick

## os-infra-2
$ mkdir -p /gluster/bricks/2/brick

## os-infra-3
$ mkdir -p /gluster/bricks/3/brick

```

In order to add the nodes to the trusted storage pool, we will have to add them by using gluster peer probe. Make sure that you can resolve the hostnames to the designated IP Addresses, and that traffic is allowed. (You can perform this command to any one node)

```
$ gluster peer probe os-infra-1
$ gluster peer probe os-infra-2
$ gluster peer probe os-infra-3
```

Now that we have added our nodes to our trusted storage pool, lets verify that by listing our pool:

```
$ gluster pool list
UUID					Hostname  	State
6f7e9bb9-4a0a-4b27-b538-4040b0208400	os-infra-2	Connected
eb0f49f7-99d5-414b-8e48-36cd3a54cde2	os-infra-3	Connected
5da0c1cb-c90c-404e-9918-ca1e1534b013	localhost 	Connected
```

Great! All looks good.

#### Create the Replicated GlusterFS Volume:

Let’s create our Replicated GlusterFS Volume, named gfs: 

```
$ gluster volume create gfs \
  replica 3 \
  gfs01:/gluster/bricks/1/brick \
  gfs02:/gluster/bricks/2/brick \
  gfs03:/gluster/bricks/2/brick \
  force

volume create: gfs: success: please start the volume to access data
```
NOTES: I have used `force` command because i don't have dedicated volume for gluster and i am creating it on root filesystem which is not a best idea because possible you may fill the disk so always better to keep gluster volume on non-root or dedicated filesystem/disk. 

Now that our volume is created, lets list it to verify that it is created:

```
$ gluster volume list
gfs
```

Now, start the volume:

```
$ gluster volume start gfs
volume start: gfs: success
```

View the status of our volume:

```
$ gluster volume status gfs
Status of volume: gfs
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick os-infra-1:/gluster/bricks/1/brick    49152     0          Y       3768304
Brick os-infra-2:/gluster/bricks/2/brick    49152     0          Y       3645053
Brick os-infra-3:/gluster/bricks/3/brick    49152     0          Y       2655608
Self-heal Daemon on localhost               N/A       N/A        Y       3768325
Self-heal Daemon on os-infra-2              N/A       N/A        Y       3645074
Self-heal Daemon on os-infra-3              N/A       N/A        Y       2655629

Task Status of Volume gfs
------------------------------------------------------------------------------
There are no active volume tasks
```

Next, view the volume inforation:

```
$ gluster volume info gfs

Volume Name: gfs
Type: Replicate
Volume ID: 382eb7bc-e61d-4149-b800-67d8c9a15c28
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: os-infra-1:/gluster/bricks/1/brick
Brick2: os-infra-2:/gluster/bricks/2/brick
Brick3: os-infra-3:/gluster/bricks/3/brick
Options Reconfigured:
auth.allow: 10.0.0.0/8
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
performance.client-io-threads: off
```

#### Security:

From a GlusterFS level, it will allow clients to connect by default. To authorize these 3 nodes to connect to the GlusterFS Volume:

```
gluster volume set gfs auth.allow 10.30.40.0/24
```

Then if you would like to remove this rule:

```
gluster volume set gfs auth.allow *
```

#### Mount the GlusterFS Volume to the Infra nodes:

Let's assuming you already have images in old deployment then you can simply rsync them to gluster volume and chown permission of glance user and group. If its new installation then you don't need to rsync old data.  

First mount gfs volume on /mnt to sync old images. Gluster will replicate your data to all 3 nodes automatically.

```
$ mount.glusterfs localhost:/gfs /mnt
$ rsync -avrp /openstack/os-infra-1_glance_container-0a4a3789/.* /mnt/.
```

Now, Just mount your filesystem to glance folder on all infra nodes.

```
## os-infra-1
$ mount.glusterfs localhost:/gfs /openstack/$(lxc-ls -f | grep glance | awk '{print $1}')

## os-infra-2
$ mount.glusterfs localhost:/gfs /openstack/$(lxc-ls -f | grep glance | awk '{print $1}')

## os-infra-3
$ mount.glusterfs localhost:/gfs /openstack/$(lxc-ls -f | grep glance | awk '{print $1}')
```

Final step go to any one of node glance lxc container and fix permission.

```
$ lxc-attach -n os-infra-1_glance_container-0a4a3789
root@os-infra-1-glance-container-0a4a3789:~# chown glance:glance -R /var/lib/glance
```

This is how it looks on infra nodes.

```
root@os-infra-1:~# df -h
Filesystem            Size  Used Avail Use% Mounted on
udev                  7.8G     0  7.8G   0% /dev
tmpfs                 1.6G  1.7M  1.6G   1% /run
/dev/mapper/vg0-root   95G   51G   40G  57% /
tmpfs                 7.9G     0  7.9G   0% /dev/shm
tmpfs                 5.0M     0  5.0M   0% /run/lock
tmpfs                 7.9G     0  7.9G   0% /sys/fs/cgroup
/dev/sda2             976M  302M  608M  34% /boot
tmpfs                 1.6G     0  1.6G   0% /run/user/0
localhost:/gfs         95G   52G   40G  57% /openstack/os-infra-1_glance_container-0a4a3789
```

Restart all 3 glance containers. Now you should be able to upload images and verify replication to all 3 nodes.  

