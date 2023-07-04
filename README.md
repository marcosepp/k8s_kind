# Using KIND

## Install toolset

> TODO

* Install `docker`
* Install `kind`
* Install `kubectl`

## TL:DR - oneliner

```bash
kind create cluster --config kind-multinode.yaml --name kind; \
kubectl apply -f k8s_metallb-native.yaml; \
sleep 20; \
kubectl apply -f k8s_metallb-native-config.yaml; \
kubectl create namespace traefik; \
kubectl apply -f k8s_traefik_helm.yaml; \
sleep 5; \
kubectl apply -f k8s_traefik_helm.yaml; \
kubectl create namespace whoami-ingress; \
kubectl apply -f wl_whoami_ingress.yaml; \
sleep 20; \
curl test.homelab
```

Takes about `0.01s user 0.02s system 0% cpu 1:21.29 total` to have a multy node local k8s cluster with load-balancer, ingress and a example deployment.

## TL:DR

Create Kind cluster:

```bash
kind create cluster --config kind-multinode.yaml --name kind
```

Deploy MetalLB:

```bash
kubectl apply -f k8s_metallb-native.yaml
```

Configure MetalLB:

```bash
kubectl apply -f k8s_metallb-native-config.yaml
```

Deply Traefik:

```bash
kubectl apply -f k8s_traefik_helm.yaml && sleep 5 && kubectl apply -f k8s_traefik_helm.yaml
```

Create DNS A record: `test.homelab 127.0.0.1`

Deploy workload with MetalLB and Traefik:

```bash
kubectl create namespace whoami-ingress && kubectl apply -f wl_whoami_ingress.yaml
```

## Create cluster

```bash
$ kind create cluster --config kind-multinode.yaml --name kind
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼 
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/

```

### Check if cluster is runnig

```bash
$ kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   73s   v1.27.3
kind-worker          Ready    <none>          48s   v1.27.3
kind-worker2         Ready    <none>          52s   v1.27.3

```

### NOTE! To remove cluster

```bash
$ kind delete cluster --name kind
Deleting cluster "kind" ...
Deleted nodes: ["kind-worker3" "kind-worker2" "kind-control-plane" "kind-worker"]
```

## Install MetalLB & Traefik

### MetalLB

```bash
$ kubectl apply -f k8s_metallb-native.yaml
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io created
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
serviceaccount/controller created
serviceaccount/speaker created
role.rbac.authorization.k8s.io/controller created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/controller created
rolebinding.rbac.authorization.k8s.io/pod-lister created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
configmap/metallb-excludel2 created
secret/webhook-server-cert created
service/webhook-service created
deployment.apps/controller created
daemonset.apps/speaker created
validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration created

```

Check container network.

```bash
$ docker network inspect -f '{{.IPAM.Config}}' kind
[{172.19.0.0/16  172.19.0.1 map[]} {fc00:f853:ccd:e793::/64   map[]}]
```

Mofify _spec.addresses_ in `k8s_metallb-native-config.yaml` to be within your network.

```yaml
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
```

Apply `k8s_metallb-native-config.yaml`.

```bash
$ kubectl apply -f k8s_metallb-native-config.yaml 
ipaddresspool.metallb.io/example created
l2advertisement.metallb.io/empty created
```

NOTE! If you get an error like following then just wait a bit and try again:

```bash
$ kubectl apply -f k8s_metallb-native-config.yaml
Error from server (InternalError): error when creating "k8s_metallb-native-config.yaml": Internal error occurred: failed calling webhook "ipaddresspoolvalidationwebhook.metallb.io": failed to call webhook: Post "https://webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-ipaddresspool?timeout=10s": dial tcp 10.96.111.241:443: connect: connection refused
Error from server (InternalError): error when creating "k8s_metallb-native-config.yaml": Internal error occurred: failed calling webhook "l2advertisementvalidationwebhook.metallb.io": failed to call webhook: Post "https://webhook-service.metallb-system.svc:443/validate-metallb-io-v1beta1-l2advertisement?timeout=10s": dial tcp 10.96.111.241:443: connect: connection refused
$ kubectl apply -f k8s_metallb-native-config.yaml
ipaddresspool.metallb.io/example created
l2advertisement.metallb.io/empty created
```

Test load balancer.

```bash
#  Deploy wl_metallb_testing manifest
$ kubectl create namespace whoami-lb
namespace/whoami-lb created
$ kubectl apply -f wl_whoami_lb.yaml
deployment.apps/whoami created
service/whoami created
$ kubectl get all --namespace whoami-lb
NAME                          READY   STATUS    RESTARTS   AGE
pod/whoami-76c79d59c8-b25hn   1/1     Running   0          16s
pod/whoami-76c79d59c8-j49vt   1/1     Running   0          16s
pod/whoami-76c79d59c8-sxh4v   1/1     Running   0          16s

NAME             TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
service/whoami   LoadBalancer   10.96.135.22   172.19.255.201   80:30033/TCP   16s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   3/3     3            3           16s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-76c79d59c8   3         3         3       16s
```

Note the `service/whoami` TYPE and EXTERNAL-IP.

Test if loadbalancer is working as expected. Use `curl` command with the EXTERNAL-IP to curl whoami loadbalancer service from cluster control-plane.

```bash
$ docker exec -it kind-control-plane curl 172.19.255.201
Hostname: whoami-76c79d59c8-sxh4v
IP: 127.0.0.1
IP: ::1
IP: 10.244.2.4
IP: fe80::ce3:91ff:fe9f:695c
RemoteAddr: 172.19.0.5:8338
GET / HTTP/1.1
Host: 172.19.255.201
User-Agent: curl/7.74.0
Accept: */*

```

### Traefik

```bash
$ kubectl create namespace traefik
namespace/traefik created
$ kubectl apply -f k8s_traefik_helm.yaml 
customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/ingressrouteudps.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/middlewaretcps.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/serverstransports.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/tlsstores.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.containo.us created
customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.io created
customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.io created
customresourcedefinition.apiextensions.k8s.io/ingressrouteudps.traefik.io created
customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.io created
customresourcedefinition.apiextensions.k8s.io/middlewaretcps.traefik.io created
customresourcedefinition.apiextensions.k8s.io/serverstransports.traefik.io created
customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.io created
customresourcedefinition.apiextensions.k8s.io/tlsstores.traefik.io created
customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.io created
serviceaccount/release-name-traefik created
clusterrole.rbac.authorization.k8s.io/release-name-traefik-default created
clusterrolebinding.rbac.authorization.k8s.io/release-name-traefik-default created
service/release-name-traefik created
deployment.apps/release-name-traefik created
ingressclass.networking.k8s.io/release-name-traefik created
error: resource mapping not found for name: "release-name-traefik-dashboard" namespace: "traefik" from "k8s_traefik_helm.yaml": no matches for kind "IngressRoute" in version "traefik.io/v1alpha1"
ensure CRDs are installed first
$ kubectl apply -f k8s_traefik_helm.yaml
customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/ingressrouteudps.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/middlewaretcps.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/serverstransports.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/tlsstores.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.containo.us configured
customresourcedefinition.apiextensions.k8s.io/ingressroutes.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/ingressroutetcps.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/ingressrouteudps.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/middlewares.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/middlewaretcps.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/serverstransports.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/tlsoptions.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/tlsstores.traefik.io configured
customresourcedefinition.apiextensions.k8s.io/traefikservices.traefik.io configured
serviceaccount/release-name-traefik unchanged
clusterrole.rbac.authorization.k8s.io/release-name-traefik-default unchanged
clusterrolebinding.rbac.authorization.k8s.io/release-name-traefik-default unchanged
service/release-name-traefik unchanged
deployment.apps/release-name-traefik configured
ingressclass.networking.k8s.io/release-name-traefik unchanged
ingressroute.traefik.io/release-name-traefik-dashboard created
```

Create dns A record in your DNS server or modify your `/etc/hosts` to point test.homelab to 127.0.0.1.

Test if ingress is working.

```bash
$ kubectl create namespace whoami-ingress
namespace/whoami-ingress created
$ kubectl apply -f wl_whoami_ingress.yaml 
deployment.apps/whoami created
service/whoami created
ingress.networking.k8s.io/whoami created
$ kubectl get all --namespace whoami-ingress
NAME                          READY   STATUS    RESTARTS   AGE
pod/whoami-76c79d59c8-7lv7p   1/1     Running   0          16s
pod/whoami-76c79d59c8-tljnf   1/1     Running   0          16s
pod/whoami-76c79d59c8-wvf8c   1/1     Running   0          16s

NAME             TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
service/whoami   LoadBalancer   10.96.34.31   172.19.255.202   80:30330/TCP   16s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   3/3     3            3           16s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-76c79d59c8   3         3         3       16s
$ kubectl get ingress --namespace whoami-ingress
NAME     CLASS                  HOSTS          ADDRESS   PORTS   AGE
whoami   release-name-traefik   test.homelab             80      33s
$ curl test.homelab
Hostname: whoami-76c79d59c8-jr9nz
IP: 127.0.0.1
IP: ::1
IP: 10.244.2.3
IP: fe80::940c:a5ff:fef8:b1bf
RemoteAddr: 10.244.0.5:47820
GET / HTTP/1.1
Host: test.homelab
User-Agent: curl/7.88.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.244.0.1
X-Forwarded-Host: test.homelab
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: release-name-traefik-6459f5858b-kk6lw
X-Real-Ip: 10.244.0.1
```

## Author

Marco Sepp
