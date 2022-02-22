---
title: "High Performance computing (HPC) on Openstack"
layout: post
date: 2022-02-20
image: /assets/images/2022-02-20-hpc-on-openstack/HPC-network.png
headerImage: true
tag:
- HPC
- Infiniband
- Openstack
- Kolla-Ansible
category: blog
blog: true
author: Satish Patel
description: "High Performance computing (HPC) on Openstack"

---

Recently i am working on deployment on High-Performance Computing (HPC) on Openstack. In this blog, I am going cover deployment of HPC network using Infiniband network for MPI jobs for high throughput workloads. 

### Prerequisit 

We need working openstack deployment. In my case i am running kolla-ansible. 

* controller node 
* compute1 node
* compute2 node

#### Network Switches

* 10Gb/s Ethernet switch
* 200Gb/s HDR Infiniband Switch (QM8700)

![<img>](/assets/images/2022-02-20-hpc-on-openstack/openstack-hpc-network.png){: width="400" }

### Configure Mellanox Infiniband Switch

In my deployment i have used mellanox breakout cable to split switch ethernet interfaces. 200Gb/s single interface i have split into two 100Gb/s virtual interface. 

```
interface ib 1/2 module-type qsfp-split-2 force
```

You can see two virtual interface in following output

```
switch [standalone: master] # show interfaces ib 1/2/1
IB1/2/1 state:
        Logical port state          : Active
        Physical port state         : LinkUp
        Current line rate           : 100.0 Gbps
        Supported speeds            : sdr, qdr, fdr, edr, hdr
        Speed                       : hdr
        Supported widths            : 1X, 2X
        Width                       : 2X
        Max supported MTUs          : 4096
        MTU                         : 4096
        VL capabilities             : VL0 - VL7
        Operational VLs             : VL0 - VL3
        Description                 :
        IB Subnet                   : infiniband-default

switch [standalone: master] # show interfaces ib 1/2/2
IB1/2/2 state:
        Logical port state          : Active
        Physical port state         : LinkUp
        Current line rate           : 100.0 Gbps
        Supported speeds            : sdr, qdr, fdr, edr, hdr
        Speed                       : hdr
        Supported widths            : 1X, 2X
        Width                       : 2X
        Max supported MTUs          : 4096
        MTU                         : 4096
        VL capabilities             : VL0 - VL7
        Operational VLs             : VL0 - VL3
        Description                 :
        IB Subnet                   : infiniband-default
```

Enable Subnet Manager on Switch

```
switch [standalone: master] # ib sm virt enable
switch [standalone: master] # configuration write
switch [standalone: master] # no ib sm
switch [standalone: master] # ib sm
```

### Configure Compute nodes

#### Enable IOMMU 

By default IOMMU is disabled, To enable add following in /etc/default/grub. I have enabled HugePages for Memory performance. 

```
GRUB_CMDLINE_LINUX=" ... intel_iommu=on iommu=pt pci=assign-busses pci=realloc hugepagesz=2M hugepages=90000 transparent_hugepage=never"
```

#### Install Mellanox Driver

Install the latest MLNX_OFED driver on the compute host. In my case its MLNX_OFED_LINUX-5.4-3.1.0.0-rhel8.5-x86_64.tar.gz. 

```
./mlnxofedinstall
```

#### Enable SR-IOV on the Firmware

Run Mellanox Firmware Tooks (MFT)

```
$ mst start
$ mst status
MST modules:
------------
    MST PCI module is not loaded
    MST PCI configuration module loaded

MST devices:
------------
/dev/mst/mt4123_pciconf0         - PCI configuration cycles access.
                                   domain:bus:dev.fn=0000:af:00.0 addr.reg=88 data.reg=92 cr_bar.gw_offset=-1
                                   Chip revision is: 00
```

Query driver for SRIOV 

```
$ mlxconfig -d /dev/mst/mt4123_pciconf0 q | grep SRIOV_EN
         SRIOV_EN                            False(0)
```

Enable SRIOV VFS. In my case it will enable 4 sriov virtual function

```
$ mlxconfig -d /dev/mst/mt4123_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=4
```

Reboot the server or just reboot the adapter firmware to apply changes.

```
$ mlxfwreset --device /dev/mst/mt4113_pciconf0 reset
```

Note: At this point, the VFs are not seen via the lspci output. You need to enable it in MLNX_OFED Driver which is next step.

```
$ echo 4 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
```

Then we have to configure each VF with following way. 

```
$ echo Follow > /sys/class/infiniband/mlx5_0/device/sriov/0/policy
$ echo 11:22:33:44:77:66:77:90 > /sys/class/infiniband/mlx5_0/device/sriov/0/node
$ echo 11:22:33:44:77:66:77:91 > /sys/class/infiniband/mlx5_0/device/sriov/0/port
$ echo 0000:af:00.1 > /sys/bus/pci/drivers/mlx5_core/unbind
$ echo 0000:af:00.1 > /sys/bus/pci/drivers/mlx5_core/bind
```

We need to assign a GUID to both nodes and port, and we need to rebind the driver for that newly assigned GUID to take effect. You need to do this on each vf and every reboot of server which is not practical. I found script created by this author which will save your life https://gist.github.com/koallen/32709a244d77a2c0f8e17ed79a4092ed. 

```
$ cd /usr/local/sbin/
$ wget https://gist.githubusercontent.com/koallen/32709a244d77a2c0f8e17ed79a4092ed/raw/88f314f90cb3f4d03deb9088453d9f94f90add90/mlnx_sriov.sh
```

Add script in /etc/rc.local file.

```
$ cat /etc/rc.local
echo 4 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs
/usr/local/sbin/mlnx_sriov.sh mlx5_0 4
```

Reboot server. 

#### Verify Infiniband SRIOV VF Link status 

Run ibstat and make sure State is Active. If its polling that means interface can't see Subnet Manager. 

```
$ ibstat
CA 'mlx5_0'
	CA type: MT4123
	Number of ports: 1
	Firmware version: 20.28.1002
	Hardware version: 0
	Node GUID: 0x043f720300cfef92
	System image GUID: 0x043f720300cfef92
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 100
		Base lid: 28
		LMC: 0
		SM lid: 1
		Capability mask: 0x2651e848
		Port GUID: 0x043f720300cfef92
		Link layer: InfiniBand
CA 'mlx5_1'
	CA type: MT4124
	Number of ports: 1
	Firmware version: 20.28.1002
	Hardware version: 0
	Node GUID: 0x0022330000014890
	System image GUID: 0x043f720300cfef92
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 100
		Base lid: 69
		LMC: 0
		SM lid: 1
		Capability mask: 0x2651ec48
		Port GUID: 0x0022330000014891
		Link layer: InfiniBand

``` 

### Configure Openstack Nova

On compute nodes nova.conf (nova-compute)

```
[PCI]
passthrough_whitelist = { "vendor_id": "10de", "product_id": "1df6" }
```

On controller nova.conf (nova-api)

```
[PCI]
alias = { "vendor_id":"15b3", "product_id":"101c", "device_type":"type-VF", "name":"mlx5-sriov-ib" }
```

On controller nova.conf (nova-scheduler)

```
[filter_scheduler]
enabled_filters=PciPassthroughFilter,NUMATopologyFilter
```

### Create flavor 

Set pci_passthrough property. I am using HugePage so i have configure mem_page_size with flavor and use cpu_policy dedicated for CPU pinning for better performance. 

```
$ openstack flavor create --vcpus 8 --ram 16384 --disk 100 --property hw:mem_page_size='large' --property hw:cpu_policy='dedicated' --property "pci_passthrough:alias"="mlx5-sriov-ib" ib.small
```

### Create virtual machine 

Use above flavor to create virtual machine to test Infiniband SRIOV passthrough. 

```
$ openstack server create --flavor ib.small --image ubuntu_20_04 --nic net-id=private1 ib-vm1
```

Verify instance (Bravo!!!) 

```
root@ib-vm1:~# lspci | grep -i mell
00:05.0 Infiniband controller: Mellanox Technologies MT28908 Family [ConnectX-6 Virtual Function]
```

### Install MLNX_OFED driver on VM

Install the latest MLNX_OFED driver on the VM. Driver is required to bring up sriov interface. 

After successfully installation of OFED driver you can see status will come Active.

```
root@ib-vm1:~# ibstat
CA 'mlx5_0'
	CA type: MT4124
	Number of ports: 1
	Firmware version: 20.28.1002
	Hardware version: 0
	Node GUID: 0x0022330003014590
	System image GUID: 0x043f720300cfefae
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 100
		Base lid: 38
		LMC: 0
		SM lid: 1
		Capability mask: 0x2651ec48
		Port GUID: 0x0022330003014591
		Link layer: InfiniBand
```

Now you are ready to run MPI workload :) In next blog i will show you how to run MPI jobs on Infiniband using RDMA between two virtual machine. Enjoy!! 
