---
title: "Build custom OS by LinuxKit"
layout: post
date: 2022-08-14
image: /assets/images/2022-08-14-build-custom-os-by-linuxkit/linuxkit-logo.png
headerImage: true
tag:
- Openstack
- LinuxKit
- Containers
- Docker
category: blog
blog: true
author: Satish Patel
description: "Build custom OS by LinuxKit"

---

I heard from one of my colleague about LinuxKit and I can't wait to get my hands dirty so thought give it a try and see what it's capable of. You will get suprised by power of it in end of this blog. In this blog i will use some example of Linuxkit to build small version of redis server running on custom OS. I would also build qcow2 image to upload on openstack to run as a VM. Lets start...

What is LinuxKit?

LinuxKit is a toolkit for building custom minimal, immutable Linux distributions. https://github.com/linuxkit/linuxkit

#### Lab 

* Ubuntu 20.04 to install linuxkit and build OS
* Working openstack environment to deploy VM. 

#### Install Docker

Install Docker CE version because its required by LinuxKit

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:

```
$ sudo apt-get update
$ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

2. Add Docker’s official GPG key: 

```
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

3. Use the following command to set up the repository:

```
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Install docker-ce

```
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

#### Install LinuxKit

To compile linuxkit we need to install development tools first.

```
$ sudo apt install build-essential
```

Clone linuxkit repo and compile.

```
$ git clone https://github.com/linuxkit/linuxkit
$ cd linuxkit
$ make
$ make install
```

Above command will install linuxkit binary in /usr/local/bin

#### Build Redis Server

Lets build our first redis server. Go to linuxkit/example directory where you can find many YAML example files.

redis-os.yml

```
# Minimal YAML to run a redis server (used at DockerCon'17)
# connect: nc localhost 6379
kernel:
  image: linuxkit/kernel:5.10.104
  cmdline: "console=tty0 console=ttyS0 console=ttyAMA0 console=ttysclp0"
init:
  - linuxkit/init:8f1e6a0747acbbb4d7e24dc98f97faa8d1c6cec7
  - linuxkit/runc:f01b88c7033180d50ae43562d72707c6881904e4
  - linuxkit/containerd:de1b18eed76a266baa3092e5c154c84f595e56da
onboot:
  - name: dhcpcd
    image: linuxkit/dhcpcd:52d2c4df0311b182e99241cdc382ff726755c450
    command: ["/sbin/dhcpcd", "--nobackground", "-f", "/dhcpcd.conf", "-1"]
services:
  - name: getty
    image: linuxkit/getty:76951a596aa5e0867a38e28f0b94d620e948e3e8
    env:
     - INSECURE=true
  - name: redis
    image: redis:4.0.5-alpine
    capabilities:
     - CAP_NET_BIND_SERVICE
     - CAP_CHOWN
     - CAP_SETUID
     - CAP_SETGID
     - CAP_DAC_OVERRIDE
    net: host
```

In above YAML file you can see we are using Kernel, init to start runc/containerd, dhcpcd for onboot and services for getty terminal and redis server. 

Lets build redis-os using YAML file. 

```
$ linuxkit build redis-os.yml
Extract kernel image: docker.io/linuxkit/kernel:5.10.104
Add init containers:
Process init image: docker.io/linuxkit/init:8f1e6a0747acbbb4d7e24dc98f97faa8d1c6cec7
Process init image: docker.io/linuxkit/runc:f01b88c7033180d50ae43562d72707c6881904e4
Process init image: docker.io/linuxkit/containerd:de1b18eed76a266baa3092e5c154c84f595e56da
Add onboot containers:
  Create OCI config for linuxkit/dhcpcd:52d2c4df0311b182e99241cdc382ff726755c450
Add service containers:
  Create OCI config for linuxkit/getty:76951a596aa5e0867a38e28f0b94d620e948e3e8
  Create OCI config for redis:4.0.5-alpine
Create outputs:
  redis-os-kernel redis-os-initrd.img redis-os-cmdline
```

If you noticed in above command it generated 3 files, kernel, initrd for rootfs and redis cmdline in same directory. Lets boot custom OS which container redis servers. You need qemu to boot OS so please install kvm/qemu on system so you can test your new image. 

Following command will boot your OS using local kvm/qemu hypervisor.

```
$ linuxkit run redis-os
...
...
Welcome to LinuxKit

                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
          {                       /  ===-
           \______ O           __/
             \    \         __/
              \____\_______/

[    2.414985] cgroup: cgroup: disabling cgroup2 socket matching due to net_prio or net_cls activation
[    2.449940] 8021q: adding VLAN 0 to HW filter on device eth0
[    2.676891] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
getty: starting getty for ttyS0

linuxkit-3a39f3081c68 login: root (automatic login)

Welcome to LinuxKit!

NOTE: This system is namespaced.
The namespace you are currently in may not be the root.
System services are namespaced; to access, use `ctr -n services.linuxkit ...`
(ns: getty) linuxkit-3a39f3081c68:~#
``` 

In this strip version of OS just running kernel, init, dhcpd, containerd and redis-server and nothing else. 

```
(ns: getty) linuxkit-3a39f3081c68:~# pstree
init-+-containerd---7*[{containerd}]
     |-containerd-shim-+-tini---rungetty.sh-+-rungetty.sh---login---sh
     |                 |                    `-rungetty.sh---login---sh---pstree
     |                 `-10*[{containerd-shim}]
     `-containerd-shim-+-redis-server---3*[{redis-server}]
                       `-11*[{containerd-shim}]
```

containerd running in namespace so you have to use following command to see running services.

```
(ns: getty) linuxkit-3a39f3081c68:~# ctr -n services.linuxkit c ls
CONTAINER    IMAGE    RUNTIME
getty        -        io.containerd.runc.v2
redis        -        io.containerd.runc.v2
```

To get access of container use following command

```
(ns: getty) linuxkit-3a39f3081c68:~# ctr -n services.linuxkit t exec -t --exec-id bash_1 redis sh
/data #
/data # redis-cli
127.0.0.1:6379> ping
PONG
```

#### Build image for Openstack

Lets build image and use format command to produce vhd image for openstack. 

```
$ linuxkit build -format vhd redis-os.yml
```

Copy redis-os.vhd image to openstack and upload to glance registery. 

```
$ openstack image create linuxkit-redis --disk-format vhd --container-format bare --file redis-os.vhd
```

Create vm 

```
$ openstack server create --image linuxkit --flavor m1.small --nic net-id=private linuxkit-vm
```

List vm to find out IP.

```
$ openstack server list
+--------------------------------------+------+--------+-------------------------------------------------------+-------------+----------+
| ID                                   | Name | Status | Networks                                              | Image       | Flavor   |
+--------------------------------------+------+--------+-------------------------------------------------------+-------------+----------+
| 5742ac34-7adb-4518-ad07-0f69ec057518 | lk1  | ACTIVE | private=10.0.0.38, fdc4:3574:a4:0:f816:3eff:fe26:c0e8 | linuxkit-vm | m1.small |
+--------------------------------------+------+--------+-------------------------------------------------------+-------------+----------+
```

Lets test redis from remote hosts in LAN.

```
$ telnet 10.0.0.38 6379
ping
+PONG
```

Enjoy your strip down version of custom OS. 


