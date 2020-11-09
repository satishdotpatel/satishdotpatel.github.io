---
title: "Build Openstack cloud using Openstack-Ansible"
layout: post
date: 2020-11-08
image: /assets/images/2020-11-08-build-openstack-cloud-using-openstack-ansible/foo.png
headerImage: true
tag:
- Openstack
- Openstack-Ansible
- CentOS
category: blog
blog: true
author: Satish Patel
description: "Build Openstack cloud using Openstack-Ansible"

---

I'm going to show you how to build openstack cloud using openstack-ansible and in later i will show you how to scale it for production workload. I am using Openstack-Ansible deployment tool to deploy production grade openstack.

For simplicity i am going to use following components:

- 1 x Controller
- 1 x Compute
- LinuxBridge for network (vlan provider)


## Prepare Controller Node

We are using centOS so lets install centOS 8.2 on controller node and prepare for Networking. 

/etc/sysconfig/network-scripts/ifcfg-bond0
```
# Bond0 Interface
NAME=bond0
DEVICE=bond0
BOOTPROTO=none
ONBOOT=yes
BONDING_OPTS="mode=1 miimon=500 downdelay=1000 primary="eno49" primary_reselect=always"
BRIDGE=br-vlan
```
/etc/sysconfig/network-scripts/ifcfg-bond0.64
```
# VLAN64 for br-host Interface 
NAME=bond0.64
DEVICE=bond0.64
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-host
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-bond0/ifcfg-bond0.65
```
# VLAN65 for br-mgmt Interface
NAME=bond0.65
DEVICE=bond0.65
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-mgmt
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-host
```
# br-host Bridge
DEVICE=br-host
NAME=br-host
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.64.0.111
NETMASK=255.255.248.0
GATEWAY=10.64.0.1
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-mgmt
```
# br-mgmt Bridge
DEVICE=br-mgmt
NAME=br-mgmt
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.65.0.111
NETMASK=255.255.248.0
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-vlan
```
# br-vlan Bridge
DEVICE=br-vlan
NAME=br-vlan
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```

Restart NetworkManager service 

```
[root@infra-lxb-1 ~]# systemctl restart NetworkManager
```

#### Download Openstack-Ansible

1. Clone the latest stable release of the OpenStack-Ansible Git repository in the /opt/openstack-ansible directory.
```
[root@infra-lxb-1 ~]# git clone -b stable/ussuri https://github.com/openstack/openstack-ansible.git /opt/openstack-ansible
```
2. Change to the /opt/openstack-ansible directory, and run the Ansible bootstrap script:
```
[root@infra-lxb-1 ~]# cd /opt/openstack-ansible/
[root@infra-lxb-1 openstack-ansible]# scripts/bootstrap-ansible.sh
```

#### Configure Openstack-Ansible

1. Copy the contents of following directory. 
```
[root@infra-lxb-1 ~]# cp -avrp /opt/openstack-ansible/etc/openstack_deploy /etc/.
```
2. Create file /etc/openstack_deploy/openstack_user_config.yml
```
---
cidr_networks:
  container: 10.65.0.0/21  # br-mgmt subnet for contole plane

used_ips:
  - "10.65.0.1,10.65.0.150" # Reserved some IPs 

global_overrides:
  internal_lb_vip_address: 10.65.0.111  # Internal VIP of haproxy
  external_lb_vip_address: 10.64.0.111  # External VIP of haproxy
  management_bridge: "br-mgmt"

  provider_networks:
    - network:
        group_binds:
          - all_containers
          - hosts
        type: "raw"
        container_bridge: "br-mgmt"
        container_interface: "eth1"
        container_type: "veth"
        ip_from_q: "container"
        is_container_address: true
        is_ssh_address: true
    - network:
        container_bridge: "br-vlan"
        container_type: "veth"
        container_interface: "eth11"
        type: "vlan"
        range: "66:68"
        net_name: "vlan"
        group_binds:
          - neutron_linuxbridge_agent

# RabbitMQ, MySQL, Memcache
shared-infra_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Repo 
repo-infra_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Heat
os-infra_hosts:
  infra-lxb-1:
    affinity:
      heat_apis_container: 0
      heat_engine_container: 0
    ip: 10.65.0.111

# Keystone
identity_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Neutron
network_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Nova Placement
placement-infra_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Haproxy LB
haproxy_hosts:
  infra-lxb-1:
    ip: 10.65.0.111

# Log server
log_hosts:
  infra-lxb-1:
    ip: 10.65.0.111
```
3. Edit /etc/openstack_deploy/user_variables.yml to disable security hardending (This is lab so not very important)
```
apply_security_hardening: false
```
4. Generate secrets file
```
[root@infra-lxb-1 ~]# cd /opt/openstack-ansible
[root@infra-lxb-1 openstack-ansible]# ./scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml
```

#### Run Playbooks to Install Controller

1. Run the host setup playbook, It will prepare containers.
```
[root@infra-lxb-1 openstack-ansible]# cd /opt/openstack-ansible/playbooks/
[root@infra-lxb-1 playbooks]# openstack-ansible setup-hosts.yml
```
2. Run the infrastructure setup playbook, It install deploy RabbitMQ, MySQL & Memcache services. 
```
[root@infra-lxb-1 playbooks]# openstack-ansible setup-infrastructure.yml
```
3. Run the OpenStack setup playbook, It will deploy your openstack components like keystone, neutron, nova etc. (it will take longer time to finish)
```
[root@infra-lxb-1 playbooks]# openstack-ansible setup-openstack.yml
```

#### Validation of Deployment of Controller

1. Determine the name of the utility container:
```
[root@infra-lxb-1 ~]# lxc-ls | grep utility
infra-lxb-1_utility_container-085107e1
```
2. Access the utility container:
```
[root@infra-lxb-1 ~]# lxc-attach -n infra-lxb-1_utility_container-085107e1
[root@infra-lxb-1-utility-container-085107e1 ~ ]#
```
3. Source the admin tenant credentials:
```
[root@infra-lxb-1-utility-container-085107e1 ~ ]# source /root/openrc
```
4. List your openstack users: 
```
[root@infra-lxb-1-utility-container-085107e1 ~ ]# openstack user list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 6251ec10ac4447cc8a5c0522a904532c | admin     |
| 8ff02444e012450898d96a41ad81ea6a | glance    |
| 07336f214cbe45c7a377159e9edffddb | nova      |
| 90e6c8f760cd42a8895327eb30f517bf | placement |
| 1e0e57c1fcf54e6d8b248e1979a2d4e5 | neutron   |
+----------------------------------+-----------+
```
5. Access Horizon GUI using External haproxy VIP IPaddress: https://10.64.0.111

Username: admin
Password: stored in /etc/openstack_deploy/user_secrets.yml file. 
```
[root@infra-lxb-1 openstack_deploy]# cat /etc/openstack_deploy/user_secrets.yml | grep keystone_auth_admin_password
keystone_auth_admin_password: 7bea3f701708a0c1f42e0dde52ba67b1d5e4a2407fe9249d4afbab14aed0
```

## Prepare Compute Node

Install CentOS 8.2 and prepare node for networking. (We are going to use LinuxBridge) 

/etc/sysconfig/network-scripts/ifcfg-bond0
```
# Bond0 Interface
NAME=bond0
DEVICE=bond0
BOOTPROTO=none
ONBOOT=yes
BONDING_OPTS="mode=1 miimon=500 downdelay=1000 primary="eno49" primary_reselect=always"
BRIDGE=br-vlan
```
/etc/sysconfig/network-scripts/ifcfg-bond0.64
```
# VLAN64 for br-host Interface
NAME=bond0.64
DEVICE=bond0.64
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-host
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-bond0/ifcfg-bond0.65
```
# VLAN65 for br-mgmt Interface
NAME=bond0.65
DEVICE=bond0.65
BOOTPROTO=static
VLAN=yes
ONPARENT=yes
BRIDGE=br-mgmt
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-host
```
# br-host Bridge
DEVICE=br-host
NAME=br-host
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.64.0.112
NETMASK=255.255.248.0
GATEWAY=10.64.0.1
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-mgmt
```
# br-mgmt Bridge
DEVICE=br-mgmt
NAME=br-mgmt
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
IPADDR=10.65.0.112
NETMASK=255.255.248.0
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```
/etc/sysconfig/network-scripts/ifcfg-br-vlan
```
# br-vlan Bridge
DEVICE=br-vlan
NAME=br-vlan
BOOTPROTO=none
TYPE=Bridge
ONPARENT=yes
DELAY=0
STP=no
ETHTOOL_OPTS="-K ${DEVICE} tso off gso off gro off sg off"
```

Restart NetworkManager service (Recommonded to reboot whole system to verify all interface comes up clean)

```
[root@infra-lxb-1 ~]# systemctl restart NetworkManager
```

#### Add compute node to controller node configuration

1. Go to Controller node and add compute defination in /etc/openstack_deploy/openstack_user_config.yml 
```
# Compute nodes
compute_hosts:
  compute-lxb-1:
    ip: 10.65.0.112
```
2. Copy ssh public key to compute node for passwd-less access to run ansible-playbooks
```
[root@infra-lxb-1 ~]# ssh-copy-id 10.64.0.112
```
3. Run Playbook to setup compute node, following command run 3 playbooks to deploy software. 
```
[root@infra-lxb-1 ~]# cd /opt/openstack-ansible/playbooks/
[root@infra-lxb-1 playbooks]# openstack-ansible setup-hosts.yml os-nova-install.yml os-neutron-install.yml --limit compute-lxb-1
```

#### Validation

Run following command on controller to validate compute node added or not.  

```
[root@infra-lxb-1 ~]# lxc-attach -n infra-lxb-1_utility_container-085107e1
[root@infra-lxb-1-utility-container-085107e1 ~ ]# source /root/openrc
[root@infra-lxb-1-utility-container-085107e1 ~ ]# openstack hypervisor list
+----+-------------------------+-----------------+-------------+-------+
| ID | Hypervisor Hostname     | Hypervisor Type | Host IP     | State |
+----+-------------------------+-----------------+-------------+-------+
|  1 | compute-lxb-1.v1v0x.net | QEMU            | 10.65.0.112 | up    |
+----+-------------------------+-----------------+-------------+-------+
```
