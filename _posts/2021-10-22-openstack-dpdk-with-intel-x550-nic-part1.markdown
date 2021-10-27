---
title: "Openstack OVS-DPDK with Intel X550T NIC - Part1"
layout: post
date: 2021-10-22
image: /assets/images/2021-10-22-openstack-dpdk-with-intel-x550t2-nic/fastlane.png
headerImage: true
tag:
- Openstack
- DPDK
- Openstack-Ansible
- X550T
- OpenvSwitch
- Ubuntu
category: blog
blog: true
author: Satish Patel
description: "Openstack OVS-DPDK with Intel X550T NIC - Part1"

---

I am going to blog about how to configure OVS-DPDK on Dell server PowerEdge 440 with Intel X550T NIC card. I am using Openstack-Ansible to setup my openstack cloud and configure DPDK environment. 

This is what i have on Server, two physical NIC card. NetXtreme BCM5720 for openstack managment traffic and X550T for VM Data traffic. 

```
# lspci | grep -i eth
04:00.0 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5720 2-port Gigabit Ethernet PCIe
04:00.1 Ethernet controller: Broadcom Inc. and subsidiaries NetXtreme BCM5720 2-port Gigabit Ethernet PCIe
3b:00.0 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
3b:00.1 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
af:00.0 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
af:00.1 Ethernet controller: Intel Corporation Ethernet Controller 10G X550T (rev 01)
```

I've changed interface name for a simplicity. int - Internal and ext - External (ext is my Intel X550)

```
$ cat /etc/udev/rules.d/70-persistent-net.rules
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="e4:3d:1a:0c:06:40", NAME="int0"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="e4:3d:1a:0b:a3:40", NAME="int1"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="e4:3d:1a:0c:06:41", NAME="ext0"
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="e4:3d:1a:0b:a3:41", NAME="ext1"
```

Lets check firmware version of Intel NIC

```
$ ethtool -i ext0
driver: ixgbe
version: 5.1.0-k
firmware-version: 0x80000f32, 19.5.12
expansion-rom-version:
bus-info: 0000:3b:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes
```

Setup grub to enable IOMMU and HugePages for DPDK (I have 96GB memory and i have setup 64GB HugePages). After setting up grub run update-grub2 and reboot server

```
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=0 rd.auto intel_iommu=on iommu=pt default_hugepagesz=1GB hugepagesz=1G hugepages=64 transparent_hugepage=never"
GRUB_CMDLINE_LINUX="rd.auto"
```

After successful reboot you can verify using following command.

```
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-5.4.0-86-generic root=/dev/mapper/vg.IDVhDHCBJA-root ro rd.auto consoleblank=0 rd.auto intel_iommu=on iommu=pt default_hugepagesz=1GB hugepagesz=1G hugepages=64 transparent_hugepage=never
```

netplan config

```
$ cat /etc/netplan/01-netcfg.yaml
---
network:
  version: 2
  renderer: networkd
  ethernets:
    int0:
      match:
        macaddress: b4:96:91:be:3c:58
      optional: true
    int1:
      match:
        macaddress: b4:96:91:bd:68:e4
      optional: true
    ext0:
      match:
        macaddress: b4:96:91:be:3c:5a
      optional: true
    ext1:
      match:
        macaddress: b4:96:91:bd:68:e6
      optional: true


  bonds:
    aggi:
      interfaces:
        - int0
        - int1
      macaddress: b4:96:91:be:3c:58
      mtu: 1500
      parameters:
        mode: 802.3ad
        mii-monitor-interval: 100
        down-delay: 200
        up-delay: 200
        lacp-rate: slow
        transmit-hash-policy: layer3+4

  # br-mgmt
  vlans:
    aggi.2001:
      id: 2001
      link: aggi
      dhcp4: no
      dhcp6: no
      mtu: 1500

  bridges:
    br-mgmt:
      interfaces: [ aggi.2001 ]
      addresses: [ 10.72.0.31/22 ]
      gateway4: 10.72.0.1
      nameservers:
        addresses:
          - 8.8.8.8
      mtu: 1500

  # br-vxlan
  vlans:
    aggi.2002:
      id: 2002
      link: aggi
      dhcp4: no
      dhcp6: no
      mtu: 1500

  bridges:
    br-vxlan:
      interfaces: [ aggi.2002 ]
      addresses: [ 10.72.4.31/22 ]
      mtu: 1500
```

Now setup openstack-ansible to cook this node as an DPDK compute node. I do have running openstack-ansible environment. if you curious you can check it out my previous post.

This is my host override to deploy dpdk on ostack-ams-comp-dpdk-5 compute node. (You can read openstack-ansible doc for details parameters)

```
$ cat /etc/openstack_deploy/host_vars/ostack-ams-comp-dpdk-5.yml
---
openstack_host_specific_kernel_modules:
  - name: "openvswitch"
    pattern: "CONFIG_OPENVSWITCH="
    group: "network_hosts"

# Neutron specific config
neutron_plugin_type: ml2.ovs

#
neutron_provider_networks:
  network_types: "vlan"
  network_vlan_ranges: "vlan:2000:2010"
  network_mappings: "vlan:br-vlan"


# Enable DPDK support
ovs_dpdk_support: True

# DPDK NIC X550T PCI bus, PMD cpu core & socket memory parameters
ovs_dpdk_pci_addresses:
  - 0000:3b:00.1
  - 0000:af:00.1
#ovs_dpdk_lcore_mask: 101000101
ovs_dpdk_pmd_cpu_mask: 0x0c
ovs_dpdk_socket_mem: "1024,1024"
```

Run openstack-ansible playbook to deploy nova and neutron using following command

```
$ openstack-ansible setup-host.yml os-nova-install.yml os-neutron-install.yml --limit ostack-ams-comp-dpdk-5
```

Lets verify DPDK NIC binding, as you can see in following output 0000:3b:00.1 & 0000:af:00.1 are now DPDK nic. 

```
$ dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:3b:00.1 'Ethernet Controller 10G X550T 1563' drv=vfio-pci unused=ixgbe
0000:af:00.1 'Ethernet Controller 10G X550T 1563' drv=vfio-pci unused=ixgbe

Network devices using kernel driver
===================================
0000:04:00.0 'NetXtreme BCM5720 2-port Gigabit Ethernet PCIe 165f' if=eno1 drv=tg3 unused=vfio-pci
0000:04:00.1 'NetXtreme BCM5720 2-port Gigabit Ethernet PCIe 165f' if=eno2 drv=tg3 unused=vfio-pci
0000:3b:00.0 'Ethernet Controller 10G X550T 1563' if=ens2f0 drv=ixgbe unused=vfio-pci
0000:af:00.0 'Ethernet Controller 10G X550T 1563' if=ens3f0 drv=ixgbe unused=vfio-pci
```

I have two DPDK nic so i am going to configure LACP bonding and attaching it to br-vlan which is my provider bridge.

```
$ ovs-vsctl add-bond br-vlan dpdkbond dpdk0 dpdk1 bond_mode=balance-tcp lacp=active -- set Interface dpdk0 type=dpdk options:dpdk-devargs=0000:3b:00.1 -- set Interface dpdk1 type=dpdk options:dpdk-devargs=0000:af:00.1
```

Verify bonding 

```
$ ovs-appctl lacp/show dpdkbond
---- dpdkbond ----
  status: active negotiated
  sys_id: b4:96:91:be:3c:5a
  sys_priority: 65534
  aggregation key: 4
  lacp_time: slow

slave: dpdk0: current attached
  port_id: 5
  port_priority: 65535
  may_enable: false

  actor sys_id: b4:96:91:be:3c:5a
  actor sys_priority: 65534
  actor port_id: 5
  actor port_priority: 65535
  actor key: 4
  actor state: activity aggregation synchronized collecting distributing

  partner sys_id: 01:e0:52:00:00:01
  partner sys_priority: 32768
  partner port_id: 520
  partner port_priority: 32768
  partner key: 8
  partner state: activity aggregation defaulted

slave: dpdk1: defaulted detached
  port_id: 4
  port_priority: 65535
  may_enable: false

  actor sys_id: b4:96:91:be:3c:5a
  actor sys_priority: 65534
  actor port_id: 4
  actor port_priority: 65535
  actor key: 4
  actor state: activity aggregation defaulted

  partner sys_id: 00:00:00:00:00:00
  partner sys_priority: 0
  partner port_id: 0
  partner port_priority: 0
  partner key: 0
  partner state:
```

validate bonding 

```
$ ovs-appctl bond/show dpdkbond
---- dpdkbond ----
bond_mode: balance-tcp
bond may use recirculation: yes, Recirc-ID : 2
bond-hash-basis: 0
updelay: 0 ms
downdelay: 0 ms
next rebalance: 3054 ms
lacp_status: negotiated
lacp_fallback_ab: false
active slave mac: b4:96:91:be:3c:5a(dpdk0)

slave dpdk0: enabled
  active slave
  may_enable: true

slave dpdk1: disabled
  may_enable: false
```

Shutdown bond0 interface to check failover 

```
$ ovs-ofctl mod-port br-vlan dpdk0 down
```

You can see dpdk0 is disabled and dpdk1 is active slave.  

```
$ ovs-appctl bond/show dpdkbond
---- dpdkbond ----
bond_mode: balance-tcp
bond may use recirculation: yes, Recirc-ID : 1
bond-hash-basis: 0
updelay: 0 ms
downdelay: 0 ms
next rebalance: 4772 ms
lacp_status: negotiated
lacp_fallback_ab: false
active slave mac: b4:96:91:bd:68:e6(dpdk1)

slave dpdk0: disabled
  may_enable: false

slave dpdk1: enabled
  active slave
  may_enable: true
```

Verify DPDK mapping with OVS br-vlan bridge

```
$ ovs-vsctl show
fbe6aa1c-f313-4f8d-b5cb-781885cb286a
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-vlan
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: netdev
        Port dpdkbond
            Interface dpdk1
                type: dpdk
                options: {dpdk-devargs="0000:af:00.1"}
            Interface dpdk0
                type: dpdk
                options: {dpdk-devargs="0000:3b:00.1"}
        Port phy-br-vlan
            Interface phy-br-vlan
                type: patch
                options: {peer=int-br-vlan}
        Port br-vlan
            Interface br-vlan
                type: internal
```

Verify PMD thread mapping with CPU cores

```
ovs-appctl dpif-netdev/pmd-rxq-show
pmd thread numa_id 1 core_id 1:
  isolated : false
  port: dpdk1             queue-id:  0 (enabled)   pmd usage:  0 %
pmd thread numa_id 0 core_id 4:
  isolated : false
  port: dpdk0             queue-id:  0 (enabled)   pmd usage:  0 %
```

Enjoy! In next section i will show you how to do a performance tunning for DPDK to get optimal performance.



















