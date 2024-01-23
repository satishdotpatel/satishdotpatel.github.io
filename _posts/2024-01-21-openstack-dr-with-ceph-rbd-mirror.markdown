---
title: "Openstack Disaster Recovery with Ceph RBD Mirror"
layout: post
date: 2024-01-21
image: /assets/images/2024-01-21-openstack-dr-with-ceph-rbd-mirror/disaster-recovery.png
headerImage: true
tag:
- Ceph
- RBD
- Storage
- Openstack
- Disater-Recovery
category: blog
blog: true
author: Satish Patel
description: "Openstack Disaster Recovery with Ceph RBD Mirror"

---

In a previous blog, I have explained how to set up [Ceph RDB Mirror](https://satishdotpatel.github.io/ceph-rbd-mirroring/) for Storage on-way replication. In this blog I will set up an openstack cluster and do a real scenario of failover from primary ceph storage to the secondary ceph storage. Just mimic disaster recovery where the primary ceph cluster goes down and how to point your all opentack vms to the secondary cluster and bring them up.

### Scope 

* Working Openstack environment (In my case kolla-ansible)
* Two working ceph1 and ceph2 cluster (with RBD mirror setup)
* For normal failover from ceph1 to ceph2 storage
  * Shutdown all the VMs 
  * Point openstack to the secondary ceph
  * Demote Primary ceph Pools/Images
  * Promote Seconday ceph Pools/Images
  * Start all the VMs

### Setup Ceph Auth on Seconday Cluster

This is personal choice but in my case to reduce typing I've export/import auth creds from primary cluster to Secondary cluster. Both clusters have similar auth/keys so I don't need to maintain keyring files on openstack. Just need to switch ceph.conf file during failover. 

On Primary ceph1 

```
$ ceph auth export client.nova -f /root/ceph-auth-nova.txt
$ ceph auth export client.cinder -f /root/ceph-auth-cinder.txt
```

Copy both files to Secondary ceph2 cluster and import them. 

```
$ ceph auth import -i /root/ceph-auth-nova.txt
$ ceph auth import -i /root/ceph-auth-cinder.txt
```

NOTES: I am using pool name "vms" to store nova instances. Just make sure you have that pool in secondary cluster. 

### Setup kolla-ansible to use ceph storage

In global.yml file 

```
nova_backend_ceph: "yes"
cinder_backend_ceph: "yes"
```

Copy ceph.conf (from primary ceph1 cluster) and keyring files in nova and cinder directory sturcture of kolla-ansible

```
$ tree /etc/kolla/config/nova
/etc/kolla/config/nova
├── ceph.client.cinder.keyring
├── ceph.client.nova.keyring
└── ceph.conf

$ tree /etc/kolla/config/cinder/
/etc/kolla/config/cinder/
├── ceph.conf
└── cinder-volume
    ├── ceph.client.cinder.keyring
```

NOTES: Make sure ceph.conf coming from Active or Primary cluster because it will point your openstack to ceph cluster. 

Run kolla-ansible to push changes.

```
$ kolla-ansible -i <inventory_file> reconfigure -t nova
```

Create few VMs for POC testing. 

```
$ openstack server create --image ubuntu --flavor m1.small --network net-vlan40 --key kolla-key vm01
$ openstack server create --image ubuntu --flavor m1.small --network net-vlan40 --key kolla-key vm02
$ openstack server create --image ubuntu --flavor m1.small --network net-vlan40 --key kolla-key vm03
```

Check status of VMs. 

```
$ openstack server list
+--------------------------------------+-------+--------+------------------------+--------+----------+
| ID                                   | Name  | Status | Networks               | Image  | Flavor   |
+--------------------------------------+-------+--------+------------------------+--------+----------+
| d4743fb5-c314-46ee-8f84-323896936d77 | vm01  | ACTIVE | net-vlan40=10.40.40.28 | ubuntu | m1.small |
| 86d463f1-e4fc-457f-962e-5270007694d3 | vm02  | ACTIVE | net-vlan40=10.40.40.15 | ubuntu | m1.small |
| cbad8833-54d0-4ffa-a9fa-0f26d80dfd02 | vm03  | ACTIVE | net-vlan40=10.40.40.35 | ubuntu | m1.small |
+--------------------------------------+-------+--------+------------------------+--------+----------+
```

### Verify ceph clusters

![<img>](/assets/images/2024-01-21-openstack-dr-with-ceph-rbd-mirror/ceph1.png){: width="800" }

Check replication between site-a and site-b ceph storage cluster. 

```
[site-a]$ rbd -p vms ls
86d463f1-e4fc-457f-962e-5270007694d3_disk
cbad8833-54d0-4ffa-a9fa-0f26d80dfd02_disk
d4743fb5-c314-46ee-8f84-323896936d77_disk
```

```
[site-b]$ rbd -p vms ls
86d463f1-e4fc-457f-962e-5270007694d3_disk
cbad8833-54d0-4ffa-a9fa-0f26d80dfd02_disk
d4743fb5-c314-46ee-8f84-323896936d77_disk
```

Check lock status of VMs disk on ceph. You can noticed openstack holding primary ceph cluster disks. 

```
[site-a]$ for qw in `rbd -p vms ls`; do  rbd lock list --image vms/$qw; done
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.25650  auto 140453684972992  10.30.50.11:0/2295878674
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.25643  auto 140053850363040  10.30.50.11:0/782370625
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.25651  auto 140572199228800  10.30.50.11:0/950338816
```

On secondary ceph cluster showing own IPs in lock status. 

```
[site-b]$ for qw in `rbd -p vms ls`; do  rbd lock list --image vms/$qw; done
There is 1 exclusive lock on this image.
Locker        ID                   Address
client.22536  auto 94462019434112  10.30.50.22:0/4140521941
There is 1 exclusive lock on this image.
Locker        ID                   Address
client.22536  auto 94462019432960  10.30.50.22:0/4140521941
There is 1 exclusive lock on this image.
Locker        ID                   Address
client.22536  auto 94461980277376  10.30.50.22:0/4140521941
```

Another command to verify primary onwership of disk images.

```
[site-a]$ for qw in `rbd -p vms ls`; do  rbd info vms/$qw | grep "mirroring primary"; done
	mirroring primary: true
	mirroring primary: true
	mirroring primary: true
```

```
[site-b]$ for qw in `rbd -p vms ls`; do  rbd info vms/$qw | grep "mirroring primary"; done
	mirroring primary: false
	mirroring primary: false
	mirroring primary: false
```

### Test ceph failover 

We are going to do normal failover where we will demote/promote cluster to gradually shift VMs to seconady cluster.

First Power-off all VMs

```
$ nova stop vm01 vm02 vm03
```

Check ceph lock status. You noticed no lock found on primary ceph because all VMs powered-off. 

```
[site-a]$ for qw in `rbd -p vms ls`; do  rbd lock list --image vms/$qw; done
[site-a]$
```

#### Point Openstack to Ceph2 cluster

Drop Ceph2 ceph.conf in kolla-ansible /etc/kolla/config directory and run kolla-ansible to push out changes. (You don't need to touch keyring files because auth creds are similar on both clusters. 

```
$ kolla-ansible -i <inventory_file> reconfigure -t nova 
```

#### Demote primary ceph cluster

Let's demote site-a primary cluster. You can do per image or entire pool. 

```
[site-a]$ rbd mirror image demote vms/d4743fb5-c314-46ee-8f84-323896936d77_disk
Image demoted to non-primary
```

OR entire pool you can demote. 

```
[site-a]$ rbd mirror pool demote vms
Demoted 2 mirrored images
```

Check status of demoted images. You noticed they are no more primary. 

```
[site-a]$ for qw in `rbd -p vms ls`; do  rbd info vms/$qw | grep "mirroring primary"; done
	mirroring primary: false
	mirroring primary: false
	mirroring primary: false
```

#### Promote secondary ceph cluster

![<img>](/assets/images/2024-01-21-openstack-dr-with-ceph-rbd-mirror/ceph2.png){: width="800" }

```
[site-b]$ rbd mirror pool promote vms
Promoted 3 mirrored images
```

Check status of promoted images. You can see images are primary now. 

```
[site-b]$ for qw in `rbd -p vms ls`; do  rbd info vms/$qw | grep "mirroring primary"; done
	mirroring primary: true
	mirroring primary: true
	mirroring primary: true
```

Let's start all the openstack VMs

```
$ nova start vm01 vm02 vm03
```

Check status of lock on site-b. You will notice openstack put lock on site-b disks images. 

```
[site-b]$ for qw in `rbd -p vms ls`; do  rbd lock list --image vms/$qw; done
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.22889  auto 140675211333056  10.30.50.11:0/2759728847
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.22883  auto 140410936628496  10.30.50.11:0/2270387954
There is 1 exclusive lock on this image.
Locker        ID                    Address
client.22891  auto 140188672070656  10.30.50.11:0/2354121502
```

You can use this command to check and verify which ceph cluster holding disk images. 

```
[site-b]$ rbd mirror image status vms/d4743fb5-c314-46ee-8f84-323896936d77_disk
d4743fb5-c314-46ee-8f84-323896936d77_disk:
  global_id:   08d05502-1bee-4f1a-a30d-9a598ddf2c3b
  state:       up+stopped
  description: local image is primary
  service:     ceph2.ghmncx on ceph2
  last_update: 2024-01-22 15:35:17
```

Now site-b is your primary ceph cluster and site-a is secondary cluster. Now you can setup/install rbd-mirror daemon on site-a and do replication in another direction or you can setup two-way replication on ceph. 

Enjoy! 
