---
title: "Openstack cloud autoscaling with Senlin"
layout: post
date: 2020-09-04
image: /assets/images/2020-09-04-openstack-senlin-autoscaling/senlin-senlin-autoscale.png
headerImage: true
tag:
- blog
- openstack
- nova
- heat
- autoscale
- autoheal
- cloud
- aws
- senlin
category: blog
blog: true
author: Satish Patel
description: "Openstack cloud autoscaling with Senlin"

---

2018 is calling, and I've been involved with OpenStack for the better part of six years. I've seen the 'Stack mature greatly over that time, Neutron included. I'm very familiar with stock Neutron components, to include namespace-based routers, openvswitch and linuxbridge mechanism drivers, DVR, etc. My overall goal with this series is to wade through various vendor offerings and see how they improve upon those stock Neutron components. First up is one of the plugins offered by Cisco for the Cisco Aggregation Services Router (ASR) known as the **Cisco ASR1k Router Service Plugin**. 

<!--more-->
Cisco hosts this plugin, along with others, at [https://github.com/openstack/networking-cisco](https://github.com/openstack/networking-cisco).

# Features and Limitations

Cisco [documentation](http://networking-cisco.readthedocs.io/en/latest/admin/l3-asr1k.html) states the following features are provided by the L3 router service plugin:

* L3 forwarding between subnets on the tenants’ neutron L2 networks
* Support for overlapping IP address ranges between different tenants 
* NAT overload (i.e. SNAT) for connections originating on private subnets behind a tenant’s neutron router
* Static NAT (i.e Floating IP) of a private IP address on a internal neutron subnet to a public IP address on an external neutron subnet/network
* Static routes on neutron routers
* HSRP-based high availability (HA) whereby a neutron router is supported by two (or more) ASR1k routers, one actively doing L3 forwarding, the others ready to take over in case of disruptions
