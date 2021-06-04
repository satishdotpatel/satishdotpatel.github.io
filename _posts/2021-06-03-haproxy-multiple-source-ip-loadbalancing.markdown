---
title: "HAProxy Multiple Source ipaddr LoadBalancing"
layout: post
date: 2021-06-03
image: /assets/images/2021-06-03-haproxy-multiple-source-ip-loadbalancing/haproxy-logo-1.png
headerImage: true
tag:
- HAProxy
- Loadbalancer
- Ubuntu
- Cisco CML
category: blog
blog: true
author: Satish Patel
description: "HAProxy Multiple Source ipaddr LoadBalancing"

---

### What is HAProxy

HAProxy is a software load balancer commonly used to distribute TCP-based traffic to multiple backend systems. It provides not only load balancing but also has the ability to detect unresponsive backend systems and reroute incoming traffic.

### Scope 

When it comes to scale millions of connections, the first thing you need to adjust is local_port_range on load balancer but if that not help then you need to add multiple IP address on your loadbalancer to increase IP:PORT socket, In this lab I am going to demonstrate how to configure HAProxy to utilize multiple source IP addresses to talk to backend applications to increase local socket count. 

### LAB Components

I'm using Cisco Modeling lab to validate my configuration.

Software:

* Haproxy (v2.4)
* Ubuntu 18.04
* Web-1 (lighttpd)
* Web-2 (lighttpd)
* Client-1 ( curl to verify )

IPaddress:

* Frontend VIP - 192.168.255.81
* Backened source IP pool - 10.0.0.1 - 10.0.0.4
* Web-1 - 10.0.0.100
* Web-2 - 10.0.0.200

![<img>](/assets/images/2021-06-03-haproxy-multiple-source-ip-loadbalancing/haproxy-lab.png){: width="900" }{: height="250"}


### Network configuration of HAProxy

```
root@haproxy:/etc/haproxy# ifconfig 
ens2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.255.81  netmask 255.255.255.0  broadcast 192.168.255.255

ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.1  netmask 255.0.0.0  broadcast 10.255.255.255

ens4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.2  netmask 255.0.0.0  broadcast 10.255.255.255

ens5: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.3  netmask 255.0.0.0  broadcast 10.255.255.255

ens6: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.4  netmask 255.0.0.0  broadcast 10.255.255.255
```

### HAProxy configuration

haproxy.cfg

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        maxconn 100000  # higher is better
        nbproc  2       # number of workers ( keep same as cpu cores)
        cpu-map 1 0     # map workers with cores
        cpu-map 2 1     # 

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        maxconn 100000  # higher is better

frontend web-front-1
  bind 192.168.255.81:80
  option httplog
  option forwardfor except 127.0.0.0/8
  mode http
  default_backend web-backend-1

backend web-backend-1
  mode http
  balance roundrobin
  option httplog

  server web101 10.0.0.100:80 source 10.0.0.1
  server web102 10.0.0.100:80 source 10.0.0.2
  server web103 10.0.0.100:80 source 10.0.0.3
  server web104 10.0.0.100:80 source 10.0.0.4
```

#### Validation

we have 4 source ips so lets run curl 4 time

```
root@client-1:~# for qw in `seq 1 4`; do curl 192.168.255.81; done
web-1
web-1
web-1
web-1
```

check web-1 logs, as you can see all 4 curl request use 4 different source ip to make connection with web-1

```
root@web-1:/var/www/html# tail -f /var/log/lighttpd/access.log
10.0.0.2 192.168.255.81 - [04/Jun/2021:02:46:24 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.58.0"
10.0.0.3 192.168.255.81 - [04/Jun/2021:02:46:24 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.58.0"
10.0.0.4 192.168.255.81 - [04/Jun/2021:02:46:24 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.58.0"
10.0.0.1 192.168.255.81 - [04/Jun/2021:02:46:37 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.58.0"

```

### Let's add web-2 server

In following way it will do better loadbalancing between both server, because haproxy send request to pool member in sequence. 

```
  # web-1/2 use source 10.0.0.1
  server web101 10.0.0.100:80 source 10.0.0.1
  server web201 10.0.0.200:80 source 10.0.0.1
  
  # web-1/2 use source 10.0.0.2
  server web102 10.0.0.100:80 source 10.0.0.2
  server web202 10.0.0.200:80 source 10.0.0.2

  # web-1/2 use source 10.0.0.3
  server web103 10.0.0.100:80 source 10.0.0.3
  server web203 10.0.0.200:80 source 10.0.0.3
  
  # web-1/2 use source 10.0.0.4
  server web104 10.0.0.100:80 source 10.0.0.4
  server web204 10.0.0.200:80 source 10.0.0.4
```

quick validation

```
root@client-1:~# for qw in `seq 1 8`; do curl 192.168.255.81; sleep 1; done
web-1
web-2
web-1
web-2
web-1
web-2
web-1
web-2
```

