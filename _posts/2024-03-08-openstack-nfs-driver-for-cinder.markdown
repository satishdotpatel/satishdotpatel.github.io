---
title: "Openstack NFS Driver for Cinder"
layout: post
date: 2024-03-08
image: /assets/images/2024-03-08-openstack-nfs-driver-for-cinder/storage-nfs.png
headerImage: true
tag:
- Openstack
- Kolla-Ansible
- NFS
- Cinder
- Storage
category: blog
blog: true
author: Satish Patel
description: "Openstack NFS Driver for Cinder"

---

Cinder can use network file system (NFS) shares as a storage backend driver using an NFS driver implementation. A Cinder-volume service takes care of mounting the NFS shares and creates volumes on the shares. The NFS driver works differently than other Cinder block storage drivers. The NFS driver does not allow an instance to access a storage block device, instead files are created on an NFS share and mapped to instances, which emulates a block device.

### Prerequisite  

* Openstack working environment (In my case kolla-ansible) 
* Working NFS server (Meke sure running NFS version 4.x)

Notes: I'm running kolla-ansible (Zed). Adjust some configuration according your deployment tools. 

![<img>](/assets/images/2024-03-08-openstack-nfs-driver-for-cinder/cinder-nfs.png){: width="800" }

### Configure kolla-ansible for NFS

Configure NFS share for mountpoint. 

In /etc/kolla/config/nfs_shares

```
192.168.24.120:/volume1/pool1
```

In /etc/kolla/global.yaml 

```
enable_cinder_backend_nfs: "yes"
```

Add NFS configuration at cinder.conf (you can find stanza name "volumes-nfs")

```
[DEFAULT]
enabled_backends = volumes-ssd,volumes-nvme,volumes-nfs
default_volume_type = ssd

[volumes-ssd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_group = volumes-ssd
volume_backend_name = volumes-ssd
rbd_pool = volumes-ssd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = 5
rbd_user = cinder

[volumes-nvme]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
ssd-volumes = volumes-nvme
volume_backend_name = volumes-nvme
rbd_pool = volumes-nvme
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = 5
rbd_user = cinder

[volumes-nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
volume_backend_name = volumes-nfs
nfs_shares_config = /etc/cinder/nfs_shares
nfs_snapshot_support = True
nas_secure_file_permissions = False
nas_secure_file_operations = False
```

### Run kolla-ansible playbook to push changes

```
$ kolla-ansible -i multinode reconfigure -t cinder
```

### Create cinder volume type

```
$ openstack volume type create nfs
$ openstack volume type set ssd --property volume_backend_name=volumes-nfs
```

Verify 

```
$ openstack volume type list
+--------------------------------------+-------------+-----------+
| ID                                   | Name        | Is Public |
+--------------------------------------+-------------+-----------+
| 96692b5d-dbff-4831-a0b2-7314eae49b3e | nfs         | True      |
| 849277c1-3e0e-47ac-83c1-f851546b378b | nvme        | True      |
| f83604f6-1ad9-4f75-bdd0-87dbeecb6358 | ssd         | True      |
| 6ab1e4d6-13c2-45d6-b6cf-08bc9e64a22e | __DEFAULT__ | True      |
+--------------------------------------+-------------+-----------+
```

### Verify cinder volume service

```
openstack volume service list --service cinder-volume
+---------------+-----------------------+------+---------+-------+----------------------------+
| Binary        | Host                  | Zone | Status  | State | Updated At                 |
+---------------+-----------------------+------+---------+-------+----------------------------+
| cinder-volume | os-ctrl2@volumes-ssd  | nova | enabled | up    | 2024-03-10T04:04:03.000000 |
| cinder-volume | os-ctrl1@volumes-ssd  | nova | enabled | up    | 2024-03-10T04:04:07.000000 |
| cinder-volume | os-ctrl2@volumes-nvme | nova | enabled | up    | 2024-03-10T04:04:03.000000 |
| cinder-volume | os-ctrl1@volumes-nvme | nova | enabled | up    | 2024-03-10T04:04:00.000000 |
| cinder-volume | os-ctrl3@volumes-ssd  | nova | enabled | up    | 2024-03-10T04:03:59.000000 |
| cinder-volume | os-ctrl3@volumes-nvme | nova | enabled | up    | 2024-03-10T04:03:59.000000 |
| cinder-volume | os-ctrl3@volumes-nfs  | nova | enabled | up    | 2024-03-10T04:04:01.000000 |
| cinder-volume | os-ctrl2@volumes-nfs  | nova | enabled | up    | 2024-03-10T04:04:07.000000 |
| cinder-volume | os-ctrl1@volumes-nfs  | nova | enabled | up    | 2024-03-10T04:04:04.000000 |
+---------------+-----------------------+------+---------+-------+----------------------------+
```

### Test create & attach volume

Create 1G nfs volume using following command.

```
$ openstack volume create --size 1 --type nfs my-nfs-vol
```

Verify 

```
$ openstack volume list
+--------------------------------------+------------+-----------+------+------------------------------+
| ID                                   | Name       | Status    | Size | Attached to                  |
+--------------------------------------+------------+-----------+------+------------------------------+
| 70d9ae7c-83eb-4b95-98ee-49fad1a6a1bc | my-nfs-vol | available |    1 |                              |
+--------------------------------------+------------+-----------+------+------------------------------+
```

You can see on NFS mount it created volume-XXXXX file. Get inside cinder_volume container on one of controller nodes.

```
$ docker exec -it -u0 cinder_volume bash

(cinder-volume)[root@os-ctrl1 /]# mount | grep nfs
192.168.24.120:/volume1/pool1 on /var/lib/cinder/mnt/5639dfcb71d252f8a2e7ab65acee56dc type nfs4 (rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.24.110,local_lock=none,addr=192.168.24.120)

$ (cinder-volume)[root@os-ctrl1 /]# ls -ltr /var/lib/cinder/mnt/5639dfcb71d252f8a2e7ab65acee56dc/volume-*
-rw-rw-rw- 1 root   cinder  1073741824 Mar 10 04:10 /var/lib/cinder/mnt/5639dfcb71d252f8a2e7ab65acee56dc/volume-70d9ae7c-83eb-4b95-98ee-49fad1a6a1bc
```

Attach nfs volume to vm instance.

```
$ openstack server list
+--------------------------------------+------+--------+---------------------+--------+---------+
| ID                                   | Name | Status | Networks            | Image  | Flavor  |
+--------------------------------------+------+--------+---------------------+--------+---------+
| e7f2b821-d9a2-4531-90fb-77c61540f211 | vm1  | ACTIVE | demo-net=10.0.0.221 | cirros | m1.tiny |
+--------------------------------------+------+--------+---------------------+--------+---------+
```

Attach "my-nfs-vol" nfs volume to vm1 

```
$ openstack server add volume vm1 my-nfs-vol
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| ID                    | 70d9ae7c-83eb-4b95-98ee-49fad1a6a1bc |
| Server ID             | e7f2b821-d9a2-4531-90fb-77c61540f211 |
| Volume ID             | 70d9ae7c-83eb-4b95-98ee-49fad1a6a1bc |
| Device                | /dev/sdb                             |
| Tag                   | None                                 |
| Delete On Termination | False                                |
+-----------------------+--------------------------------------+
```

After attachment you will notice NFS share will auto mount on compute nodes and pass to running vm on that compute node. Go to compute node where vm1 is running and verify with following commands. 

```
root@os-comp5:~# docker exec -it nova_compute bash
(nova-compute)[nova@os-comp5 /]$ mount | grep nfs
192.168.24.120:/volume1/pool1 on /var/lib/nova/mnt/5639dfcb71d252f8a2e7ab65acee56dc type nfs4 (rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.24.108,local_lock=none,addr=192.168.24.120)
```

Enjoy!!! 










