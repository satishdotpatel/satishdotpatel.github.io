---
title: "Openstack Manila Integration to GlusterFS with Ganesha-NFS"
layout: post
date: 2022-03-15
image: /assets/images/2022-03-15-openstack-manila-integration-with-glusterfs/manila-share-1.png
headerImage: true
tag:
- Ganesha-NFS
- GlusterFS
- Openstack
- Manila
- Kolla-Ansible
category: blog
blog: true
author: Satish Patel
description: "Openstack Manila Integration to GlusterFS with Ganesha-NFS"

---

This blogpost introduces the shared file system service for OpenStack Manila. In this lab i am going to integrate GlusterFS with Ganesha-NFS plugin to provide NFS base shared file system to all my vms.  

### Prerequisit 

We need working openstack deployment. In my case i am running kolla-ansible. 

* control node 
* compute1 node
* storage1 node (Running glusterfs & ganesha-nfs services)


![<img>](/assets/images/2022-03-15-openstack-manila-integration-with-glusterfs/manila-glusterfs-design.png){: width="400" }

#### Install/Setup GlusterFS server

Install glusterfs on storage1 server.

```
$ sudo apt-get update
$ sudo apt-get install glusterfs
$ sudo apt -y install glusterfs-server
$ sudo systemctl enable --now glusterd
$ sudo service glusterd start
```

Create directory for gluster volume

```
$ sudo mkdir -p /gluster/manila-vol
```

Create gluster volume and enable quota on volume. 

```
$ sudo gluster volume create manila-vol storage1:/gluster/manila-vol force
$ sudo gluster volume start manila-vol
$ sudo gluster volume quota gluster enable
```

Verify

```
$ sudo gluster volume status
Status of volume: manila-vol
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick storage1:/gluster/manila-vol          49153     0          Y       3668

Task Status of Volume manila-vol
------------------------------------------------------------------------------
There are no active volume tasks
```

#### Install/Setup Ganesha-NFS Server

Install ganesha-nfs server on storage1 server.

```
$ sudo apt -y install nfs-ganesha-gluster
```

Configuration ganesha.conf file. (10.40.1.12 is storage1 ip)

```
$ cat /etc/ganesha/ganesha.conf
# create new
NFS_CORE_PARAM {
    # possible to mount with NFSv3 to NFSv4 Pseudo path
    mount_path_pseudo = true;
    # NFS protocol
    Protocols = 3,4;
}
EXPORT_DEFAULTS {
    # default access mode
    Access_Type = RW;
}
EXPORT {
    # uniq ID
    Export_Id = 101;
    # mount path of Gluster Volume
    Path = "/manila-vol";
    FSAL {
    	# any name
        name = GLUSTER;
        # hostname or IP address of this Node
        hostname="10.40.1.12";
        # Gluster volume name
        volume="manila-vol";
    }
    # config for root Squash
    Squash="No_root_squash";
    # NFSv4 Pseudo path
    Pseudo="/gluster/manila-vol";
    # allowed security options
    SecType = "sys";
}
LOG {
    # default log level
    Default_Log_Level = WARN;
}
```

Enable and restart service

```
$ sudo systemctl enable nfs-ganesha
$ sudo systemctl restart nfs-ganesha
```

Verify

```
$ sudo showmount -e localhost
Export list for storage1:
/gluster/manila-vol (everyone)
```

#### Install Manila on kolla controller server

Enable Manila in /etc/kolla/globals.yml

```
enable_manila: "yes"
```

Create manila nfs share file at /etc/kolla/config/manila.conf (I have used root passwrd here but you can use ssh key for more secure deployment)

```
$ sudo cat /etc/kolla/config/manila.conf
[DEFAULT]
enabled_share_backends=ganesha
enabled_share_protocols = NFS,CIFS,GLUSTERFS

[ganesha]
share_backend_name = ganesha
glusterfs_nfs_server_type = Ganesha
glusterfs_target=root@10.40.1.12:/manila-vol
glusterfs_server_password=<<ROOT PASSWD>>
share_driver = manila.share.drivers.glusterfs.GlusterfsShareDriver
ganesha_service_name = nfs-ganesha
driver_handles_share_servers = False
glusterfs_ganesha_server_ip=10.40.1.12
glusterfs_ganesha_server_username=root
glusterfs_ganesha_server_password=<<ROOT PASSWD>>
```

Deploy manila 

```
$ sudo kolla-ansible -i multinode deploy
```

After successful deployment you will see following container on control server. 

```
$ sudo docker ps | grep manila
220d71402848   kolla/ubuntu-source-manila-share:wallaby            "dumb-init --single-…"   3 days ago     Up 4 hours (healthy)              manila_share
cee0579f1c7c   kolla/ubuntu-source-manila-scheduler:wallaby        "dumb-init --single-…"   3 days ago     Up 2 days (healthy)               manila_scheduler
dbe7cf07bd9a   kolla/ubuntu-source-manila-data:wallaby             "dumb-init --single-…"   3 days ago     Up 2 days (healthy)               manila_data
c4479813e23f   kolla/ubuntu-source-manila-api:wallaby              "dumb-init --single-…"   3 days ago     Up 2 days (healthy)               manila_api
```

Verify

```
$ source /etc/kolla/admin-openrc.sh
$ manila service-list
+----+------------------+-------------------------------+------+---------+-------+----------------------------+
| Id | Binary           | Host                          | Zone | Status  | State | Updated_at                 |
+----+------------------+-------------------------------+------+---------+-------+----------------------------+
| 1  | manila-data      | os-ovn-ctrl.example.net       | nova | enabled | up    | 2022-03-15T19:16:44.620520 |
| 2  | manila-scheduler | os-ovn-ctrl.example.net       | nova | enabled | up    | 2022-03-15T19:16:51.697674 |
| 3  | manila-share     | os-ovn-ctrl.example@ganesha   | nova | enabled | up    | 2022-03-15T19:16:49.927305 |
+----+------------------+-------------------------------+------+---------+-------+----------------------------+
```

To create a share type, we need to disable snapshot because this is a directory mapping type, so snapshots are not supported.

```
$ manila type-create ganesha False
$ manila type-key ganesha set share_backend_name=ganesha
$ manila type-key ganesha set snapshot_support=False
```

#### Validation 

First create 2G share for VMs

```
$ manila create nfs 2 --name myshare-2g --share-type ganesha
+---------------------------------------+--------------------------------------+
| Property                              | Value                                |
+---------------------------------------+--------------------------------------+
| id                                    | dcf610f1-a0ab-42f2-b55d-23cb58a36b87 |
| size                                  | 2                                    |
| availability_zone                     | None                                 |
| created_at                            | 2022-03-15T19:28:04.875107           |
| status                                | creating                             |
| name                                  | myshare-2g                           |
| description                           | None                                 |
| project_id                            | 3a864fd72b92403cb352ec9236709435     |
| snapshot_id                           | None                                 |
| share_network_id                      | None                                 |
| share_proto                           | NFS                                  |
| metadata                              | {}                                   |
| share_type                            | cdaf3423-0eba-4b29-bf00-488eca5acaf9 |
| is_public                             | False                                |
| snapshot_support                      | False                                |
| task_state                            | None                                 |
| share_type_name                       | ganesha                              |
| access_rules_status                   | active                               |
| replication_type                      | None                                 |
| has_replicas                          | False                                |
| user_id                               | 8863bb3eb7c746db8c9e3eccc4346fb7     |
| create_share_from_snapshot_support    | False                                |
| revert_to_snapshot_support            | False                                |
| share_group_id                        | None                                 |
| source_share_group_snapshot_member_id | None                                 |
| mount_snapshot_support                | False                                |
| progress                              | None                                 |
| share_server_id                       | None                                 |
| host                                  |                                      |
+---------------------------------------+--------------------------------------+
```

Verify

```
$ manila list
+--------------------------------------+------------+------+-------------+-----------+-----------+-----------------+-----------------------------------------+-------------------+
| ID                                   | Name       | Size | Share Proto | Status    | Is Public | Share Type Name | Host                                    | Availability Zone |
+--------------------------------------+------------+------+-------------+-----------+-----------+-----------------+-----------------------------------------+-------------------+
| dcf610f1-a0ab-42f2-b55d-23cb58a36b87 | myshare-2g | 2    | NFS         | available | False     | ganesha         | os-ovn-ctrl.example.net@ganesha#ganesha | nova              |
+--------------------------------------+------------+------+-------------+-----------+-----------+-----------------+-----------------------------------------+-------------------+
```

Allow access to share. (I have used whole /24 subnet to access, it will allow any vms i create)

```
$ manila access-allow myshare-2g ip 10.40.255.0/24
+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| id           | dc54ae95-d5e4-4c86-a6f7-a2f55987e80c |
| share_id     | dcf610f1-a0ab-42f2-b55d-23cb58a36b87 |
| access_level | rw                                   |
| access_to    | 10.40.255.0/24                       |
| access_type  | ip                                   |
| state        | queued_to_apply                      |
| access_key   | None                                 |
| created_at   | 2022-03-15T19:33:33.322397           |
| updated_at   | None                                 |
| metadata     | {}                                   |
+--------------+--------------------------------------+
```

Above command will create export file on ganesha-server which you can see in following path on storage1 server.

```
$ cat /etc/ganesha/export.d/share-95ffa738-6b35-4296-b1c8-d8995a500a14--df705d40-09e0-4fb0-bb9e-76df6ef789ae.conf
EXPORT {
    Export_Id = 106;
    Path = "/share-95ffa738-6b35-4296-b1c8-d8995a500a14";
    FSAL {
        Name = "GLUSTER";
        Hostname = "10.40.1.12";
        Volume = "manila-vol";
        Volpath = "/share-95ffa738-6b35-4296-b1c8-d8995a500a14";
    }
    Pseudo = "/share-95ffa738-6b35-4296-b1c8-d8995a500a14--df705d40-09e0-4fb0-bb9e-76df6ef789ae";
    SecType = "sys";
    Tag = "df705d40-09e0-4fb0-bb9e-76df6ef789ae";
    CLIENT {
        Clients = 10.40.255.0/24;
        Access_Type = "RW";
    }
    Squash = "None";
}
```

Verify new share export. Now we are ready to mount this share on two vms.

```
$ sudo showmount -e storage1.example.net
Export list for storage1.example.net:
/gluster/manila-vol                                                               (everyone)
/share-95ffa738-6b35-4296-b1c8-d8995a500a14--df705d40-09e0-4fb0-bb9e-76df6ef789ae 10.40.255.0/24
```

Create two vms and attach floating IP based on your network provider.  

```
+--------------------------------------+-------+--------+------------+-------------+-----------------------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks                          |
+--------------------------------------+-------+--------+------------+-------------+-----------------------------------+
| 16c872d4-bf32-40cb-96c5-02b40330ba89 | vm8-1 | ACTIVE | -          | Running     | demo-net=10.0.0.100, 10.40.255.27 |
| 46c3fb93-10b9-44b5-8459-fcd8d913c934 | vm8-2 | ACTIVE | -          | Running     | demo-net=10.0.0.82, 10.40.255.35  |
+--------------------------------------+-------+--------+------------+-------------+-----------------------------------+
```

Mount new share on both vms using following command

```
$ ssh centos@10.40.255.27
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Tue Mar 15 19:25:20 2022 from 10.30.1.230
[centos@vm8-1 ~]$ sudo su -
[root@vm8-1 ~]#
[root@vm8-1 ~]# mount -t nfs 10.40.1.12:/share-95ffa738-6b35-4296-b1c8-d8995a500a14--df705d40-09e0-4fb0-bb9e-76df6ef789ae /mnt
[root@vm8-1 ~]# df -h /mnt
Filesystem                                                                                    Size  Used Avail Use% Mounted on
10.40.1.12:/share-95ffa738-6b35-4296-b1c8-d8995a500a14--df705d40-09e0-4fb0-bb9e-76df6ef789ae  2.0G     0  2.0G   0% /mnt
```

Leti's create file on shared filesystem

```
[root@vm8-1 ~]# cd /mnt/
[root@vm8-1 mnt]# dd if=/dev/zero of=foo.img bs=1024 count=102400
102400+0 records in
102400+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.91495 s, 115 MB/s
```

Delete share (WARNNING!!! it will delete your data when you delete share) 

```
$ manila delete myshare-2g
```

Enjoy!!!
