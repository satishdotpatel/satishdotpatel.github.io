---
title: "GPU Passthrough for Openstack"
layout: post
date: 2022-02-17
image: /assets/images/2022-02-17-gpu-passthrough-for-openstack/tesla-gpu.png
headerImage: true
tag:
- GPU
- Openstack
- Kolla-Ansible
category: blog
blog: true
author: Satish Patel
description: "GPU Passthrough for Openstack"

---

This is to demonstrate how to configure GPU passthrough configuration with Openstack. I am running kolla-ansible based openstack running on Wallaby and for the GPU I have two Tesla V100S PCIe 32GB. 

#### Verify host environment 

Check VT is enabled

```

[root@compute1 ~]# dmesg | grep -e "Directed I/O"
[    1.819954] DMAR: Intel(R) Virtualization Technology for Directed I/O

```

Verify GPU card 

```

[root@compute1 ~]# lspci | grep -i nv
5e:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] (rev a1)
d8:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] (rev a1)

```
#### Enable IOMMU

IOMMU is disabled by default. You need to enable boot time in the /etc/default/grub configuration file, Also I have blacklist nVidia drive to load on the host machine otherwise we can't bind it with vfio. 

```

GRUB_CMDLINE_LINUX="rd.driver.blacklist=nouveau,nvidia,nvidia_drm nouveau.modeset=0 intel_iommu=on iommu=pt"

```

Compile grub file and reboot host

```

[root@compute1 ~]# grub2-mkconfig -o /boot/grub2/grub.cfg

```

#### Configure VFIO 

VFIO will allow userspace programs to access PCI devices. 

```

[root@compute1 ~]# cat /etc/modules-load.d/vfio-pci.conf
vfio-pci

```

Find vendor and product ID 

```
[root@GPUN05 ~]# lspci -nn | grep -i nvidia
5e:00.0 3D controller [0302]: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] [10de:1df6] (rev a1)
d8:00.0 3D controller [0302]: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] [10de:1df6] (rev a1)

```

Pass above vendor/product ID (10de:1df6) to driver in following option 

```

[root@compute1 ~]# cat /etc/modprobe.d/gpu-vfio.conf
options vfio-pci ids=10de:1df6

```

#### Nova compute add passthrough_whitelist 

In compute node nova.conf add the following under the PCI section. You can find vendor_id and product_id in lspci -nn output. 

```

[PCI]
passthrough_whitelist = { "vendor_id": "10de", "product_id": "1df6" }

```

#### Nova API add alias 

In openstack controller node add following in nova.conf file.

```

[PCI]
alias: { "vendor_id":"10de", "product_id":"1df6", "device_type":"type-PCI", "name":"tesla-v100" }

```

#### Nova Scheduler add PCI Passthrough filter

In openstack controller node add following in nova scheduler conf file

```

[filter_scheduler]
enabled_filters = PciPassthroughFilter

```


#### Create flavor 

Create flavor with GPU properties. Use the alias which you specified in nova-api.conf file. "pci_passthrough:alias"="tesla-v100:1" will pass a single GPU card to your VirtualMachine. If you want two GPU in your VM use "pci_passthrough:alias"="tesla-v100:2"

```

$ openstack flavor create --vcpus 16 --ram 32768 --disk 160 --property "pci_passthrough:alias"="tesla-v100:1"  gpu1.medium


```

#### Create VM 

```

$ openstack server create --flavor gpu1.medium --image ubuntu20_04 --nic net-id=private1 gpu1-vm

```

#### Verify GPU inside VM


```

root@gpu1-vm:~# lspci | grep -i nvidia
00:05.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] (rev a1)

```

Let's pass two GPU to VirtualMachine, For that we need to create new flavor with property tesla-v100:2 

```

$ openstack flavor create --vcpus 16 --ram 32768 --disk 160 --property "pci_passthrough:alias"="tesla-v100:1"  gpu2.medium

```

Now you can see following output after creating vm using above flavor 

```
root@gpu2-vm:~# lspci | grep -i nvidia
00:05.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] (rev a1)
00:06.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100S PCIe 32GB] (rev a1)
```

Enjoy!!! 

