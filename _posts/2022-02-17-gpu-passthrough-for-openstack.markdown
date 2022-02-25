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

#### GPU Burn Test

Install NVDIA driver for Ubuntu 20.04

```
root@gpu1-vm:~# apt-get update
root@gpu1-vm:~# apt-get install ubuntu-drivers-common
root@gpu1-vm:~# apt-get install nvidia-driver-470
root@gpu1-vm:~# apt-get install nvidia-cuda-dev
root@gpu1-vm:~# apt install nvidia-cuda-toolkit
```

Download GPU Burn program

```
root@gpu1-vm:~# git clone https://github.com/wilicc/gpu-burn
```

Edit Makefile and change first line to following

```
CUDAPATH ?= /usr
```

Run Make

```
root@gpu1-vm:~/gpu-burn# make 
```

Run program for 60 second 

```
root@gpu1-vm:~/gpu-burn# ./gpu_burn 60
GPU 0: Tesla V100S-PCIE-32GB (UUID: GPU-dd342e0f-155c-18d7-f398-b415faf0a350)
Initialized device 0 with 32510 MB of memory (32139 MB available, using 28925 MB of it), using FLOATS
Results are 16777216 bytes each, thus performing 1805 iterations
11.7%  proc'd: 3610 (12462 Gflop/s)   errors: 0   temps: 50 C
	Summary at:   Fri Feb 25 15:43:19 UTC 2022

23.3%  proc'd: 9025 (12391 Gflop/s)   errors: 0   temps: 53 C
	Summary at:   Fri Feb 25 15:43:26 UTC 2022

35.0%  proc'd: 12635 (12386 Gflop/s)   errors: 0   temps: 56 C
	Summary at:   Fri Feb 25 15:43:33 UTC 2022

48.3%  proc'd: 19855 (12277 Gflop/s)   errors: 0   temps: 60 C
	Summary at:   Fri Feb 25 15:43:41 UTC 2022

60.0%  proc'd: 23465 (12263 Gflop/s)   errors: 0   temps: 61 C
	Summary at:   Fri Feb 25 15:43:48 UTC 2022

75.0%  proc'd: 30685 (12264 Gflop/s)   errors: 0   temps: 63 C
	Summary at:   Fri Feb 25 15:43:57 UTC 2022

86.7%  proc'd: 36100 (12253 Gflop/s)   errors: 0   temps: 64 C
	Summary at:   Fri Feb 25 15:44:04 UTC 2022

100.0%  proc'd: 41515 (12259 Gflop/s)   errors: 0   temps: 65 C
	Summary at:   Fri Feb 25 15:44:12 UTC 2022

100.0%  proc'd: 41515 (12259 Gflop/s)   errors: 0   temps: 65 C
Killing processes.. Freed memory for dev 0
Uninitted cublas
done

Tested 1 GPUs:
	GPU 0: OK
```

On other terminal you can see GPU stats in realtime with following command

```
root@gpu-vm1:~/gpu-burn# nvidia-smi
Fri Feb 25 16:56:49 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.103.01   Driver Version: 470.103.01   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla V100S-PCI...  Off  | 00000000:00:05.0 Off |                    0 |
| N/A   48C    P0   250W / 250W |  29283MiB / 32510MiB |    100%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     38716      C   ./gpu_burn                      29279MiB |
+-----------------------------------------------------------------------------+
```

Enjoy!!! 

