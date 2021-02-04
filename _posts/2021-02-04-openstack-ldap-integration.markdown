---
title: "Openstack Integration with LDAP"
layout: post
date: 2021-02-04
image: /assets/images/2021-02-04-openstack-ldap-integration/openstack-identity-ldap.png
headerImage: true
tag:
- openstack
- openstack-ansible
- ldap
- FreeIPA
- keystone
category: blog
blog: true
author: Satish Patel
description: "Openstack Integration with LDAP"

---

Keystone includes the option to store your actors (Users and Groups) in SQL; supported databases include MySQL, PostgreSQL, and DB2. Keystone will store information such as name, password, and description. The settings for the database must be specified in Keystone’s configuration file. Essentially, Keystone is acting as an Identity Provider, which may not be the best case for everyone, and certainly not the best case for enterprise customers, For Identity centralization many companies use LDAP so in this blog i will show you how we can use LDAP for identitiy and keep all assignment information in SQL.

Example:
![<img>](/assets/images/2021-02-04-openstack-ldap-integration/openstack-ldap-sql.png) 

### Required Components 

* FreeIPA (LDAP)
* Working Openstack Environment
 

### Getting start

I am not going to show you how to configure FreeIPA LDAP here so assuming its already working and functional. Lets start with keystone configuration. 

Create "domains" directory inside /etc/keystone 

```
$ mkdir /etc/keystone/domains
```

Create ldap domain configuration file inside /etc/keystone/domains (Example: If my LDAP domain is mydomain then i will create file keystone.mydomain.conf)

```
$ cat /etc/keystone/domains/keystone.mydomain.conf
[identity]
driver = ldap

[ldap]
group_allow_create = False
group_allow_delete = False
group_allow_update = False
group_id_attribute = cn
group_member_attribute = memberof
group_name_attribute = cn
group_objectclass = organizationalUnit
group_tree_dn = cn=groups,cn=compat,dc=mydomain,dc=com
password = XXXXXXXXX
project_allow_create = False
project_allow_delete = False
project_allow_update = False
role_allow_create = False
role_allow_delete = False
role_allow_update = False
suffix = dc=mydomain,dc=com
tls_cacertfile = /etc/keystone/ssl/ipa-ldap.crt
tls_req_cert = allow
url = ldaps://ldap.mydomain.com
use_dump_member = False
use_tls = False
user = uid=svc-openstack,cn=users,cn=accounts,dc=mydomain,dc=com
user_allow_create = False
user_allow_delete = False
user_allow_update = False
user_enabled_attribute = userAccountControl
user_filter = (memberof=cn=mygroup,cn=groups,cn=accounts,dc=mydomain,dc=com)
user_id_attribute = cn
user_mail_attribute = mail
user_name_attribute = uid
user_objectclass = person
user_pass_attribute = password
user_tree_dn = cn=users,cn=accounts,dc=mydomain,dc=com
```

Make sure you have following "sql" settings in /etc/keystone/keystone.conf file. domain_specific_drivers_enabled is very important option which enable domain specific authentication service.

```
[identity]
driver = sql
domain_config_dir = /etc/keystone/domains
domain_specific_drivers_enabled = True
...
...
[assignment]
driver = sql
...
...
[resource]
cache_time = 3600
driver = sql
...
...
[revoke]
driver = sql
```

Restart keystone and verify LDAP integration.

```
$ source /root/openrc
$
$ openstack domain list
+----------------------------------+-----------+---------+--------------------+
| ID                               | Name      | Enabled | Description        |
+----------------------------------+-----------+---------+--------------------+
| 5efb8789ad624c2581ed50fa5cbbcd6f | mydomain  | True    |                    |
| default                          | Default   | True    | The default domain |
+----------------------------------+-----------+---------+--------------------+
```

Let verify wheather keystone can fetch all LDAP users in mydomain. (Hurry!! we can see all LDAP users now)

```
$ openstack user list --domain mydomain
+------------------------------------------------------------------+----------------+
| ID                                                               | Name           |
+------------------------------------------------------------------+----------------+
| 4f7b2a24bf111aac87bc6344a2637c7758694829f2f809cb1f50a5e662965be5 | ldap-user1     |
| a0f8607c33694937133ad99b4a79f6c49554a2769cbed65bec53c1971b3c3837 | ldap-user2     |
| 05fbc62924fbfc5fd95b99895aef87269c54dce0f9182e7de05266c2fdadc858 | spatel         |
| af2edc9821a13d8c7fad6f7b6f8295612f7a207d4475882e70beb8389190bc8a | ldap-user3     |
| df6163c38e9b8038f38a43a0cf854de4d2cff6324679ed56a4f680d8b5c9128a | ldap-user4     |
| 55e84029635d46846ec1339ce857baabf718151922ad3bf0aa853f43ad700727 | ldap-user5     |
+------------------------------------------------------------------+----------------+
```

Now you have to add your ldap users in desired project to give access to users. 

```
$ openstack role add --project myproject --user spatel --user-domain mydomain _member_ 
```

Now you can use your LDAP username/password to access your openstack from commandline. For horizon you need to enable multidomain support to use "mydomain" users to authenticate. 

```
$ cat /etc/openstack-dashboard/local_settings.py
...
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True 
...
```
