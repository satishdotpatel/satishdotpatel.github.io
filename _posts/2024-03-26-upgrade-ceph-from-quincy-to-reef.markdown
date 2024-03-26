---
title: "Upgrade Ceph from Quincy to Reef"
layout: post
date: 2024-03-26
image: /assets/images/2024-03-26-upgrade-ceph-from-quincy-to-reef/ceph-reef.png
headerImage: true
tag:
- Openstack
- Ceph
- Quincy
- Reef
- Storage
category: blog
blog: true
author: Satish Patel
description: "Upgrade Ceph from Quincy to Reef"

---

In this blog post, I'm going to upgrade production ceph storage from Quincy to [Reef](https://ceph.io/en/news/blog/2023/v18-2-0-reef-released/) release using cephadm. Please read [Upgrade Guide](https://docs.ceph.com/en/latest/cephadm/upgrade/) before proceed.  

Cephadm use following order to upgrade all the components: 

mgr -> mon -> crash -> osd -> mds -> rgw -> rbd-mirror -> cephfs-mirror -> iscsi -> nfs

### Prerequisite  

* Ceph cluster running Quincy release 

### Before upgrade

Check version

```
$ ceph version
ceph version 17.2.6 (d7ff0d10654d2280e08f1ab989c7cdf3064446a5) quincy (stable)
```

List of nodes (3x ceph controller and 3x OSD nodes)

```
# ceph orch host ls
HOST       ADDR            LABELS  STATUS
ceph-osd1  192.168.24.101
ceph-osd2  192.168.24.102
ceph-osd3  192.168.24.103
os-ctrl1   192.168.24.110  _admin
os-ctrl2   192.168.24.111  _admin
os-ctrl3   192.168.24.112  _admin
6 hosts in cluster
```
List of running daemons.

```
ceph orch ps

ceph-exporter.ceph-osd3  ceph-osd3               running (6M)     89s ago   7M    50.1M        -  17.2.6   84d64ce389a6  fc7d9b70814a
ceph-exporter.os-ctrl1   os-ctrl1                running (23h)    87s ago   7M    16.2M        -  17.2.6   84d64ce389a6  7f8c63c4f92f
ceph-exporter.os-ctrl2   os-ctrl2                running (23h)    88s ago   7M    16.8M        -  17.2.6   84d64ce389a6  b35fd925e076
ceph-exporter.os-ctrl3   os-ctrl3                running (23h)    88s ago   7M    16.4M        -  17.2.6   84d64ce389a6  5c1941242c6e
crash.ceph-osd1          ceph-osd1               running (6M)     89s ago   7M    14.4M        -  17.2.6   84d64ce389a6  23f74d6c4a88
crash.ceph-osd2          ceph-osd2               running (6M)     89s ago   7M    14.3M        -  17.2.6   84d64ce389a6  7c224db69c0f
crash.ceph-osd3          ceph-osd3               running (6M)     89s ago   7M    14.6M        -  17.2.6   84d64ce389a6  0b1c22101446
crash.os-ctrl1           os-ctrl1                running (23h)    87s ago   7M    7088k        -  17.2.6   84d64ce389a6  421a7614cbed
crash.os-ctrl2           os-ctrl2                running (23h)    88s ago   7M    7092k        -  17.2.6   84d64ce389a6  edab383569de
crash.os-ctrl3           os-ctrl3                running (23h)    88s ago   7M    7060k        -  17.2.6   84d64ce389a6  b4ac1f41004e
grafana.os-ctrl1         os-ctrl1   *:3000       running (23h)    87s ago   7M    69.1M        -  8.3.5    dad864ee21e9  f122af2ddb2a
mgr.os-ctrl1.dgyjfi      os-ctrl1   *:9283       running (23h)    87s ago   7M     421M        -  17.2.6   84d64ce389a6  0e5f5eccc3d8
mgr.os-ctrl2.bmqlnf      os-ctrl2   *:8443,9283  running (23h)    88s ago   7M     602M        -  17.2.6   84d64ce389a6  bcdae0cba0b2
mon.os-ctrl1             os-ctrl1                running (23h)    87s ago   7M     400M    2048M  17.2.6   84d64ce389a6  a59dda7e7d93
mon.os-ctrl2             os-ctrl2                running (23h)    88s ago   7M     395M    2048M  17.2.6   84d64ce389a6  48406e0821a3
mon.os-ctrl3             os-ctrl3                running (23h)    88s ago   7M     386M    2048M  17.2.6   84d64ce389a6  23bff20ba870
node-exporter.ceph-osd1  ceph-osd1  *:9100       running (6M)     89s ago   7M    37.2M        -  1.3.1    1dbe0e931976  b16c51aa919f
node-exporter.ceph-osd2  ceph-osd2  *:9100       running (6M)     89s ago   7M    39.3M        -  1.3.1    1dbe0e931976  d0b80e61bb04
node-exporter.ceph-osd3  ceph-osd3  *:9100       running (6M)     89s ago   7M    38.0M        -  1.3.1    1dbe0e931976  7ad70f112530
node-exporter.os-ctrl1   os-ctrl1   *:9100       running (23h)    87s ago   7M    22.4M        -  1.3.1    1dbe0e931976  97cc7705fa54
node-exporter.os-ctrl2   os-ctrl2   *:9100       running (23h)    88s ago   7M    22.4M        -  1.3.1    1dbe0e931976  9f6f27e983cf
node-exporter.os-ctrl3   os-ctrl3   *:9100       running (23h)    88s ago   7M    23.0M        -  1.3.1    1dbe0e931976  e00cee580326
osd.0                    ceph-osd3               running (6M)     89s ago   6M    2509M    24.8G  17.2.6   84d64ce389a6  a6ac6da7c87b
osd.1                    ceph-osd3               running (6M)     89s ago   6M    2733M    24.8G  17.2.6   84d64ce389a6  83781e6541e0
osd.2                    ceph-osd3               running (6M)     89s ago   6M    2330M    24.8G  17.2.6   84d64ce389a6  a43760560f5f
osd.3                    ceph-osd1               running (6M)     89s ago   7M    8086M    24.8G  17.2.6   84d64ce389a6  1da75db2a117
osd.4                    ceph-osd2               running (6M)     89s ago   7M    12.9G    24.8G  17.2.6   84d64ce389a6  835be17ea22c
osd.5                    ceph-osd3               running (6M)     89s ago   7M    13.2G    24.8G  17.2.6   84d64ce389a6  01a79d7e1009
osd.6                    ceph-osd1               running (6M)     89s ago   7M    8461M    24.8G  17.2.6   84d64ce389a6  82fc463c4920
osd.7                    ceph-osd2               running (6M)     89s ago   7M    9362M    24.8G  17.2.6   84d64ce389a6  fed55b03eebe
osd.8                    ceph-osd3               running (6M)     89s ago   7M    14.6G    24.8G  17.2.6   84d64ce389a6  ff143271b727
osd.9                    ceph-osd1               running (6M)     89s ago   7M    15.3G    24.8G  17.2.6   84d64ce389a6  3decd8978afd
osd.10                   ceph-osd2               running (6M)     89s ago   7M    14.2G    24.8G  17.2.6   84d64ce389a6  51e9e9261976
osd.11                   ceph-osd3               running (6M)     89s ago   7M    10.3G    24.8G  17.2.6   84d64ce389a6  238521d04ad4
osd.12                   ceph-osd1               running (6M)     89s ago   7M    8616M    24.8G  17.2.6   84d64ce389a6  fd4b492c5c6f
osd.13                   ceph-osd2               running (6M)     89s ago   7M    9105M    24.8G  17.2.6   84d64ce389a6  9b4bec06a6cb
osd.14                   ceph-osd3               running (6M)     89s ago   7M    6848M    24.8G  17.2.6   84d64ce389a6  a3007e5775b1
osd.15                   ceph-osd2               running (6M)     89s ago   6M    2398M    24.8G  17.2.6   84d64ce389a6  e6aef528287a
osd.16                   ceph-osd2               running (6M)     89s ago   6M    3180M    24.8G  17.2.6   84d64ce389a6  525439e38abb
osd.17                   ceph-osd2               running (6M)     89s ago   6M    2644M    24.8G  17.2.6   84d64ce389a6  b2d4abc6d570
osd.18                   ceph-osd1               running (6M)     89s ago   6M    2701M    24.8G  17.2.6   84d64ce389a6  9ba07d5c84c8
osd.19                   ceph-osd1               running (6M)     89s ago   6M    2843M    24.8G  17.2.6   84d64ce389a6  3bd5a34fe159
osd.20                   ceph-osd1               running (6M)     89s ago   6M    2921M    24.8G  17.2.6   84d64ce389a6  7390ffc3ef57
prometheus.os-ctrl1      os-ctrl1   *:9095       running (23h)    87s ago   7M     215M        -  2.33.4   514e6a882f6e  bd2d34e7bccb
```

Make sure ceph is in health stat because upgrade start 

```
$ ceph health
HEALTH_OK
```

### Start ceph upgrade to Reef

Check latest and stable released version [here](https://docs.ceph.com/en/latest/releases/). To upgrade run following command with proper release tag version. In my case its 18.2.2. 

```
$ ceph orch upgrade start --ceph-version 18.2.2
```

As soon as you will start upgrade it will first download docker images which you can see in following output 

```
$ docker images | grep quay
quay.io/ceph/ceph                                                  v18.2.2               a27483cc3ea0   34 hours ago    1.26GB
quay.io/ceph/ceph-grafana                                          9.4.7                 954c08fa6188   3 months ago    633MB
quay.io/ceph/ceph                                                  v17                   84d64ce389a6   8 months ago    1.26GB
```

Check upgrade status with following command.

```
$ ceph orch upgrade status
{
    "target_image": "quay.io/ceph/ceph@sha256:798f1b1e71ca1bbf76c687d8bcf5cd3e88640f044513ae55a0fb571502ae641f",
    "in_progress": true,
    "which": "Upgrading all daemon types on all hosts",
    "services_complete": [
        "mgr"
    ],
    "progress": "2/47 daemons upgraded",
    "message": "",
    "is_paused": false
}
```

First it will upgrade MGR daemon then MON. 

```
$ ceph orch ps | grep mgr
mgr.os-ctrl1.dgyjfi      os-ctrl1   *:8443,9283,8765  running (113s)    57s ago   7M     423M        -  18.2.2   a27483cc3ea0  e95dc6b2cb7d
mgr.os-ctrl2.bmqlnf      os-ctrl2   *:8443,9283,8765  running (2m)      42s ago   7M     490M        -  18.2.2   a27483cc3ea0  a34b4d66c32d
```

When it will upgrade OSD during that time it will restart OSD daemon which will trigger recovery.

```
$ ceph osd tree

-1         52.39732  root default
-3         17.46577      host ceph-osd1
18   nvme   1.94060          osd.18          up   1.00000  1.00000
19   nvme   1.94060          osd.19          up   1.00000  1.00000
20   nvme   1.94060          osd.20          up   1.00000  1.00000
 3    ssd   2.91100          osd.3           up   1.00000  1.00000
 6    ssd   2.91100          osd.6         down   1.00000  1.00000
 9    ssd   2.91100          osd.9           up   1.00000  1.00000
12    ssd   2.91100          osd.12          up   1.00000  1.00000
```

You can see recovery status here. 

```
$ ceph health
HEALTH_WARN 1 osds down; Degraded data redundancy: 113159/807519 objects degraded (14.013%), 165 pgs degraded
```

If you interested in upgrade logs. 

```
ceph log last cephadm
2024-03-21T02:34:53.206361+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 327 : cephadm [INF] Upgrade: osd.7 is safe to restart
2024-03-21T02:34:53.207197+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 328 : cephadm [INF] Upgrade: osd.4 is also safe to restart
2024-03-21T02:34:53.207369+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 329 : cephadm [INF] Upgrade: osd.10 is also safe to restart
2024-03-21T02:34:53.207522+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 330 : cephadm [INF] Upgrade: osd.13 is also safe to restart
2024-03-21T02:34:53.874655+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 331 : cephadm [INF] Upgrade: Updating osd.7 (1/4)
2024-03-21T02:34:53.905404+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 332 : cephadm [INF] Deploying daemon osd.7 on ceph-osd2
2024-03-21T02:35:24.855273+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 349 : cephadm [INF] Upgrade: Updating osd.4 (2/4)
2024-03-21T02:35:24.891699+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 350 : cephadm [INF] Deploying daemon osd.4 on ceph-osd2
2024-03-21T02:35:47.735582+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 362 : cephadm [INF] Upgrade: Updating osd.10 (3/4)
2024-03-21T02:35:47.768120+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 363 : cephadm [INF] Deploying daemon osd.10 on ceph-osd2
2024-03-21T02:36:11.963329+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 376 : cephadm [INF] Upgrade: Updating osd.13 (4/4)
2024-03-21T02:36:11.994598+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 377 : cephadm [INF] Deploying daemon osd.13 on ceph-osd2
2024-03-21T02:37:00.799319+0000 mgr.os-ctrl2.bmqlnf (mgr.36605239) 404 : cephadm [INF] Upgrade: unsafe to stop osd(s) at this time (23 PGs are or would become offline)
```

If all goes well then voila!! 

```
$ ceph version
ceph version 18.2.2 (531c0d11a1c5d39fbfe6aa8a521f023abf3bf3e2) reef (stable)
```

Enjoy!!! 

