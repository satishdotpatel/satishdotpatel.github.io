---
title: "Designate Integration with PowerDNS"
layout: post
date: 2021-02-05
image: /assets/images/2021-02-05-designate-integration-with-powerdns/openstack-ansible.png
headerImage: true
tag:
- openstack
- openstack-ansible
- designate
- PowerDNS
category: blog
blog: true
author: Satish Patel
description: "Designate Integration with PowerDNS"

---

Designate is a multi-tenant DNSaaS service for OpenStack. It provides a REST API with integrated Keystone authentication. It can be configured to auto-generate records based on Nova and Neutron actions. Designate supports a variety of DNS servers including Bind9 and PowerDNS 4, In this blog i am going to show you how designate integrate with PowerDNS 4.

mDNS is python based mini DNS service which use AXFR to transfer zone to PowerDNS. 

![<img>](/assets/images/2021-02-05-designate-integration-with-powerdns/designate-dns.png) 

### Required Components 

* PowerDNS (Version 4.1.4-1 )
* Openstack Environment (openstack-ansible Victoria release)
 

### PowerDNS

PowerDNS configuration, pdns running on port 5300.

```
$ cat /etc/pdns/pdns.conf
setuid=pdns
setgid=pdns
launch=bind
allow-dnsupdate-from=127.0.0.0/8,10.0.0.0/8,::1
allow-notify-from=10.65.0.0/21
api=yes
api-key=SuperSecretPowerDNSapiPassword
disable-axfr=no
dnsupdate=yes
local-port=5300
log-dns-details=yes
log-dns-queries=yes
loglevel=999
master=no
slave=yes
slave-cycle-interval=60
webserver=yes
webserver-address=10.65.0.10
webserver-allow-from=127.0.0.0/8,10.65.0.0/21,::1
webserver-password=Password123
launch=gmysql
gmysql-host=127.0.0.1
gmysql-user=pdns-admin
gmysql-password=MysqlPassword
gmysql-dbname=pdns
```
PowerDNS Recursor configuration, It's running on port 53.

```
$ cat /etc/pdns-recursor/recursor.conf
setuid=pdns-recursor
setgid=pdns-recursor
allow-from=0.0.0.0/0
dont-query=100.64.0.0/10, 169.254.0.0/16, 192.168.0.0/16, ::1/128, fc00::/7, fe80::/10, 0.0.0.0/8, 192.0.0.0/24, 192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24, 240.0.0.0/4, ::/96, ::ffff:0:0/96, 100::/64, 2001:db8::/32
local-address=127.0.0.1,10.65.0.10
max-cache-ttl=3600
setgid=pdns
setuid=pdns
webserver=yes
tux.com=10.64.0.10:5300
```

### Openstack-Ansible Designate Configuration

Add following settings in user_varaibles.yml file.

```
dns_hosts:
  - { ip: 10.65.0.10, name: ns1.tux.com, port: 5300 }
  - { ip: 10.65.0.11, name: ns2.tux.com, port: 5300 }

_designate_pools_yaml_nameservers: |
  {% for item in dns_hosts %}
  - host: "{{ item.ip }}"
    port: {{ item.port }}
  {% endfor %}

_designate_pools_yaml_ns_records: |
  {% for item in dns_hosts %}
  - hostname: "{{ item.name }}."
    priority: 1
  {% endfor %}

_designate_pools_yaml_targets: |
  {% for item in dns_hosts %}
  - type: pdns4
    description: PowerDNS 4
    masters:
  {% for mdns_item in groups['designate_mdns'] | map('extract', hostvars, 'container_address') | list %}
      - host: "{{ mdns_item }}"
        port: 5354
  {% endfor %}
    options:
      host: "{{ item.ip }}"
      port: {{ item.port }}
      api_endpoint: http://{{ item.ip }}:8081
      api_token: SuperSecretPowerDNSapiPassword
  {% endfor %}

designate_pools_yaml:
  - name: "default"
    description: pool for PowerDNS running on infra hosts
    attributes: {}
    ns_records: "{{ _designate_pools_yaml_ns_records | from_yaml }}"
    nameservers: "{{ _designate_pools_yaml_nameservers | from_yaml }}"
    targets: "{{ _designate_pools_yaml_targets | from_yaml }}"
```

Add following variables for neutron-server in user_variables.yml in my case i am adding in /etc/openstack_deploy/group_vars/neutron_server.yml.

```
$ cat /etc/openstack_deploy/group_vars/neutron_server.yml
# Enable designate support for neutron
neutron_designate_enabled: True
neutron_dns_domain: phx.v1v0x.net.
neutron_ml2_conf_ini_overrides:
  ml2:
    extension_drivers: port_security,subnet_dns_publish_fixed_ip
```

Run designate and neutron playbooks to deploy changes. 

```
$ openstack-ansible os-designate-install.yml --limit designate_api
$ openstack-ansible os-neutron-install.yml --limit neutron_server
```

Notes: I would highly recommand to restart designate services on all 3 infra nodes containers. (I believe there is a bug in playbook which doesn't restart designate after deploying pool config)

### Designate configuration validation

After running playbooks you will see it created /etc/designate/pools.yaml file. 

```
-   attributes: {}
    description: pool for PowerDNS running on infra hosts
    name: default
    nameservers:
    -   host: 10.65.0.10
        port: 5300
    -   host: 10.65.0.11
        port: 5300
    ns_records:
    -   hostname: ns1.tux.com.
        priority: 1
    -   hostname: ns2.tux.com.
        priority: 1
    targets:
    -   description: PowerDNS 4
        masters:
        -   host: 10.65.7.6
            port: 5354
        -   host: 10.65.7.219
            port: 5354
        -   host: 10.65.7.113
            port: 5354
        options:
            api_endpoint: http://10.65.0.10:8081
            api_token: SuperSecretPowerDNSapiPassword
            host: 10.65.0.10
            port: 5300
        type: pdns4
    -   description: PowerDNS 4
        masters:
        -   host: 10.65.7.6
            port: 5354
        -   host: 10.65.7.219
            port: 5354
        -   host: 10.65.7.113
            port: 5354
        options:
            api_endpoint: http://10.65.0.11:8081
            api_token: SuperSecretPowerDNSapiPassword
            host: 10.65.0.11
            port: 5300
        type: pdns4
 ```

 ### Validate designate functionality

 Let's create domain 

 ```
 $ openstack zone create --email spatel@tux.com tux.com.
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| action         | CREATE                               |
| attributes     |                                      |
| created_at     | 2021-02-05T06:05:12.000000           |
| description    | None                                 |
| email          | spatel@tux.com                       |
| id             | fa760c59-b04c-4da8-b533-f4b781c88e92 |
| masters        |                                      |
| name           | tux.com.                             |
| pool_id        | 7533562a-cfb7-4ac0-a42f-a4e58142465e |
| project_id     | f1502c79c70f4651be8ffc7b844b584f     |
| serial         | 1612505112                           |
| status         | PENDING                              |
| transferred_at | None                                 |
| ttl            | 3600                                 |
| type           | PRIMARY                              |
| updated_at     | None                                 |
| version        | 1                                    |
+----------------+--------------------------------------+
```

Check status 

```
$ openstack recordset list tux.com.
+--------------------------------------+----------+------+-------------------------------------------------------------+--------+--------+
| id                                   | name     | type | records                                                     | status | action |
+--------------------------------------+----------+------+-------------------------------------------------------------+--------+--------+
| 89267742-0e01-42e1-9f64-35e2a73e268f | tux.com. | NS   | ns2.tux.com.                                                | ACTIVE | NONE   |
|                                      |          |      | ns1.tux.com.                                                |        |        |
| b3afa6df-f80d-494b-a5d4-447b68c6e5b0 | tux.com. | SOA  | ns2.tux.com. spatel.tux.com. 1612505112 3575 600 86400 3600 | ACTIVE | NONE   |
+--------------------------------------+----------+------+-------------------------------------------------------------+--------+--------+
```

Add records 

```
$ openstack recordset create --record '10.65.0.99' --type A tux.com. www
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| action      | CREATE                               |
| created_at  | 2021-02-05T06:07:39.000000           |
| description | None                                 |
| id          | 6ca0db53-178e-4100-9119-c5b06748394f |
| name        | www.tux.com.                         |
| project_id  | f1502c79c70f4651be8ffc7b844b584f     |
| records     | 10.65.0.99                           |
| status      | PENDING                              |
| ttl         | None                                 |
| type        | A                                    |
| updated_at  | None                                 |
| version     | 1                                    |
| zone_id     | fa760c59-b04c-4da8-b533-f4b781c88e92 |
| zone_name   | tux.com.                             |
+-------------+--------------------------------------+
```

Verify records

```
$ openstopenstack recordset list tux.com.
+--------------------------------------+--------------+------+-------------------------------------------------------------+--------+--------+
| id                                   | name         | type | records                                                     | status | action |
+--------------------------------------+--------------+------+-------------------------------------------------------------+--------+--------+
| 89267742-0e01-42e1-9f64-35e2a73e268f | tux.com.     | NS   | ns2.tux.com.                                                | ACTIVE | NONE   |
|                                      |              |      | ns1.tux.com.                                                |        |        |
| b3afa6df-f80d-494b-a5d4-447b68c6e5b0 | tux.com.     | SOA  | ns2.tux.com. spatel.tux.com. 1612505259 3575 600 86400 3600 | ACTIVE | NONE   |
| 6ca0db53-178e-4100-9119-c5b06748394f | www.tux.com. | A    | 10.65.0.99                                                  | ACTIVE | NONE   |
+--------------------------------------+--------------+------+-------------------------------------------------------------+--------+--------+
```

Lets verify on PowerDNS using mysql database query, As you can see tux.com has been created and pointing to 3 designated containers ipaddress which is running mDNS service on port 5354.

```
$ mysql pdns -e "select * from domains where name='tux.com'"
+-----+---------------+---------------------------------------------------+------------+-------+-----------------+---------+
| id  | name          | master                                            | last_check | type  | notified_serial | account |
+-----+---------------+---------------------------------------------------+------------+-------+-----------------+---------+
| 181 | tux.com       | 10.65.7.113:5354 10.65.7.219:5354 10.65.7.6:5354  | 1612504450 | SLAVE |            NULL |         |
+-----+---------------+---------------------------------------------------+------------+-------+-----------------+---------+
```

At this point your designate successfully integrated with PowerDNS. 

### Neutron Integration with designate 

Lets tell neutron to create DNS records when we create openstack instances using neutron network. (make sure you are using trailing dot (.) in domain)

```
$ openstack network set net_vlan_66 --dns-domain tux.com.
$ openstack subnet set sub_vlan_66 --dns-publish-fixed-ip
```

Lets create openstack instance to verify.

```
$ openstack server create --flavor m1.small --image centos-7 --nic net-id=net_vlan_66 --key-name ssh-key --security-group 7b46125c-833d-483c-811c-e91a6e81f64f vm-1
```

Verify vm-1 A records in designate and powerDNS. Hooray!!! 

```
$ openstopenstack recordset list tux.com.
+--------------------------------------+---------------+------+-------------------------------------------------------------+--------+--------+
| id                                   | name          | type | records                                                     | status | action |
+--------------------------------------+---------------+------+-------------------------------------------------------------+--------+--------+
| 89267742-0e01-42e1-9f64-35e2a73e268f | tux.com.      | NS   | ns2.tux.com.                                                | ACTIVE | NONE   |
|                                      |               |      | ns1.tux.com.                                                |        |        |
| b3afa6df-f80d-494b-a5d4-447b68c6e5b0 | tux.com.      | SOA  | ns2.tux.com. spatel.tux.com. 1612505259 3575 600 86400 3600 | ACTIVE | NONE   |
| 6ca0db53-178e-4100-9119-c5b06748394f | www.tux.com.  | A    | 10.65.0.99                                                  | ACTIVE | NONE   |
| ddfd0f0b-6835-46ee-b76e-3a49907e6a86 | vm-1.tux.com. | A    | 10.66.2.221                                                 | ACTIVE | NONE   |
+--------------------------------------+---------------+------+-------------------------------------------------------------+--------+--------+
```

Enjoy!!! 


