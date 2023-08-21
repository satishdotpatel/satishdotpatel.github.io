---
title: "Build Multinode Kolla-Ansible Openstack LAB using LXD"
layout: post
date: 2023-07-02
image: /assets/images/2023-07-02-build-multinode-kolla-lab-using-lxd/kolla-logo.png
headerImage: true
tag:
- LXD
- Kolla
- Kolla-ansible
- Openstack
- LAB
category: blog
blog: true
author: Satish Patel
description: "Build Multinode Kolla-Ansible Openstack LAB using LXD"

---

In this blog, I'm going to build openstack multinode lab using kolla-ansible with help of LXD virtualization. 

### Multinode lxd lab

![<img>](/assets/images/2023-07-02-build-multinode-kolla-lab-using-lxd/lxd-kolla-lab.png){: width="800" }

* Ansible node: kolla-mgmt
* Docker registry: docker-registry
* Loadbalancer: ha1, ha2
* Controller node: ctrl1, ctrl2, ctrl3
* Compute node: comp1, comp2

### Installation 

In my last blog post we saw how to use LXD to quickly spin up VMs and containers, We are going to use LXD to build our openstack kolla lab. 

```
$ lxc launch ubuntu:22.04 ctrl1 --vm -p default -p kolla
$ lxc launch ubuntu:22.04 ctrl2 --vm -p default -p kolla
$ lxc launch ubuntu:22.04 ctrl3 --vm -p default -p kolla
$ lxc launch ubuntu:22.04 comp1 --vm -p default -p kolla
$ lxc launch ubuntu:22.04 comp2 --vm -p default -p kolla
$ lxc launch ubuntu:22.04 kolla-mgmt -p default -p kolla
$ lxc launch ubuntu:22.04 docker-registry --vm -p default -p kolla
$ lxc launch images:ubuntu/22.04/cloud ha1 --vm -p default -p kolla
$ lxc launch images:ubuntu/22.04/cloud ha2 --vm -p default -p kolla
```
**NOTES:** If you see carefully, I have spun up kolla-mgmt as a container image instead of vm because its our ansible management node. ha1 and ha2 is using cloud image because we want ip_vs kernel module support to run keepalived & HAProxy. docker-registry is nothing but just docker images registry. 

```
[root@lxd-lab-1 ~]# lxc list
+-----------------+---------+------------------------+------+-----------------+-----------+
|      NAME       |  STATE  |          IPV4          | IPV6 |      TYPE       | SNAPSHOTS |
+-----------------+---------+------------------------+------+-----------------+-----------+
| comp1           | RUNNING | 192.168.1.103 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| comp2           | RUNNING | 192.168.1.210 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| ctrl1           | RUNNING | 192.168.1.149 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| ctrl2           | RUNNING | 192.168.1.165 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| ctrl3           | RUNNING | 192.168.1.147 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| docker-registry | RUNNING | 192.168.1.107 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| ha1             | RUNNING | 192.168.1.65 (enp5s0)  |      | VIRTUAL-MACHINE | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
| ha2             | RUNNING | 192.168.1.205 (enp5s0) |      | VIRTUAL-MACHINE | 0         |
|                 |         | 192.168.1.100 (enp5s0) |      |                 |           |
+-----------------+---------+------------------------+------+-----------------+-----------+
| kolla-mgmt      | RUNNING | 192.168.1.93 (eth0)    |      | CONTAINER       | 0         |
+-----------------+---------+------------------------+------+-----------------+-----------+
```

### Download kolla images

I wrote small script to download kolla images from public repo and push them to local docker registry. (In our case its docker-registry vm )

Go to docker-registry VM and create local registry for docker

```
[root@lxd-lab-1 ~]# lxc shell docker-registry
root@docker-registry:~# docker run -d  --network host  --name registry  --restart=always  -e REGISTRY_HTTP_ADDR=0.0.0.0:4000  -v registry:/var/lib/registry  registry:2
```

Create daemon.json file in /etc/docker directory and restart docker daemon. 

```
root@docker-registry:~# cat /etc/docker/daemon.json
{
    "bridge": "none",
    "insecure-registries": [
        "docker-registry:4000"
    ],
    "ip-forward": false,
    "iptables": false,
    "log-opts": {
        "max-file": "5",
        "max-size": "50m"
    }
}
```

Now download script to pull kolla images https://github.com/satishdotpatel/kolla-pull-images/ 

```
[root@lxd-lab-1 ~]# git clone https://github.com/satishdotpatel/kolla-pull-images/
[root@lxd-lab-1 ~]# cd kolla-pull-images
[root@lxd-lab-1 ~]# chmod 700 kolla-pull-images.sh
[root@lxd-lab-1 ~]# ./kolla-pull-images.sh
```
NOTES: Make sure you have correct name/ip entry in /etc/hosts file to resolve hostname. 

If all goes well then you can list all images.

```
root@docker-registry:~# docker images
REPOSITORY                                                    TAG                IMAGE ID       CREATED       SIZE
docker-registry:4000/openstack.kolla/horizon                  zed-ubuntu-jammy   a90071bea8be   6 days ago    1.11GB
docker-registry:4000/openstack.kolla/nova-compute             zed-ubuntu-jammy   03bf2030f9e9   6 days ago    1.47GB
docker-registry:4000/openstack.kolla/nova-novncproxy          zed-ubuntu-jammy   c7f07c233777   6 days ago    1.22GB
docker-registry:4000/openstack.kolla/glance-api               zed-ubuntu-jammy   45fb0bc16757   6 days ago    1.04GB
docker-registry:4000/openstack.kolla/nova-ssh                 zed-ubuntu-jammy   ee4d86a32eb7   6 days ago    1.11GB
docker-registry:4000/openstack.kolla/nova-api                 zed-ubuntu-jammy   c2e1a22849fb   6 days ago    1.11GB
docker-registry:4000/openstack.kolla/nova-conductor           zed-ubuntu-jammy   3d865bf24674   6 days ago    1.11GB
docker-registry:4000/openstack.kolla/heat-api-cfn             zed-ubuntu-jammy   c07b41976f63   6 days ago    965MB
docker-registry:4000/openstack.kolla/nova-scheduler           zed-ubuntu-jammy   2f51040a765a   6 days ago    1.11GB
docker-registry:4000/openstack.kolla/heat-api                 zed-ubuntu-jammy   4308687071f4   6 days ago    965MB
docker-registry:4000/openstack.kolla/heat-engine              zed-ubuntu-jammy   70cef4eb204a   6 days ago    965MB
docker-registry:4000/openstack.kolla/neutron-server           zed-ubuntu-jammy   c91e09bfbe9a   6 days ago    1.05GB
docker-registry:4000/openstack.kolla/placement-api            zed-ubuntu-jammy   089192ea63ea   6 days ago    899MB
docker-registry:4000/openstack.kolla/cinder-volume            zed-ubuntu-jammy   01c938e25654   6 days ago    1.36GB
docker-registry:4000/openstack.kolla/cinder-backup            zed-ubuntu-jammy   6ba053f1b1aa   6 days ago    1.35GB
docker-registry:4000/openstack.kolla/neutron-metadata-agent   zed-ubuntu-jammy   5291e782a611   6 days ago    1.04GB
docker-registry:4000/openstack.kolla/cinder-api               zed-ubuntu-jammy   cae1a1eb698d   6 days ago    1.32GB
docker-registry:4000/openstack.kolla/cinder-scheduler         zed-ubuntu-jammy   90bb4e7485f6   6 days ago    1.32GB
docker-registry:4000/openstack.kolla/keystone-ssh             zed-ubuntu-jammy   de5a1feab718   6 days ago    953MB
docker-registry:4000/openstack.kolla/keystone-fernet          zed-ubuntu-jammy   ab58015fcb3b   6 days ago    951MB
docker-registry:4000/openstack.kolla/keystone                 zed-ubuntu-jammy   a8c993608769   6 days ago    947MB
docker-registry:4000/openstack.kolla/kolla-toolbox            zed-ubuntu-jammy   ef3915665dbe   6 days ago    816MB
docker-registry:4000/openstack.kolla/ovn-nb-db-server         zed-ubuntu-jammy   303eb4001f0e   6 days ago    270MB
docker-registry:4000/openstack.kolla/ovn-sb-db-server         zed-ubuntu-jammy   66e81d780c2f   6 days ago    270MB
docker-registry:4000/openstack.kolla/ovn-northd               zed-ubuntu-jammy   8e1036bd07e6   6 days ago    270MB
docker-registry:4000/openstack.kolla/mariadb-server           zed-ubuntu-jammy   b5ccc6dbe5b4   6 days ago    596MB
docker-registry:4000/openstack.kolla/ovn-controller           zed-ubuntu-jammy   7a9c90b8cbd5   6 days ago    270MB
docker-registry:4000/openstack.kolla/mariadb-clustercheck     zed-ubuntu-jammy   01a562432a38   6 days ago    314MB
docker-registry:4000/openstack.kolla/nova-libvirt             zed-ubuntu-jammy   088c360b0cca   6 days ago    987MB
docker-registry:4000/openstack.kolla/openvswitch-db-server    zed-ubuntu-jammy   16a118ae074a   6 days ago    262MB
docker-registry:4000/openstack.kolla/openvswitch-vswitchd     zed-ubuntu-jammy   1c7bfdd16f60   6 days ago    262MB
docker-registry:4000/openstack.kolla/keepalived               zed-ubuntu-jammy   61ae7d342950   6 days ago    261MB
docker-registry:4000/openstack.kolla/fluentd                  zed-ubuntu-jammy   fe74611e53d8   6 days ago    519MB
docker-registry:4000/openstack.kolla/rabbitmq                 zed-ubuntu-jammy   f055d6e11826   6 days ago    304MB
docker-registry:4000/openstack.kolla/memcached                zed-ubuntu-jammy   0a7bd8ceba59   6 days ago    250MB
docker-registry:4000/openstack.kolla/cron                     zed-ubuntu-jammy   3ad32d54a974   6 days ago    250MB
docker-registry:4000/openstack.kolla/haproxy                  zed-ubuntu-jammy   0c4254acf5fc   6 days ago    257MB
registry                                                      2                  4bb5ea59f8e0   2 weeks ago   24MB
```

### Prepare Ansible node for kolla-ansible

Go to kolla-mgmt container and run following command to install kolla-ansible for zed release. 

```
[root@lxd-lab-1 ~]# lxc shell kolla-mgmt
root@kolla-mgmt:~# apt update
root@kolla-mgmt:~# apt install python3-dev libffi-dev gcc libssl-dev python3-docker ca-certificates python3-venv
root@kolla-mgmt:~# python3 -m venv /opt/venv-kolla
root@kolla-mgmt:~# source /opt/venv-kolla/bin/activate
root@kolla-mgmt:~# pip install -U pip
root@kolla-mgmt:~# pip install ansible==5.10.0
root@kolla-mgmt:~# pip install git+https://opendev.org/openstack/kolla-ansible@stable/zed
root@kolla-mgmt:~# kolla-ansible install-deps
root@kolla-mgmt:~# mkdir -p /etc/kolla
root@kolla-mgmt:~# cp /opt/venv-kolla/share/kolla-ansible/etc_examples/kolla/globals.yml /etc/kolla/.
root@kolla-mgmt:~# cp /opt/venv-kolla/share/kolla-ansible/etc_examples/kolla/passwords.yml /etc/kolla/.
root@kolla-mgmt:~# cp /opt/venv-kolla/share/kolla-ansible/ansible/inventory/multinode /etc/kolla/.
root@kolla-mgmt:~# kolla-genpwd
```

Edit /etc/kolla/multinode file and adjust your hosts according roles or groups. 

```
[control]
ctrl1
ctrl2
ctrl3

[network]
ctrl1
ctrl2
ctrl3

[compute]
comp1
comp2

[monitoring]
#monitoring01

[storage]
#storage01

[haproxy]
ha1
ha2

...
...

[loadbalancer:children]
#network
haproxy

...
...
[output ommited]
```

Edit /etc/kolla/globals.yml 

```
---
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
docker_namespace: "openstack.kolla"
openstack_release: "zed"
kolla_internal_vip_address: "192.168.1.100"
kolla_external_vip_address: "{{"{{kolla_internal_vip_address}}"}}"
network_interface: "enp5s0"
neutron_external_interface: "enp6s0"
neutron_plugin_agent: "ovn"
enable_neutron_agent_ha: "yes"
dhcp_agents_per_network: "3"
enable_neutron_provider_networks: "yes"
enable_haproxy: "yes"
keystone_token_provider: 'fernet'
nova_compute_virt_type: "kvm"
libvirt_tls: "no"
libvirt_enable_sasl: "false"

## Use local registry for docker
docker_registry: "docker-registry:4000"
docker_registry_insecure: "false"
docker_custom_config:
  insecure-registries:
    - docker-registry:4000

## Pin Docker package
docker_apt_package: "docker.io"

## RabbitMQ HA
om_enable_rabbitmq_high_availability: True

## Enable mariadb backup
enable_mariabackup: "yes"
```
NOTES: 192.168.1.100 is free IP from LXD pool for handover to keepalived floating ip. 

Generate ssh key and push it to all nodes to run ansible. 

```
root@kolla-mgmt:~# ssh-keygen
```

Add all hostname and IPs in /etc/hosts to all nodes, I have regex to prepare /etc/hosts file using following command. Run this on LXD host node

```
[root@lxd-lab-1 ~]# lxc ls -f csv | awk -F'[,(]' '{print $3 "\t" $1}'
192.168.1.103 	comp1
192.168.1.210 	comp2
192.168.1.149 	ctrl1
192.168.1.165 	ctrl2
192.168.1.147 	ctrl3
192.168.1.107 	docker-registry
192.168.1.65 	ha1
192.168.1.205 	ha2
192.168.1.93 	kolla-mgmt
```

Source kolla virtual environment 

```
root@kolla-mgmt:~# source /opt/venv-kolla/bin/activate
```
Run bootstrap-server

```
(venv-kolla) root@kolla-mgmt:~# kolla-ansible -i /etc/kolla/multinode bootstrap-servers
```

If all looks good then run precheck 

```
(venv-kolla) root@kolla-mgmt:~# kolla-ansible -i /etc/kolla/multinode prechecks
```

Finally deploy

```
(venv-kolla) root@kolla-mgmt:~# kolla-ansible -i /etc/kolla/multinode deploy
```

If all looks good then run post-deploy to generate creds

```
(venv-kolla) root@kolla-mgmt:~# kolla-ansible -i /etc/kolla/multinode post-deploy
```

Install openstack sdk client 

```
(venv-kolla) root@kolla-mgmt:~# pip3 install python-openstackclient
```

Obtain creds 

```
(venv-kolla) root@kolla-mgmt:~# source /etc/kolla/admin-openrc.sh
```

Run init-runonce script to setup image/flavor/network etc. 

```
(venv-kolla) root@kolla-mgmt:~# /opt/venv-kolla/share/kolla-ansible/init-runonce
```

Launch demo1 vm 

```
(venv-kolla) root@kolla-mgmt:~# openstack server create --image cirros --flavor m1.tiny --key-name mykey --network demo-net demo1
(venv-kolla) root@kolla-mgmt:~# openstack server list
+--------------------------------------+-------+--------+---------------------+--------+---------+
| ID                                   | Name  | Status | Networks            | Image  | Flavor  |
+--------------------------------------+-------+--------+---------------------+--------+---------+
| 67a0e9a8-b32a-427d-a28f-98b36f44b1b4 | demo3 | ACTIVE | demo-net=10.0.0.103 | cirros | m1.tiny |
+--------------------------------------+-------+--------+---------------------+--------+---------+
```

To expose horizon GUI use following iptables rules on LXD host.

```
[root@lxd-lab-1 ~]# iptables -t nat -A PREROUTING -i agge -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
```

Enjoy your lab!!!
