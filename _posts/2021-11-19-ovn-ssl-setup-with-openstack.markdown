---
title: "OVN SSL setup for Openstack"
layout: post
date: 2021-11-19
image: /assets/images/2021-11-19-ovn-ssl-setup-with-openstack/lock.png
headerImage: true
tag:
- Openstack
- OVN
- SSL
category: blog
blog: true
author: Satish Patel
description: "OVN SSL setup for Openstack"

---

This document explains the way one could use SSL for connectivity between OVN components. For demostration i have deploy openstack using openstack-ansible and i have 1 controller (Infra node) and 2 compute nodes in my lab setup.

* Infra node - (openstack controller where ovn-central/ovn-north service is running)
* compute-1 - (where ovn-controller service is running)
* compute-2 - (where ovn-controller service is running)

### Create a certificate authority.

My CA authority is my Infra node. Following command will create certificate authority. 

```
$ ovs-pki init --force
```

Above command will create following directories structure. We are interested in switchca directory. 

```
$ ls -l /var/lib/openvswitch/pki/
total 8
drwxr-xr-x 6 root root 4096 Nov 19 14:48 controllerca
drwxr-xr-x 6 root root 4096 Nov 19 20:59 switchca
```

### Generate signed certificates for OVN components running on the master.

#### Generate signed certificates for OVN NB Database

On Infra node, run the following commands. For simplicity i have created /etc/openvswitch directory on Infra node to track all certificate in that directory. 

```
$ mkdir /etc/openvswitch
$ cd /etc/openvswitch
$ ovs-pki req ovnnb
$ ovs-pki -b sign ovnnb
```

The above command will generate following 3 files in /etc/openvswitch directory. Later we will copy them to ovn-northd container where ovs-central service running. 

```
$ ls /etc/openvswitch/
ovnnb-cert.pem ovnnb-privkey.pem ovnnb-req.pem
```

#### Generate signed certificates for OVN SB Database

On Infra node. Do the similar things to generate SB Database certificate. 

```
$ cd /etc/openvswitch
$ ovs-pki req ovnsb
$ ovs-pki -b sign ovnsb
```

#### Generate signed certificates for ovn-northd daemon

If you are running ovn-northd on the same host as the OVN NB and SB database servers, then there is no need to secure the communication between ovn-northd and OVN NB/SB daemons. ovn-northd will communicate using UNIX path. But in my case i not using Unix socket so i need to setup SSL communication. 

On Infra node.

```
$ cd /etc/openvswitch
$ ovs-pki req ovnnorthd
$ ovs-pki -b sign ovnnorthd
```

#### Generate certificates for the compute nodes (ovn-controller)

On Infra node. 

```
$ cd /etc/openvswitch
$ ovs-pki req ovncontroller
$ ovs-pki -b sign ovncontroller switch
```

#### Copy generated certs/key/CA files to ovn-central node

Let's copy ovnnb/ovnsb/ovn-northd certs file to ovn-northd container from Infra node where we generated all certs.

```
$ scp /etc/openvswitch/ovnnb-* ovn-lab-infra-1-neutron-ovn-northd-container-cb55f5ef:/etc/openvswitch/
$ scp /etc/openvswitch/ovnsb-* ovn-lab-infra-1-neutron-ovn-northd-container-cb55f5ef:/etc/openvswitch/
$ scp /etc/openvswitch/ovnnorthd-* ovn-lab-infra-1-neutron-ovn-northd-container-cb55f5ef:/etc/openvswitch/
```

Let's copy CA cert file 

```
$ scp /var/lib/openvswitch/pki/switchca/cacert.pem ovn-lab-infra-1-neutron-ovn-northd-container-cb55f5ef:/etc/openvswitch/
```

#### Configure SSL for ovn NB/SB 

On ovn-northd container run the following commands to ask NB ovsdb-server to use these certificates and also to open up SSL ports via which the database can be accessed.

```
$ ovn-nbctl set-ssl /etc/openvswitch/ovnnb-privkey.pem \
     /etc/openvswitch/ovnnb-cert.pem  /etc/openvswitch/cacert.pem
$ ovn-nbctl set-connection pssl:6641
```

Same for SB Database

```
$ ovn-sbctl set-ssl /etc/openvswitch/ovnsb-privkey.pem \
    /etc/openvswitch/ovnsb-cert.pem  /etc/openvswitch/cacert.pem

$ ovn-sbctl set-connection pssl:6642
```

Verify 

```
$ ovn-nbctl get-ssl
Private key: /etc/openvswitch/ovnnb-privkey.pem
Certificate: /etc/openvswitch/ovnnb-cert.pem
CA Certificate: /etc/openvswitch/cacert.pem
Bootstrap: false
```

```
$ ovn-nbctl get-connection
pssl:6641
```

Configure SSL for ovn-northd daemon. 

```
$ cat /etc/default/ovn-central
# OVN cluster parameters
OVN_CTL_OPTS=" \
  --db-nb-create-insecure-remote=no \
  --db-sb-create-insecure-remote=no \
  --db-nb-addr=10.62.7.252 \
  --db-sb-addr=10.62.7.252 \
  --db-nb-cluster-local-addr=10.62.7.252 \
  --db-sb-cluster-local-addr=10.62.7.252 \
  --ovn-northd-nb-db=ssl:10.62.7.252:6641 \
  --ovn-northd-sb-db=ssl:10.62.7.252:6642 \
  --ovn-northd-ssl-key=/etc/openvswitch/ovnnorthd-privkey.pem \
  --ovn-northd-ssl-cert=/etc/openvswitch/ovnnorthd-cert.pem \
  --ovn-northd-ssl-ca-cert=/etc/openvswitch/cacert.pem \
"
```

Restart ovn-central.service

```
$ systemctl restart ovn-central.service
```

Check following logs files for errors.

```
$ tail -f /var/log/ovn/ovn-northd.log
$ tail -f /var/log/ovn/ovsdb-server-nb.log
$ tail -f /var/log/ovn/ovsdb-server-sb.log
```

#### Configure SSL for ovn-controller on compute nodes

Copy following files from Infra node to compute nodes. 

```
$ scp /etc/openvswitch/ovncontroller-* compute01:/etc/openvswitch/
$ scp /etc/openvswitch/ovncontroller-* compute02:/etc/openvswitch/
```

Copy CA cert file to compute nodes

```
$ scp /var/lib/openvswitch/pki/switchca/cacert.pem compute01:/etc/openvswitch/
$ scp /var/lib/openvswitch/pki/switchca/cacert.pem compute02:/etc/openvswitch/
```

Tell ovs to use ssl to connect SB Database. 

```
$ ovs-vsctl set Open_vSwitch . external-ids:ovn-remote=ssl:$OVN_CENTRAL_IP:6642
```

Add following option in /etc/default/ovn-host file. 

```
$ cat /etc/default/ovn-host
OVN_CTL_OPTS="--ovn-controller-ssl-key=/etc/openvswitch/ovncontroller-privkey.pem  --ovn-controller-ssl-cert=/etc/openvswitch/ovncontroller-cert.pem --ovn-controller-ssl-ca-cert=/etc/openvswitch/cacert.pem"
```

Restart ovn-controller

```
$ systemctl restart ovn-controller
```

Verify, If all good then you will see all your ports and bridge in following command output. Also check logs here for any errors /var/log/ovn/ovn-controller.log

```
$ ovs-vsctl show
```

Enjoy!! 







