---
title: "Openstack cloud autoscaling with Senlin"
layout: post
date: 2020-09-04
image: /assets/images/2020-09-04-openstack-senlin-autoscaling/senlin-autoscale.png
headerImage: true
tag:
- senlin
category: blog
blog: true
author: Satish Patel
description: "Openstack cloud autoscaling with Senlin"

---

I am going to show you how senlin works with openstack cloud to scale up/down production workload on-demand. Senlin is a clustering service for OpenStack clouds. It creates and operates clusters of homogeneous objects exposed by other OpenStack services. You can feed external metrics to senlin to trigger scaling up/down your workload based on-demand, Just like AWS ASG.

<!--more-->
[Senlin Documentation](https://docs.openstack.org/senlin/latest/install/).

# Installation of senlin 

Notes: Assuming you already have basic openstack cloud up and running.

Install RDO repo for ussuri release:

```
$ yum install centos-release-openstack-ussuri
```

Install senlin packages:

```
$ yum install openstack-senlin-engine.noarch \
  openstack-senlin-api.noarch \
  openstack-senlin-common.noarch \
  python3-senlinclient.noarch
```

MySQL Database setup:

```
$ mysql -u root -p

MariaDB [(none)]> CREATE DATABASE senlin DEFAULT CHARACTER SET utf8;

MariaDB [(none)]> GRANT ALL ON senlin.* TO 'senlin'@'localhost' \
  IDENTIFIED BY 'SENLIN_DBPASS';
GRANT ALL ON senlin.* TO 'senlin'@'%' \
  IDENTIFIED BY 'SENLIN_DBPASS';
```

Create the senlin users in keystone:

```
$ source /root/openrc
$
$ openstack user create --project services --password-prompt senlin
User Password:
Repeat User Password:
$
$ openstack role add --project services --user senlin admin
$
$ openstack service create --name senlin --description "Senlin Service" clustering
$
```

Create the senlin service API endpoints:

Notes: Make sure you don't have any other services using 8778 port (nova placement API default port is 8778 also)

```
$ openstack endpoint create senlin --region RegionOne public http://controller:8778
$
$ openstack endpoint create senlin --region RegionOne admin http://controller:8778
$
$ openstack endpoint create senlin --region RegionOne internal http://controller:8778
```

Senline configuration file should look like following:

```
[DEFAULT]
debug = true
transport_url = rabbit://senlin:SENLINE_MQ_PASS@172.28.40.141:5671//senlin?ssl=1

[database]
connection = mysql+pymysql://senlin:SENLINE_DB_PASS@172.28.40.9/senlin?charset=utf8

[keystone_authtoken]
service_token_roles_required = True
auth_type = password
auth_url = http://172.28.40.230:5000/v3
www_authenticate_uri = http://172.28.40.230:5000/v3
project_domain_id = default
user_domain_id = default
project_name = service
username = senlin
password = SENLINE_KEYSTONE_PASS

[authentication]
auth_url = http://172.28.40.230:5000/v3
service_username = senlin
service_password = SENLINE_KEYSTONE_PASS
service_project_name = service

[oslo_messaging_notifications]
driver = messaging

[oslo_messaging_rabbit]
ssl = True
rabbit_notification_exchange = senlin
rabbit_notification_topic = notifications
```

Populate the Senlin database:

```
$ senlin-manage db_sync
```

Start the Senlin services:

```
$ systemctl enable openstack-senlin-api.service \
   openstack-senlin-conductor.service \
   openstack-senlin-engine.service \
   openstack-senlin-health-manager.service

$ systemctl start openstack-senlin-api.service \
   openstack-senlin-conductor.service \
   openstack-senlin-engine.service \
   openstack-senlin-health-manager.service
```
