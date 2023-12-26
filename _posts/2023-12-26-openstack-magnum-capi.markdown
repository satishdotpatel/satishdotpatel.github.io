---
title: "Openstack Magnum Cluster API"
layout: post
date: 2023-12-26
image: /assets/images/2023-12-26-openstack-magnum-capi.markdown/magnum-logo.png
headerImage: true
tag:
- Openstack
- Magnum
- k8s
- ClusterAPI
- Kolla-Ansible
category: blog
blog: true
author: Satish Patel
description: "Openstack Magnum Cluster API"

---

The Cluster API driver for Magnum allows you to deploy fully conformant Kubernetes cluster using the Cluster API. In short CAPI requires a dedicated “k8s management plane”. This means you need a Kubernetes cluster to manage/deploy your Kubernetes clusters. 

[Magnum Cluster Api](https://github.com/vexxhost/magnum-cluster-api)
[Magnum CAPI Operations](https://vexxhost.github.io/magnum-cluster-api/user/getting-started/)
[CAPI official Doc](https://cluster-api.sigs.k8s.io/user/quick-start.html)

### Components 

* Management clusters: These clusters are responsible for the creation and oversight of other clusters through Cluster API. They are your agents for infrastructure management, and in that respect are a bit of an oddity—they’re probably not running application workloads like a normal Kubernetes cluster, but instead focusing entirely on provisioning, monitoring, and managing other clusters. 

* Workload clusters: These are the clusters that will actually handle application workloads for your users—the business-as-usual clusters that do exactly what you would expect a Kubernetes cluster to do, running microservices and handling requests. 


![<img>](/assets/images/2023-12-26-openstack-magnum-capi/magnum-capi.png){: width="1000" }


### Rrerequisite 

Working openstack environment with Magnum. I am running kolla-ansible so all my configuration will reflect related kolla-ansible. If you have different deployment tool then change according. 

### Installation of kind k8s cluster

Deploy kind cluster for Managment cluster (This can be outside your openstack infra). Kind cluster required docker engin so make sure you have already installed docker. For testing kind k8s cluster is good tool but in production please use kubesprey or any other production grade cluster tool. 

```
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

Create kind k8s cluster config file. apiServerAddress is your interface IP. 

```
$ cat kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "192.168.18.234"
  apiServerPort: 6443 
```

Create cluster 

```
$ kind create cluster --config=kind-config.yaml
```

Install kubectl to manage cluster

```
$ curl -LO https://dl.k8s.io/release/v1.27.4/bin/linux/amd64/kubectl
$ chmod 700 kubectl
$ mv kubectl /usr/local/bin/.

$ kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   7d    v1.27.3
```

### Deploy Cluster API 

Deploy Cluster API on kind based k8s cluster. 

```
$ CAPI_VERSION=${CAPI_VERSION:-v1.5.1}
$ sudo curl -Lo /usr/local/bin/clusterctl https://github.com/kubernetes-sigs/cluster-api/releases/download/${CAPI_VERSION}/clusterctl-linux-amd64
$ sudo chmod +x /usr/local/bin/clusterctl

$ export EXP_CLUSTER_RESOURCE_SET=true
$ export EXP_KUBEADM_BOOTSTRAP_FORMAT_IGNITION=true 
$ export CLUSTER_TOPOLOGY=true

$ clusterctl init --core cluster-api --bootstrap kubeadm --control-plane kubeadm --infrastructure openstack
```

Verify deployment 

```
$ kubectl get deploy -A
NAMESPACE                           NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager       1/1     1            1           7d
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager   1/1     1            1           7d
capi-system                         capi-controller-manager                         1/1     1            1           7d
capo-system                         capo-controller-manager                         1/1     1            1           7d
cert-manager                        cert-manager                                    1/1     1            1           7d
cert-manager                        cert-manager-cainjector                         1/1     1            1           7d
cert-manager                        cert-manager-webhook                            1/1     1            1           7d
kube-system                         coredns                                         2/2     2            2           7d
local-path-storage                  local-path-provisioner                          1/1     1            1           7d
```

Verify logs incase of any failure

```
$ kubectl -n cert-manager logs deploy/cert-manager -f
$ kubectl -n cert-manager logs deploy/cert-manager-cainjector -f
$ kubectl -n cert-manager logs deploy/cert-manager-webhook -f

$ kubectl -n capi-system logs deploy/capi-controller-manager -f
$ kubectl -n capo-system logs deploy/capi-controller-manager -f
```

### Install magnum and capi driver

Install magnum-capi driver on your magnum container. (I have already deployed kolla-ansible with magnum). In following steps I will install magnum-capi driver inside magnum containers. This step can be different according your deployment tools.

In kolla-ansible you need to add following config to allow calico cni. 

```
$ cat /etc/kolla/config/magnum.conf
[cluster_template]
kubernetes_allowed_network_drivers = calico
kubernetes_default_network_driver = calico
```

Install magnum-cluster-api driver on both containers magnum_api and magnum_conductor. 

```
$ docker exec -it -u0 magnum_api bash
(magnum-api)[root@os2-ctrl-01 /]# apt update
(magnum-api)[root@os2-ctrl-01 /]# pip install magnum-cluster-api
(magnum-api)[root@os2-ctrl-01 /]# wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz
(magnum-api)[root@os2-ctrl-01 /]# tar -zxvf helm-v3.13.2-linux-amd64.tar.gz
(magnum-api)[root@os2-ctrl-01 /]# mv linux-amd64/helm /usr/local/bin/

$ docker exec -it -u0 magnum_conductor bash
(magnum-conductor)[root@os2-ctrl-01 /]# apt update
(magnum-conductor)[root@os2-ctrl-01 /]# pip install magnum-cluster-api
(magnum-conductor)[root@os2-ctrl-01 /]# wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz
(magnum-conductor)[root@os2-ctrl-01 /]# tar -zxvf helm-v3.13.2-linux-amd64.tar.gz 
(magnum-conductor)[root@os2-ctrl-01 /]# mv linux-amd64/helm /usr/local/bin/
```

Copy ~/.kube/config file from kind k8s cluster to magnum_api and magnum_conductor containers so it can access kind k8s clusters. In have copied my ~/.kube/config file on all openstack controller nodes in /root directory and then running following docker command to copy inside container and changing ownership. 

```
$ docker exec -it -u0 magnum_api mkdir /var/lib/magnum/.kube
$ docker exec -it -u0 magnum_conductor mkdir /var/lib/magnum/.kube

$ docker cp /root/config magnum_api:/var/lib/magnum/.kube/.
$ docker cp /root/config magnum_conductor:/var/lib/magnum/.kube/.

$ docker exec -it -u0 magnum_api chown magnum:magnum /var/lib/magnum/.kube/config
$ docker exec -it -u0 magnum_conductor chown magnum:magnum /var/lib/magnum/.kube/config
```

Restart containers

```
$ docker restart magnum_api magnum_conductor
```

Verify logs of any errors

```
$ tail -f /var/log/kolla/magnum/magnum-*.log
```

If all good then you will notice in logs that magnum makes calls to kind k8s cluster running ourside openstack. If it failed with any reason then you have to make sure its reachable/routable or no firewall in between. (you can use curl etc tools to verify)

### Import Kubernetes images and create templates

You may need to adjust some key/values like external-network and flavors etc. 

```
export OS_DISTRO=ubuntu # you can change this to "flatcar" if you want to use Flatcar
for version in v1.24.16 v1.25.12 v1.26.7 v1.27.4; do \
  [[ "${OS_DISTRO}" == "ubuntu" ]] && IMAGE_NAME="ubuntu-2204-kube-${version}" || IMAGE_NAME="flatcar-kube-${version}"; \
  curl -LO https://object-storage.public.mtl1.vexxhost.net/swift/v1/a91f106f55e64246babde7402c21b87a/magnum-capi/${IMAGE_NAME}.qcow2; \
  openstack image create ${IMAGE_NAME} --disk-format=qcow2 --container-format=bare --property os_distro=${OS_DISTRO} --file=${IMAGE_NAME}.qcow2; \
  openstack coe cluster template create \
      --image $(openstack image show ${IMAGE_NAME} -c id -f value) \
      --external-network public \
      --dns-nameserver 8.8.8.8 \
      --master-lb-enabled \
      --master-flavor m1.medium \
      --flavor m1.medium \
      --network-driver calico \
      --docker-storage-driver overlay2 \
      --coe kubernetes \
      --label kube_tag=${version} \
      k8s-${version};
done;
```

### Create k8s workload cluster in openstack 


```
$ openstack coe cluster create --cluster-template k8s-v1.27.4 --master-count 1 --node-count 2 --keypair spatel-key my-cluster1
```

```
$ # openstack coe cluster list
+--------------------------------------+--------------+------------+------------+--------------+-----------------+---------------+
| uuid                                 | name         | keypair    | node_count | master_count | status          | health_status |
+--------------------------------------+--------------+------------+------------+--------------+-----------------+---------------+
| 8ce037aa-95f8-4eb4-91af-0c8fc6574f7f | my-cluster1  | spatel-key |          2 |            1 | CREATE_COMPLETE | HEALTHY       |
+--------------------------------------+--------------+------------+------------+--------------+-----------------+---------------+
```





 



