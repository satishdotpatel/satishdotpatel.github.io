---
title: "Virtualization With LXD"
layout: post
date: 2023-06-29
image: /assets/images/2023-06-29-lxd/lxd-logo.png
headerImage: true
tag:
- LXD
- Virtualization
- LXC
category: blog
blog: true
author: Satish Patel
description: "Virtualization with LXD"

---

In this blog, we’ll explore some of the main LXD virtual machine features and how you can use them to run your infrastructure. While LXD is mostly known for providing system containers, since the 4.0 LTS, it also natively supports virtual machines.

### Installation 

```
$ sudo apt install lxd-installer
```

### First time setup LXD

```
$ sudo lxd init
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]: no
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 192.168.1.1/24
Would you like LXD to NAT IPv4 traffic on your bridge? [default=yes]:
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: none
Would you like the LXD server to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```

### Setup storage 

There are couple of ways to setup storage backend for LXC VM/Containers. In above setup you can choose "yes" for storage to setup default. Here I'm using directory based storage. 

```
$ sudo lxc storage create pool1 dir
```
```
[root@lxd-lab-1 ~]# lxc storage ls
+-------+--------+----------------------------------------------+-------------+---------+---------+
| NAME  | DRIVER |                    SOURCE                    | DESCRIPTION | USED BY |  STATE  |
+-------+--------+----------------------------------------------+-------------+---------+---------+
| pool1 | dir    | /var/snap/lxd/common/lxd/storage-pools/pool1 |             | 10      | CREATED |
+-------+--------+----------------------------------------------+-------------+---------+---------+
```

### Launch container or VM

Launch Container

```
$ lxc launch ubuntu:22.04
```

Launch VM 

```
$ lxc launch ubuntu:22.04 --vm
```

### List Images 

```
$ lxc image list images:
$ lxc image list images: | grep -i -E 'centos'
$ lxc image list images: alpine
```

### Create Profile

```
$ lxc profile create kolla
```

Add CPUs/Memory limit to profile

```
$ lxc profile set kolla limits.cpu 4
$ lxc profile set kolla limits.memory 8GB
```

Edit Profile and add custome values 

```
$ lxc profile edit kolla
```

Example profile snippet
```
config:
  limits.cpu: "4"
  limits.memory: 8GB
description: Kolla Testing LAB
devices:
  kolla-net1:
    nictype: bridged
    parent: kolla-net1
    type: nic
  root:
    path: /
    pool: pool1
    size: 100GB
    type: disk
name: kolla
```

List profile

```
$  lxc profile list
+---------+---------------------+---------+
|  NAME   |     DESCRIPTION     | USED BY |
+---------+---------------------+---------+
| default | Default LXD profile | 9       |
+---------+---------------------+---------+
| kolla   | Kolla Testing LAB   | 9       |
+---------+---------------------+---------+
```


### Create custom network bridge

```
$ lxc network create kolla-net1 ipv6.address=none ipv4.address=none ipv4.nat=false
```

List network

```
$ lxc network list
```

Attach network to profile, Later you can apply this profile to VM or containers during launch

```
$ lxc network attach-profile kolla-net1 kolla
```

Or attach network to specific VM 

```
$ lxc network attach kolla-net1 vm-foo1
```

### Launch vm with custom profile

In following example it will apply "default" profile first and then "kolla" 

```
$ lxc launch images:ubuntu/22.10/cloud foo1 --vm -p default -p kolla 
```

Get shell 

```
$ lxc shell foo1
```

Get console 

```
$ lxc console foo1
```

Stop vm 

```
$ lxc stop foo1
```

delete vm

```
$ lxc delete foo1 --force
```

Get info 

```
$ lxc info foo1
Name: foo1
Status: RUNNING
Type: virtual-machine
Architecture: x86_64
PID: 167883
Created: 2023/06/28 19:54 UTC
Last Used: 2023/06/30 01:34 UTC

Resources:
  Processes: 28
  CPU usage:
    CPU usage (in seconds): 3586
  Memory usage:
    Memory (current): 985.51MiB
  Network usage:
    enp5s0:
      Type: broadcast
      State: UP
      Host interface: tapdca4cff3
      MAC address: 00:16:3e:ee:a6:7f
      MTU: 1500
      Bytes received: 223.49MB
      Bytes sent: 395.27MB
      Packets received: 3129929
      Packets sent: 5637425
      IP addresses:
        inet:  192.168.1.65/24 (global)
        inet6: fe80::216:3eff:feee:a67f/64 (link)
    enp6s0:
      Type: broadcast
      State: DOWN
      Host interface: tap5480eca6
      MAC address: 00:16:3e:fd:bf:23
      MTU: 1500
      Bytes received: 0B
      Bytes sent: 0B
      Packets received: 0
      Packets sent: 0
      IP addresses:
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 5.80MB
      Bytes sent: 5.80MB
      Packets received: 63445
      Packets sent: 63445
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)
```

Check configuration

```
$ lxc config show foo1
architecture: x86_64
config:
  image.architecture: amd64
  image.description: Ubuntu kinetic amd64 (20230616_08:49)
  image.os: Ubuntu
  image.release: kinetic
  image.serial: "20230616_08:49"
  image.type: disk-kvm.img
  image.variant: cloud
  volatile.base_image: af82af1972e645280c90cfdcc35e58356a027d14e6e5d4db271da1a9bece85c8
  volatile.cloud-init.instance-id: d82e34f0-2aa9-4a30-9b89-7e8662f31feb
  volatile.eth0.host_name: tapdca4cff3
  volatile.eth0.hwaddr: 00:16:3e:ee:a6:7f
  volatile.kolla-net1.host_name: tap5480eca6
  volatile.kolla-net1.hwaddr: 00:16:3e:fd:bf:23
  volatile.last_state.power: RUNNING
  volatile.last_state.ready: "false"
  volatile.uuid: 6431f00c-1ac5-4304-9db3-95394d82a4d0
  volatile.uuid.generation: 6431f00c-1ac5-4304-9db3-95394d82a4d0
  volatile.vsock_id: "19"
devices: {}
ephemeral: false
profiles:
- default
- kolla
stateful: false
description: ""
```

Default root disk size is 10G but you can increase it with following command

```
$ lxc config device override foo1 root size=150GB
$ lxc stop foo1
$ lxc start foo1
```

Increase CPU/Memory resources

```
$ lxc config set os-comp1 limits.cpu 8
$ lxc config set os-comp1 limits.memory 16GB
```

### Remote managment of LXD nodes

Type the following command server2 to enable a remote access via API:

```
$ lxc config set core.https_address 192.168.1.6:8443
```

Set the password for remote lxd daemon:

```
$ lxc config set core.trust_password MyTopSecretPassword
```

On Local lxd node add remote node server2

```
$ lxc remote add server2 192.168.1.6
```

You can list your remotes and you will see “server2” listed as follows:

```
$ lxc remote list
```



