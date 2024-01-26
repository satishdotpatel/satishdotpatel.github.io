---
title: "Ceph Rados Gateway Integration Openstack Keystone"
layout: post
date: 2024-01-25
image: /assets/images/2024-01-25-ceph-rados-gateway-integration-with-openstack-keystone/ceph-keystone.png
headerImage: true
tag:
- Ceph
- RGW
- Cephadm
- Openstack
- Keystone
category: blog
blog: true
author: Satish Patel
description: "Ceph Rados Gateway Integration Openstack Keyston"

---

On this post I'll focus on the integration of Ceph as Object storage with Openstack. Ceph can be integrated with the OpenStack identity management service, 'Keystone'. With this integration, the Ceph RGW is configured to accept keystone tokens for user authority. So, any user who is validated by Keystone will get rights to access the RGW.

### Scope 

* Deploy RGW using cephadm
* Create User account on Openstack keyston
* Create openstack API endpoint for ceph
* Add Keystone user in ceph 
* Test integration

![<img>](/assets/images/2024-01-25-ceph-rados-gateway-integration-with-openstack-keystone/ceph-openstack-rgw.png){: width="800" }

### Deploy Ceph RGW Service

Assuming we have working ceph cluster and going to turn on Object storage services (rgw). 

```
$ radosgw-admin realm create --rgw-realm=example.com --default
$ radosgw-admin zonegroup create --rgw-zonegroup=default  --master --default
$ radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=us-east --master --default
$ radosgw-admin period update --rgw-realm=example.com --commit
```

Deply rgw on ceph1 and ceph2 nodes. 

```
$ ceph orch apply rgw my-rgw --realm=example.com --zone=us-east --placement="2 ceph1 ceph2"
```

Verify status

```
$ ceph orch ps --daemon_type=rgw
NAME                       HOST   PORTS  STATUS        REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
rgw.my-rgw.ceph1.rjymsf  ceph1  *:80   running (4h)     5m ago   6h     119M        -  18.2.0   14060fbd7be7  f23419dffc46
rgw.my-rgw.ceph2.pracgl  ceph2  *:80   running (4h)     5m ago   6h     124M        -  18.2.0   14060fbd7be7  91883e0f7a8f
```

### Create swift user on Keyston

Don't get confused with swift as a storage service but Swift will not be deployed at all. You'll take advantage of the Swift API compatibility on the Rados Gateway. 

Go to openstack and run following commands. 

```
$ openstack service create --name=swift --description="Swift Service" object-store
$ openstack user create --domain default --password-prompt swift
$ openstack role add --project service --user swift admin
```

### Create keyston endpoint for ceph 

Create endpoint for ceph so openstack services can talk to ceph. 

```
$ openstack endpoint create --region RegionOne object-store public "http://ceph.example.com:80/swift/v1/AUTH_%(project_id)s"
$ openstack endpoint create --region RegionOne object-store internal "http://ceph.example.com:80/swift/v1/AUTH_%(project_id)s"
$ openstack endpoint create --region RegionOne object-store admin "http://ceph.example.com:80/swift/v1/AUTH_%(project_id)s"
```

### Add keyston setting on ceph

I have two rgw service running on ceph1 and ceph2 so I have to set it for two rgw instances. You can find their NAME in following output. 

```
$ ceph orch ps --daemon_type=rgw
NAME                       HOST   PORTS  STATUS        REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
rgw.my-rgw.ceph1.rjymsf  ceph1  *:80   running (4h)     5m ago   6h     119M        -  18.2.0   14060fbd7be7  f23419dffc46
rgw.my-rgw.ceph2.pracgl  ceph2  *:80   running (4h)     5m ago   6h     124M        -  18.2.0   14060fbd7be7  91883e0f7a8f
```

rgw.my-rgw.ceph1.rjymsf

```
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_url https://openstack.example.com:5000
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_api_version 3
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_admin_user swift
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_admin_password timetime
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_admin_domain default
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_admin_project service
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_s3_auth_use_keystone true
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_implicit_tenants false
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_accepted_roles admin,member,_member_
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_keystone_verify_ssl false
ceph config set client.rgw.my-rgw.ceph1.rjymsf rgw_swift_account_in_url true
```

rgw.my-rgw.ceph2.pracgl

```
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_url https://openstack.example.com:5000
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_api_version 3
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_admin_user swift
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_admin_password timetime
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_admin_domain default
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_admin_project service
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_s3_auth_use_keystone true
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_implicit_tenants false
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_accepted_roles admin,member,_member_
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_keystone_verify_ssl false
ceph config set client.rgw.my-rgw.ceph2.pracgl rgw_swift_account_in_url true
```

Redeploy rgw service to refresh config. 

```
$ ceph orch redeploy rgw.my-rgw
```

Verify setting using following command. 

```
$ ceph config dump
```

### Test Integration 

Create a container. As said the container is like a directory 

```
$ openstack container create foo1
$ openstack container create bar1
```

List them

```
$ openstack container list
+------+
| Name |
+------+
| bar1 |
| foo1 |
+------+
```

Go back to ceph and run bucket list. There yo go! 

```
$ radosgw-admin bucket list
[
    "bar1",
    "foo1"
]
```

Let's upload a file using swift command. As I said earlier we are using swift API not a storage. 

```
$ echo "Hello World :)" > helloworld.txt
$ swift upload bar1 helloworld.txt
$ swift list bar1
helloworld.txt
```

You can see stats 

```
$ swift stat bar1
                      Account: AUTH_670dea6393824b3ca32426474848f0e1
                    Container: bar1
                      Objects: 1
                        Bytes: 15
                     Read ACL:
                    Write ACL:
                      Sync To:
                     Sync Key:
                  X-Timestamp: 1706238800.70154
X-Container-Bytes-Used-Actual: 4096
             X-Storage-Policy: default-placement
              X-Storage-Class: STANDARD
                Last-Modified: Fri, 26 Jan 2024 03:13:20 GMT
                   X-Trans-Id: tx000004f8a146e265f89fa-0065b325f0-1e3a959-sin
       X-Openstack-Request-Id: tx000004f8a146e265f89fa-0065b325f0-1e3a959-sin
                Accept-Ranges: bytes
                 Content-Type: text/plain; charset=utf-8
                   Connection: Keep-Alive
```

You can do same upload/download objects from horizon UI. 

![<img>](/assets/images/2024-01-25-ceph-rados-gateway-integration-with-openstack-keystone/horizon-rgw.png){: width="800" }


Enjoy!!! 

