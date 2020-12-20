---
title: "Ubuntu 20.04 autoinstall using PXE Server"
layout: post
date: 2020-12-20
image: /assets/images/2020-12-20-ubuntu-20-04-autoinstall-pxe/ubuntu-pxe.png
headerImage: true
tag:
- Linux
- Ubuntu
- PXE
category: blog
blog: true
author: Satish Patel
description: "Ubuntu 20.04 autoinstall using PXE Server"

---

With Ubuntu 20.04, Canonical switched from the Debian installer to their own autoinstaller. This means that you have to build a new PXEBoot environment.

Before you start, you need to have:

* Working TFTP Server
* Working HTTP Server
* Working DHCP Server

I'm not going to show you how to configure TFTP/HTTP/DHCP because there are planty of documentation available on internet. I'm assuming you have all 3 components already setup and running. 

### Download Ubuntu 20.04 ISO

Download Ubuntu ISO 20.04 from https://releases.ubuntu.com/20.04/ and copy iso file to http web path. 
```
[root@spatel ~]# ls /var/www/html/pxe_repo/ubuntu-20.04.1-live-server-amd64.iso
/var/www/html/pxe_repo/ubuntu-20.04.1-live-server-amd64.iso
```

### TFTP server prep

Add following snippet in file /var/lib/tftpboot/pxelinux.cfg/default 
```
LABEL Ubuntu-20.04.1
        kernel /images/ubuntu/vmlinuz
        MENU DEFAULT
        MENU LABEL Ubuntu-20.04.1
        append initrd=/images/ubuntu/initrd ip=dhcp autoinstall url=http://www.spatel.net/pxe_repo/ubuntu-20.04.1-live-server-amd64.iso ds=nocloud-net;s=http://www.spatel.net/pxe_ks/ubuntu-20-04-1/
```

### HTTP server prep.

Create empty meta-data file
```
[root@spatel ~]# touch /var/www/html/pxe_ks/ubuntu-20-04-1/meta-data
```
Create user-data file which contain all installation instruction in yaml style
```
[root@spatel ~]# cat /var/www/html/pxe_ks/ubuntu-20-04-1/user-data
#cloud-config
autoinstall:
  version: 1
  apt:
    geoip: true
    preserve_sources_list: false
    primary:
    - arches: [amd64, i386]
      uri: http://us.archive.ubuntu.com/ubuntu
    - arches: [default]
      uri: http://ports.ubuntu.com/ubuntu-ports
  identity:
    hostname: ubuntu-server
    password: $6$wjbhAd/wF1HKn7V.$W8IIgNtvQCu1L90XXvZahP9Lm2ILPF/juY.ExliRbpyXclEyBTK1F3u1FJdWGL0HnNPwThorz/
    username: ubuntu
  keyboard: {layout: us, toggle: null, variant: ''}
  locale: en_US
  network:
    version: 2
    ethernets:
      bootif:
        critical: true
        dhcp-identifier: mac
        dhcp4: true
        nameservers:
          addresses: [8.8.8.8]
          search: [spatel.net]
  ssh:
    allow-pw: true
    authorized-keys: []
    install-server: true

  storage:
    #### Don't create /root/swap.img file
    swap:
      size: 0
    config:
    #### prep system disks
    - type: disk
      id: disk-sda
      path: /dev/sda
      preserve: false
      wipe: superblock-recursive
      ptable: gpt
      grub_device: true
     #### disk partitions
    - type: partition
      number: 1
      id: bios_boot_part   # required by GRUB on first disk
      size: 1MB
      device: disk-sda
      flag: bios_grub
    - type: partition
      number: 2
      id: boot_part       # /boot partition
      size: 1G
      device: disk-sda
      preserve: false
    - type: partition
      number: 3
      id: lvm_part        # add remaining space to LVM
      size: -1
      device: disk-sda
      preserve: false
     #### LVM volume group and logical volumes
    - type: lvm_volgroup
      id: vg0
      name: vg0
      devices:
        - lvm_part
    - type: lvm_partition
      id: lvm_swap        # create swap partition on lvm
      volgroup: vg0
      name: swap
      size: 8G
    - type: lvm_partition
      id: lvm_root        # create / partition on lvm, all available space
      volgroup: vg0
      name: root
      size: -1
    #### Format the filesystems
    - type: format
      id: fs_boot_part
      fstype: ext4
      volume: boot_part
    - type: format
      id: fs_root
      fstype: ext4
      volume: lvm_root
    - type: format
      id: fs_swap
      fstype: swap
      volume: lvm_swap
    #### Mount the filesystems
    - type: mount
      id: mount_boot
      device: fs_boot_part
      path: /boot
    - type: mount
      id: mount_root
      device: fs_root
      path: /
    - type: mount
      id: mount_swap
      device: fs_swap
      path: none
  #### POST Install commands
  late-commands:   
    - sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /target/etc/ssh/sshd_config
    - sed -i 's|^root:.:|root:$6$wjbhAd/wF1HKn7V.$W8IIgNtvQCu1L90XXvZahP9Lm2ILPF/juY.ExliRbpyXclEyBTK1F3u1FJdWGL0HnNPwThorz/:|' /target/etc/shadow
    - rm /target/etc/netplan/00-installer-config.yaml
    - wget http://www.spate.net/pxe_scripts/00-network-config.yaml -P /target/etc/netplan/
    - curtin in-target --target=/target -- apt-get purge -y snapd
```

You're ready for pxe boot your ubunut 20.04. Enjoy!
