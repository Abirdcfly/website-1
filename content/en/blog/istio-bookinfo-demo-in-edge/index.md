---
authors:
- Abirdcfly 
categories:
- Demo 
date: 2021-03-12
draft: false
image:
caption: try istio bookinfo demo in edge
focal_point: Center
lastmod: 2021-03-12
summary: try istio bookinfo demo in edge.
tags:
- KubeEdge
- kubeedge
- edge computing
- kubernetes edge computing
- K8S edge orchestration
- edge computing platform
- istio
title: try istio bookinfo demo in edge
---

show how to run a istio bookinfo demo in edge node base on cni flannel with edge list-watch feature.
## prepare in edge
### install docker in edge node
follow [https://docs.docker.com/engine/install/]
### download cni binary to edge node
download cni plugins from [https://github.com/containernetworking/plugins/releases] 
````
wget "https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz"
mkdir -p /opt/cni/bin
tar xf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin
````

## install kubeedge
### cloud node init
````
./keadm init --advertise-address 192.168.1.113
````
#### change config
````
# vim /etc/kubeedge/config/cloudcore.yaml
modules:
  ..
  dynamicController:
    enable: true
  ..
````

#### restart cloudcore
````
pkill cloudcore ; nohup /usr/local/bin/cloudcore > /var/log/kubeedge/cloudcore.log 2>&1 &
````

### edge node join
````
./keadm join  --cgroupdriver systemd -e 192.168.1.113:10000 --token d38d3ee01d66ff283d502c6970447b1f1bf9dc35e5889c8273c6cbedc8a4d858.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MTU1MzMyNDR9.1RpMAoo8KiKb_a_bXfhcPq-JemYVdwOmHq-4wE8yrqc
````

#### change config
````
# vim /etc/kubeedge/config/edgecore.yaml
modules:
  ..
  edgeMesh:
    enable: false
  ..
  edged:
    ..
    networkPluginName: cni
    clusterDNS: "10.96.0.10"
    clusterDomain: "cluster.local"
  ..
  metaManager:
    mataServer:
      enable: true
  ..
````
`clusterDomain` and `clusterDNS` should be consistent with your kubernetes cluster.

#### restart edgecore
````
service edgecore restart
````
## install istio
follow [https://istio.io/latest/docs/setup/getting-started/#install] to install istio.
## bookinfo example
Because we want to try on edge, just add nodeSelector in pod template in `samples/bookinfo/platform/kube/bookinfo.yaml ` and `kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml `
````
# kubectl get node -l node-role.kubernetes.io/edge -o wide
NAME                    STATUS   ROLES        AGE   VERSION                   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
localhost.localdomain   Ready    agent,edge   16h   v1.19.3-kubeedge-v1.6.0   192.168.2.135   <none>        CentOS Linux 7 (Core)   3.10.0-1160.15.2.el7.x86_64   docker://20.10.5

# kubectl get po -n default -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE                    NOMINATED NODE   READINESS GATES
details-v1-79b7dcdf96-cjmm5       2/2     Running   0          14h   172.31.5.8    localhost.localdomain   <none>           <none>
productpage-v1-64f644d8b6-vg9t7   2/2     Running   0          14h   172.31.5.10   localhost.localdomain   <none>           <none>
ratings-v1-b84b7465b-9nxfn        2/2     Running   0          14h   172.31.5.7    localhost.localdomain   <none>           <none>
reviews-v1-779f4d69b8-tsjxf       2/2     Running   0          14h   172.31.5.12   localhost.localdomain   <none>           <none>
reviews-v2-b5b4b579d-8h4t5        2/2     Running   0          14h   172.31.5.11   localhost.localdomain   <none>           <none>
reviews-v3-6c4c5fc7b-sw892        2/2     Running   0          14h   172.31.5.9    localhost.localdomain   <none>           <none>
````
check results:
````
# curl in another node, you can see the whole html page.
curl -k http://172.31.5.10:9080/productpage

# and productpage sidecar log show like above.
[2021-03-12T01:53:22.334Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 54 53 "-" "curl/7.29.0" "b218d0d6-1038-96a7-8d2d-5d27b9193e66" "details:9080" "172.31.5.8:9080" outbound|9080||details.default.svc.cluster.local 172.31.5.10:33918 10.106.56.126:9080 172.31.5.10:39348 - default
[2021-03-12T01:53:22.415Z] "GET /reviews/0 HTTP/1.1" 200 - via_upstream - "-" 0 379 1409 1408 "-" "curl/7.29.0" "b218d0d6-1038-96a7-8d2d-5d27b9193e66" "reviews:9080" "172.31.5.11:9080" outbound|9080||reviews.default.svc.cluster.local 172.31.5.10:36650 10.106.193.62:9080 172.31.5.10:48704 - default
[2021-03-12T01:53:22.253Z] "GET /productpage HTTP/1.1" 200 - via_upstream - "-" 0 5183 1653 1602 "-" "curl/7.29.0" "b218d0d6-1038-96a7-8d2d-5d27b9193e66" "172.31.5.10:9080" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:53842 172.31.5.10:9080 172.31.1.0:33090 - default
[2021-03-12T01:53:42.559Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 2 2 "-" "curl/7.29.0" "0fc74561-e39f-9b74-89c6-c2eadabd52f2" "details:9080" "172.31.5.8:9080" outbound|9080||details.default.svc.cluster.local 172.31.5.10:33918 10.106.56.126:9080 172.31.5.10:39488 - default
[2021-03-12T01:53:42.567Z] "GET /reviews/0 HTTP/1.1" 200 - via_upstream - "-" 0 295 958 957 "-" "curl/7.29.0" "0fc74561-e39f-9b74-89c6-c2eadabd52f2" "reviews:9080" "172.31.5.12:9080" outbound|9080||reviews.default.svc.cluster.local 172.31.5.10:35916 10.106.193.62:9080 172.31.5.10:48840 - default
[2021-03-12T01:53:42.554Z] "GET /productpage HTTP/1.1" 200 - via_upstream - "-" 0 4183 995 995 "-" "curl/7.29.0" "0fc74561-e39f-9b74-89c6-c2eadabd52f2" "172.31.5.10:9080" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:53982 172.31.5.10:9080 172.31.1.0:33222 - default
[2021-03-12T01:53:54.708Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 49 15 "-" "curl/7.29.0" "e2aed470-689b-95d7-8eeb-a33aadd146bd" "details:9080" "172.31.5.8:9080" outbound|9080||details.default.svc.cluster.local 172.31.5.10:34140 10.106.56.126:9080 172.31.5.10:39570 - default
[2021-03-12T01:53:54.779Z] "GET /reviews/0 HTTP/1.1" 200 - via_upstream - "-" 0 295 28 27 "-" "curl/7.29.0" "e2aed470-689b-95d7-8eeb-a33aadd146bd" "reviews:9080" "172.31.5.12:9080" outbound|9080||reviews.default.svc.cluster.local 172.31.5.10:36004 10.106.193.62:9080 172.31.5.10:48928 - default
[2021-03-12T01:53:54.702Z] "GET /productpage HTTP/1.1" 200 - via_upstream - "-" 0 4183 120 119 "-" "curl/7.29.0" "e2aed470-689b-95d7-8eeb-a33aadd146bd" "172.31.5.10:9080" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:54064 172.31.5.10:9080 172.31.1.0:33304 - default
[2021-03-12T01:56:19.695Z] "GET /details/0 HTTP/1.1" 200 - via_upstream - "-" 0 178 3 3 "-" "curl/7.29.0" "8d50ef9d-b41c-9c67-b39d-9d513579add8" "details:9080" "172.31.5.8:9080" outbound|9080||details.default.svc.cluster.local 172.31.5.10:34140 10.106.56.126:9080 172.31.5.10:40458 - default
[2021-03-12T01:56:19.705Z] "GET /reviews/0 HTTP/1.1" 200 - via_upstream - "-" 0 375 1246 1245 "-" "curl/7.29.0" "8d50ef9d-b41c-9c67-b39d-9d513579add8" "reviews:9080" "172.31.5.9:9080" outbound|9080||reviews.default.svc.cluster.local 172.31.5.10:33126 10.106.193.62:9080 172.31.5.10:49812 - default
[2021-03-12T01:56:19.688Z] "GET /productpage HTTP/1.1" 200 - via_upstream - "-" 0 5179 1305 1305 "-" "curl/7.29.0" "8d50ef9d-b41c-9c67-b39d-9d513579add8" "172.31.5.10:9080" "127.0.0.1:9080" inbound|9080|| 127.0.0.1:54952 172.31.5.10:9080 172.31.1.0:34224 - default

````
tips：
1. in this demo, sidecar need to connect istiod service. you can expose istiod service use a ingress. and use it just add hostAliases in pod template:
   ````
    hostAliases:
    - ip: "192.168.1.118"
      hostnames:
        - "istiod.istio-system.svc"

   ````

## use mosn for istio
follow [https://mosn.io/docs/quick-start/istio/] to change enovy to mosn.
mosn now support istio 1.5.2.