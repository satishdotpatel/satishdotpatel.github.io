---
title: "Openstack Skyline Dashboard"
layout: post
date: 2021-08-15
image: /assets/images/2021-08-15-openstack-skyline-dashboard/skyline-ac130.png
headerImage: true
tag:
- openstack
- horizon
- skyline
category: blog
blog: true
author: Satish Patel
description: "Openstack Skyline Dashboard"

---

As you know horizon is long standing web-interface for Openstack but today i am going show you how to install Skyline dahsboard. Skyline is an OpenStack dashboard optimized by UI and UE. It has a modern technology stack and ecology, is easier for developers to maintain and operate by users, and has higher concurrency performance.

### Prerequisite

* Working openstack environment
* VM to run Skyline app (Ubuntu 20.04)

### Add skyline account in Openstack

```
# Source the admin credentials
$ source admin-openrc

# Create the skyline user
$ openstack user create --domain default --password-prompt skyline
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 1qaz2wsx3edc4rfv5tgb6yhn7ujm8ikl |
| name                | skyline                          |
| options             | {}                               |
| password_expires_at | 2020-08-08T08:08:08.123456       |
+---------------------+----------------------------------+

# Add the admin role to the skyline user:
$ openstack role add --project service --user skyline admin
```

### Configure MySQL DB

You can run skyline using MySQL DB or SQLite DB, In my example i am using MySQL.

```
root@skyline:~# apt-get install mariadb-server
```

Create database

```
root@skyline:~# mysql -u root -p
MariaDB [(none)]> CREATE DATABASE IF NOT EXISTS skyline DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
Query OK, 1 row affected (0.001 sec)
```

Grant proper access to the databases

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON skyline.* TO 'skyline'@'localhost' IDENTIFIED BY 'MySkylineDBPassword';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON skyline.* TO 'skyline'@'%'  IDENTIFIED BY 'MySkylineDBPassword';
Query OK, 0 rows affected (0.001 sec)
```

### Install Docker

Its very easy and quick to install Skyline in Docker container.

```
root@skyline:~# apt-get install docker.io
```

### Install Skyline app 

Check out repo

```
root@skyline:~# cd /root/
root@skyline:~# git clone https://opendev.org/skyline/skyline-apiserver
```

Setup skyline.yaml file

```
root@skyline:~# mkdir /etc/skyline
root@skyline:~# cp skyline-apiserver/etc/skyline.yaml.sample /etc/skyline/skyline.yaml
```

Modify the following parameters of skyline.yaml file according to the actual environment.

```
default:
  access_token_expire: 3600
  access_token_renew: 1800
  cors_allow_origins: []
  database_url: mysql://skyline:MySkylineDBPassword@localhost:3306/skyline
  debug: false
  log_dir: ./log
  secret_key: aCtmgbcUqYUy_HNVg5BDXCaeJgJQzHJXwqbXr0Nmb2o
  session_name: session
developer:
  show_raw_sql: false
openstack:
  base_domains:
  - heat_user_domain
  base_roles:
  - keystone_system_admin
  - keystone_system_reader
  - keystone_project_admin
  - keystone_project_member
  - keystone_project_reader
  - nova_system_admin
  - nova_system_reader
  - nova_project_admin
  - nova_project_member
  - nova_project_reader
  - cinder_system_admin
  - cinder_system_reader
  - cinder_project_admin
  - cinder_project_member
  - cinder_project_reader
  - glance_system_admin
  - glance_system_reader
  - glance_project_admin
  - glance_project_member
  - glance_project_reader
  - neutron_system_admin
  - neutron_system_reader
  - neutron_project_admin
  - neutron_project_member
  - neutron_project_reader
  - heat_system_admin
  - heat_system_reader
  - heat_project_admin
  - heat_project_member
  - heat_project_reader
  - placement_system_admin
  - placement_system_reader
  - panko_system_admin
  - panko_system_reader
  - panko_project_admin
  - panko_project_member
  - panko_project_reader
  - ironic_system_admin
  - ironic_system_reader
  - octavia_system_admin
  - octavia_system_reader
  - octavia_project_admin
  - octavia_project_member
  - octavia_project_reader
  default_region: RegionOne
  extension_mapping:
    fwaas_v2: neutron_firewall
    vpnaas: neutron_vpn
  interface_type: public
  keystone_url: https://os-lab.example.com:5000/v3/
  nginx_prefix: /api/openstack
  reclaim_instance_interval: 604800
  service_mapping:
    compute: nova
    identity: keystone
    image: glance
    network: neutron
    orchestration: heat
    placement: placement
    volumev3: cinder
  system_admin_roles:
  - admin
  - system_admin
  system_project: service
  system_project_domain: Default
  system_reader_roles:
  - system_reader
  system_user_domain: Default
  system_user_name: skyline
  system_user_password: skyline123
setting:
  base_settings:
  - flavor_families
  - gpu_models
  - usb_models
  flavor_families:
  - architecture: x86_architecture
    categories:
    - name: general_purpose
      properties: []
    - name: compute_optimized
      properties: []
    - name: memory_optimized
      properties: []
    - name: high_clock_speed
      properties: []
  - architecture: heterogeneous_computing
    categories:
    - name: compute_optimized_type_with_gpu
      properties: []
    - name: visualization_compute_optimized_type_with_gpu
      properties: []
  gpu_models:
  - nvidia_t4
  usb_models:
  - usb_c
```

Deploy skyline app in docker, repo contain Dockerfile. 

```
# go to container directory
root@skyline:~# cd /root/skyline-apiserver/container/

# Run the skyline_bootstrap container to bootstrap
root@skyline:~# docker run -d --name skyline_bootstrap -e KOLLA_BOOTSTRAP="" -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml --net=host 99cloud/skyline:latest

# Check bootstrap is normal `exit 0`
root@skyline:~# docker logs skyline_bootstrap
```

Run the skyline service after bootstrap is complete

```
# Remove bootstrap
root@skyline:~# docker rm -f skyline_bootstrap

# Run docker 
root@skyline:~# docker run -d --name skyline --restart=always -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml --net=host 99cloud/skyline:latest
```

### Verify

```
root@skyline:~# docker ps
CONTAINER ID   IMAGE                    COMMAND              CREATED        STATUS        PORTS     NAMES
33929d68df7c   99cloud/skyline:latest   "start_service.sh"   15 hours ago   Up 15 hours             skyline
```

You can now access the dashboard: https://<ip_address>:8080

![<img>](/assets/images/2021-08-15-openstack-skyline-dashboard/skyline-login.png){: width="850" }

