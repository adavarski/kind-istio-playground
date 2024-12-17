## Kind Istio Playground

Isto playground in a kind-cluster

Istio: An open platform to connect, manage, and secure microservices.

- For in-depth information about how to use Istio, visit [istio.io](https://istio.io)
- To ask questions and get assistance from community, visit [discuss.istio.io](https://discuss.istio.io)
- To learn how to participate in overall community, visit [our community page](https://istio.io/about/community)

In this README:

- [Introduction](#introduction)
- [Repositories](#repositories)

In addition, here are some other documents you may wish to read:

- [Istio Community](https://github.com/istio/community#istio-community) - describes how to get involved and contribute to the Istio project
- [Istio Developer's Guide](https://github.com/istio/istio/wiki/Preparing-for-Development) - explains how to set up and use an Istio development environment
- [Project Conventions](https://github.com/istio/istio/wiki/Development-Conventions) - describes the conventions we use within the code base
- [Creating Fast and Lean Code](https://github.com/istio/istio/wiki/Writing-Fast-and-Lean-Code) - performance-oriented advice and guidelines for the code base

You'll find many other useful documents on [Wiki](https://github.com/istio/istio/wiki).

## Introduction

[Istio](https://istio.io/latest/docs/concepts/what-is-istio/) is an open platform for providing a uniform way to [integrate
microservices](https://istio.io/latest/docs/examples/microservices-istio/), manage [traffic flow](https://istio.io/latest/docs/concepts/traffic-management/) across microservices, enforce policies
and aggregate telemetry data. Istio's control plane provides an abstraction
layer over the underlying cluster management platform, such as Kubernetes.

Istio is composed of these components:

- **Envoy** - Sidecar proxies per microservice to handle ingress/egress traffic
   between services in the cluster and from a service to external
   services. The proxies form a _secure microservice mesh_ providing a rich
   set of functions like discovery, rich layer-7 routing, circuit breakers,
   policy enforcement and telemetry recording/reporting
   functions.

  > Note: The service mesh is not an overlay network. It
  > simplifies and enhances how microservices in an application talk to each
  > other over the network provided by the underlying platform.

- **Istiod** - The Istio control plane. It provides service discovery, configuration and certificate management. It consists of the following sub-components:

    - **Pilot** - Responsible for configuring the proxies at runtime.

    - **Citadel** - Responsible for certificate issuance and rotation.

    - **Galley** - Responsible for validating, ingesting, aggregating, transforming and distributing config within Istio.

- **Operator** - The component provides user friendly options to operate the Istio service mesh.

## Repositories

The Istio project is divided across a few GitHub repositories:

- [istio/api](https://github.com/istio/api). This repository defines
component-level APIs and common configuration formats for the Istio platform.

- [istio/community](https://github.com/istio/community). This repository contains
information on the Istio community, including the various documents that govern
the Istio open source project.

- [istio/istio](README.md). This is the main code repository. It hosts Istio's
core components, install artifacts, and sample programs. It includes:

    - [istioctl](istioctl/). This directory contains code for the
[_istioctl_](https://istio.io/latest/docs/reference/commands/istioctl/) command line utility.

    - [operator](operator/). This directory contains code for the
[Istio Operator](https://istio.io/latest/docs/setup/install/operator/).

    - [pilot](pilot/). This directory
contains platform-specific code to populate the
[abstract service model](https://istio.io/docs/concepts/traffic-management/#pilot), dynamically reconfigure the proxies
when the application topology changes, as well as translate
[routing rules](https://istio.io/latest/docs/reference/config/networking/) into proxy specific configuration.

    - [security](security/). This directory contains [security](https://istio.io/latest/docs/concepts/security/) related code,
including Citadel (acting as Certificate Authority), citadel agent, etc.

- [istio/proxy](https://github.com/istio/proxy). The Istio proxy contains
extensions to the [Envoy proxy](https://github.com/envoyproxy/envoy) (in the form of
Envoy filters) that support authentication, authorization, and telemetry collection.


## Istio’s Architecture

Istio needs to intercept all the network communication to and from every service and apply a set of rules. This is achieved and logically split into two planes: The Data Plane and The Control Plane.

### The Data Plane
In Kubernetes the network traffic between pods is managed using Services as shown:

<img src="pictures/Network-traffic-in-Kubernetes.png?raw=true" width="1000">

But to intercept all the network communication Istio injects an intelligent Envoy proxy as a sidecar in every pod. The injected proxies represent the data plane. Using those proxies Istio easily can achieve our requirements, for an example let’s check out the retrying and Circuit breaking functionalities.

<img src="pictures/Envoys-in-action.gif?raw=true" width="1000">

To summarize:

Envoy sends a request to the first instance of service B and it fails.
The Envoy Sidecar retries. (1)
Returns a failed requested to the calling proxy.
Which opens the Circuit Breaker and calls the next Service on subsequent requests. (2)
This means that you don’t have to use another Retry library, you don’t have to develop your own implementation of Circuit Breaking and Service Discovery in Programming Language X, Y or Z. All of those and more are provided out of the box by Istio and NO code changes are required.

Great! Now, you want to join the voyage, but you still have some doubts, some open questions. Is this a One-Size-Fits-All Solution, which you are suspicious about, as it always ends up being One-Size-Fits None solution!

You finally mutter the question: Is this configurable?

Welcome to the cruise and let’s get introduced to the Control Plane.

### The Control Plane:

Is composed of three components: The Pilot, the Mixer, and the Citadel that in combination configure Envoys to route traffic, enforce policies and collect telemetry data. Visually presented in the image below:

<img src="pictures/The-Control-Plane-CORRECT.png?raw=true" width="1000">

The envoys are configurable using Kubernetes [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) provided by Istio and specialized for this purpose. Which means that for you it’s just another Kubernetes Resource with a familiar syntax. The same way you defined a service and applied it using the command kubectl apply the same we will do with Istio resources. And the control plane will take over applying the configuration to the envoys.

A deeper dive into the internals of the Control Plane is out of the scope of this article and would replicate the [official documentation](https://istio.io/docs/concepts/what-is-istio/#mixer), a redundancy I would like to refrain from.


## Istio Components Summary:

<img src="pictures/Istio-Components.jpg?raw=true" width="1000">


## Prerequisite:

- Docker installed
- Kind installed
- kubectl installed


## Create kubernetes cluster with kind (over docker) and metallb
```
$ cat /etc/docker/daemon.json 
{
  "default-address-pools": [
    {"base": "172.17.0.0/16", "size": 24}
  ]
}

$ docker network rm kind
$ sudo service docker restart

$ kind create cluster
$ kubectl get pod -A

NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-565d847f94-ddw95                     1/1     Running   0          15s
kube-system          coredns-565d847f94-gbtrl                     1/1     Running   0          15s
kube-system          etcd-kind-control-plane                      1/1     Running   0          29s
kube-system          kindnet-6xbvn                                1/1     Running   0          16s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          29s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          29s
kube-system          kube-proxy-vqbfl                             1/1     Running   0          16s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          29s
local-path-storage   local-path-provisioner-684f458cdd-snjr9      1/1     Running   0          15s


REF: https://metallb.universe.tf/installation/ -> kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml OR kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml

$ LB_NET_PREFIX=$(docker network inspect -f '{{range .IPAM.Config }}{{ .Gateway }}{{end}}' kind | cut -d '.' -f1,2,3)
172.17.1

$ LB_NET_IP_FIRST=${LB_NET_PREFIX}.255.200
$ LB_NET_IP_LAST=${LB_NET_PREFIX}.255.250
$ kubectl apply -f - <<EOT
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - ${LB_NET_IP_FIRST}-${LB_NET_IP_LAST}
EOT
```

Or via helm
```
  helm upgrade --install --wait --timeout 35m --atomic --namespace metallb-system --create-namespace \
    --repo https://metallb.github.io/metallb metallb metallb

$ LB_NET_PREFIX=$(docker network inspect -f '{{range .IPAM.Config }}{{ .Gateway }}{{end}}' kind | cut -d '.' -f1,2,3)
172.17.1

$ LB_NET_IP_FIRST=${LB_NET_PREFIX}.200
$ LB_NET_IP_LAST=${LB_NET_PREFIX}.250


  kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - ${LB_NET_IP_FIRST}-${LB_NET_IP_LAST}
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: layer2
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

## Install istio and addons

```
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.24.1
$ export PATH=$PWD/bin:$PATH

$ istioctl install --set profile=demo -y
✔ Istio core installed                                                                                                                                                                                                                                                                                                        
✔ Istiod installed                                                                                                                                                                                                                                                                                                            
✔ Egress gateways installed                                                                                                                                                                                                                                                                                                   
✔ Ingress gateways installed                                                                                                                                                                                                                                                                                                  
✔ Installation complete                                                                                                                                                                                                                                                                                                       Made this installation the default for injection and validation.

# Enable namespace-auto-injection for default ns
$ kubectl label namespace default  istio-injection=enabled
$ kubectl get namespaces --label-columns=istio-injection
NAME                 STATUS   AGE     ISTIO-INJECTION
default              Active   6m12s   enabled
istio-system         Active   80s     
kube-node-lease      Active   6m14s   
kube-public          Active   6m14s   
kube-system          Active   6m14s   
local-path-storage   Active   6m9s    
metallb-system       Active   5m4s    

# Install addons: kiali, prometheus, etc
$ kubectl apply -f samples/addons
$ kubectl rollout status deployment/kiali -n istio-system
$ ISTIOINGRESSGW_LB_IP=$(kubectl -n istio-system get svc/istio-ingressgateway -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ echo "$ISTIOINGRESSGW_LB_IP istioigw" | sudo tee -a /etc/hosts
172.17.1.200 istioigw

```

## Install an app prepared for istio

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

# Check app services, and pods with auto-injected sidecar 2/2
$ kubectl get services
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
details           ClusterIP   10.96.113.45    <none>        9080/TCP            34s
kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP             9m26s
loki              ClusterIP   10.96.36.222    <none>        3100/TCP,9095/TCP   2m50s
loki-memberlist   ClusterIP   None            <none>        7946/TCP            2m50s
productpage       ClusterIP   10.96.108.103   <none>        9080/TCP            33s
ratings           ClusterIP   10.96.155.255   <none>        9080/TCP            34s
reviews           ClusterIP   10.96.254.71    <none>        9080/TCP            33s

$ timeout 60 watch kubectl get pods
$ kubectl get po
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-ccbdcf56-wl8mf         2/2     Running   0          3m53s
loki-0                            2/2     Running   0          6m9s
productpage-v1-6c4f8467b9-6vhzf   2/2     Running   0          3m53s
ratings-v1-96d98f9f6-6hmdx        2/2     Running   0          3m53s
reviews-v1-57c85f47fb-tr98n       2/2     Running   0          3m53s
reviews-v2-64776cb9bd-pwxwt       2/2     Running   0          3m53s
reviews-v3-5c8886f9c6-t5vc9       2/2     Running   0          3m53s
$  kubectl describe pod productpage-v1-6c4f8467b9-6vhzf
Name:             productpage-v1-6c4f8467b9-6vhzf
Namespace:        default
Priority:         0
Service Account:  bookinfo-productpage
Node:             kind-control-plane/172.17.0.2
Start Time:       Mon, 25 Sep 2023 11:51:51 +0300
Labels:           app=productpage
                  pod-template-hash=6c4f8467b9
                  security.istio.io/tlsMode=istio
                  service.istio.io/canonical-name=productpage
                  service.istio.io/canonical-revision=v1
                  version=v1
Annotations:      istio.io/rev: default
                  kubectl.kubernetes.io/default-container: productpage
                  kubectl.kubernetes.io/default-logs-container: productpage
                  prometheus.io/path: /stats/prometheus
                  prometheus.io/port: 15020
                  prometheus.io/scrape: true
                  sidecar.istio.io/status:
                    {"initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["workload-socket","credential-socket","workload-certs","istio-env...
Status:           Running
IP:               10.244.0.20
IPs:
  IP:           10.244.0.20
Controlled By:  ReplicaSet/productpage-v1-6c4f8467b9
Init Containers:
  istio-init:
    Container ID:  containerd://89e36a2d9fd05e9a5f64221a3bb6830c5e4291be311382e4dc2789c9d17e9e3e
    Image:         docker.io/istio/proxyv2:1.19.0
    Image ID:      docker.io/istio/proxyv2@sha256:35ad0149ac3d02911541198607cd876abbecc981083e9daa4748aeab23674456
    Port:          <none>
    Host Port:     <none>
    Args:
      istio-iptables
      -p
      15001
      -z
      15006
      -u
      1337
      -m
      REDIRECT
      -i
      *
      -x
      
      -b
      *
      -d
      15090,15021,15020
      --log_output_level=default:info
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Mon, 25 Sep 2023 11:51:54 +0300
      Finished:     Mon, 25 Sep 2023 11:51:54 +0300
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  1Gi
    Requests:
      cpu:        10m
      memory:     40Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-m6tn4 (ro)
Containers:
  productpage:
    Container ID:   containerd://a109db913e73f7df845758648cf69e9f0d9f4b41a705e58ebd62785661ec09d1
    Image:          docker.io/istio/examples-bookinfo-productpage-v1:1.18.0
    Image ID:       docker.io/istio/examples-bookinfo-productpage-v1@sha256:e2723a59bde95d630ed96630af25058ebea255219161d6a8ac6c25d67d7e1b5a
    Port:           9080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 25 Sep 2023 11:53:59 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /tmp from tmp (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-m6tn4 (ro)
  istio-proxy:
    Container ID:  containerd://8be642f5dd0e80851265cf8e1c63cae34eddc902c1bffade8741aa60d138eb2c
    Image:         docker.io/istio/proxyv2:1.19.0
    Image ID:      docker.io/istio/proxyv2@sha256:35ad0149ac3d02911541198607cd876abbecc981083e9daa4748aeab23674456
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=default:info
    State:          Running
      Started:      Mon, 25 Sep 2023 11:53:59 +0300
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  1Gi
    Requests:
      cpu:      10m
      memory:   40Mi
    Readiness:  http-get http://:15021/healthz/ready delay=1s timeout=3s period=2s #success=1 #failure=30
    Environment:
      JWT_POLICY:                    third-party-jwt
      PILOT_CERT_PROVIDER:           istiod
      CA_ADDR:                       istiod.istio-system.svc:15012
      POD_NAME:                      productpage-v1-6c4f8467b9-6vhzf (v1:metadata.name)
      POD_NAMESPACE:                 default (v1:metadata.namespace)
      INSTANCE_IP:                    (v1:status.podIP)
      SERVICE_ACCOUNT:                (v1:spec.serviceAccountName)
      HOST_IP:                        (v1:status.hostIP)
      ISTIO_CPU_LIMIT:               2 (limits.cpu)
      PROXY_CONFIG:                  {}
                                     
      ISTIO_META_POD_PORTS:          [
                                         {"containerPort":9080,"protocol":"TCP"}
                                     ]
      ISTIO_META_APP_CONTAINERS:     productpage
      GOMEMLIMIT:                    1073741824 (limits.memory)
      GOMAXPROCS:                    2 (limits.cpu)
      ISTIO_META_CLUSTER_ID:         Kubernetes
      ISTIO_META_NODE_NAME:           (v1:spec.nodeName)
      ISTIO_META_INTERCEPTION_MODE:  REDIRECT
      ISTIO_META_WORKLOAD_NAME:      productpage-v1
      ISTIO_META_OWNER:              kubernetes://apis/apps/v1/namespaces/default/deployments/productpage-v1
      ISTIO_META_MESH_ID:            cluster.local
      TRUST_DOMAIN:                  cluster.local
      ISTIO_PROMETHEUS_ANNOTATIONS:  {"scrape":"true","path":"/metrics","port":"9080"}
    Mounts:
      /etc/istio/pod from istio-podinfo (rw)
      /etc/istio/proxy from istio-envoy (rw)
      /var/lib/istio/data from istio-data (rw)
      /var/run/secrets/credential-uds from credential-socket (rw)
      /var/run/secrets/istio from istiod-ca-cert (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-m6tn4 (ro)
      /var/run/secrets/tokens from istio-token (rw)
      /var/run/secrets/workload-spiffe-credentials from workload-certs (rw)
      /var/run/secrets/workload-spiffe-uds from workload-socket (rw)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  workload-socket:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  credential-socket:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  workload-certs:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  istio-envoy:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  istio-data:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  istio-podinfo:
    Type:  DownwardAPI (a volume populated by information about the pod)
    Items:
      metadata.labels -> labels
      metadata.annotations -> annotations
  istio-token:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  43200
  istiod-ca-cert:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      istio-ca-root-cert
    Optional:  false
  tmp:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kube-api-access-m6tn4:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  3m56s                default-scheduler  Successfully assigned default/productpage-v1-6c4f8467b9-6vhzf to kind-control-plane
  Normal   Pulled     3m53s                kubelet            Container image "docker.io/istio/proxyv2:1.19.0" already present on machine
  Normal   Created    3m53s                kubelet            Created container istio-init
  Normal   Started    3m53s                kubelet            Started container istio-init
  Normal   Pulling    3m52s                kubelet            Pulling image "docker.io/istio/examples-bookinfo-productpage-v1:1.18.0"
  Normal   Pulled     109s                 kubelet            Successfully pulled image "docker.io/istio/examples-bookinfo-productpage-v1:1.18.0" in 2m3.499948217s
  Normal   Created    109s                 kubelet            Created container productpage
  Normal   Started    108s                 kubelet            Started container productpage
  Normal   Pulled     108s                 kubelet            Container image "docker.io/istio/proxyv2:1.19.0" already present on machine
  Normal   Created    108s                 kubelet            Created container istio-proxy
  Normal   Started    108s                 kubelet            Started container istio-proxy
  Warning  Unhealthy  106s (x3 over 107s)  kubelet            Readiness probe failed: Get "http://10.244.0.20:15021/healthz/ready": dial tcp 10.244.0.20:15021: connect: connection refused

# Check: pod can connect to the app via service productpage:9080
$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>

# Expose app via Gateway + VirtualService
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created

$ kubectl get service,gateway,virtualservice
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/details           ClusterIP   10.96.113.45    <none>        9080/TCP            5m38s
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP             14m
service/loki              ClusterIP   10.96.36.222    <none>        3100/TCP,9095/TCP   7m54s
service/loki-memberlist   ClusterIP   None            <none>        7946/TCP            7m54s
service/productpage       ClusterIP   10.96.108.103   <none>        9080/TCP            5m37s
service/ratings           ClusterIP   10.96.155.255   <none>        9080/TCP            5m38s
service/reviews           ClusterIP   10.96.254.71    <none>        9080/TCP            5m37s

NAME                                           AGE
gateway.networking.istio.io/bookinfo-gateway   30s

NAME                                          GATEWAYS               HOSTS   AGE
virtualservice.networking.istio.io/bookinfo   ["bookinfo-gateway"]   ["*"]   30s

$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                      AGE
grafana                ClusterIP      10.96.176.80    <none>           3000/TCP                                                                     14m
istio-egressgateway    ClusterIP      10.96.219.200   <none>           80/TCP,443/TCP                                                               15m
istio-ingressgateway   LoadBalancer   10.96.72.82     172.17.255.200   15021:32689/TCP,80:31672/TCP,443:30562/TCP,31400:32531/TCP,15443:30130/TCP   15m
istiod                 ClusterIP      10.96.10.68     <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        16m
jaeger-collector       ClusterIP      10.96.91.40     <none>           14268/TCP,14250/TCP,9411/TCP,4317/TCP,4318/TCP                               14m
kiali                  ClusterIP      10.96.116.220   <none>           20001/TCP,9090/TCP                                                           14m
loki                   ClusterIP      10.96.40.113    <none>           3100/TCP,9095/TCP                                                            14m
loki-headless          ClusterIP      None            <none>           3100/TCP                                                                     14m
loki-memberlist        ClusterIP      None            <none>           7946/TCP                                                                     14m
prometheus             ClusterIP      10.96.26.165    <none>           9090/TCP                                                                     14m
tracing                ClusterIP      10.96.176.197   <none>           80/TCP,16685/TCP                                                             14m
zipkin                 ClusterIP      10.96.36.20     <none>           9411/TCP                                                                     14m


# Check: from outside the cluster (from VM), we can connect to the app http://...../productpage
# via istio-ingressgateway > Gateway > VirtualService > Service > Deployment > Pod
$ curl -s http://istioigw/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>

# Take note of the following ip: <istio-ingressgateway_ExternalIp>, should be 172.17.255.200
$ getent hosts istioigw
172.17.255.200  istioigw



```

And then in your laptop open chrome tab to the application webpage http://172.17.1.200/productpage
and the web-application should load in the laptop


<img src="pictures/kind-istio-bookinfo-productpage.png?raw=true" width="1000">


## Istio demo

### Generate traffic
- leave shell open running:
```
while :; do echo -n "$(date)  "; curl -sv http://istioigw/productpage  2>&1 | grep '< HTTP' ; sleep 0.5; done
```

### Open Istio dashboards

#### Kiali
- in Laptop leave a shell open running: `kubectl proxy`
- In laptop, open chrome tab to http://localhost:8001/api/v1/namespaces/istio-system/services/http:kiali:20001/proxy/
  
<img src="pictures/kind-istio-kiali-bookinfo.png?raw=true" width="1000">


#### Grafana
- In laptop, leave a shell open running: `kubectl -n istio-system port-forward service/grafana 8002:3000`
- In laptop, open chrome tab to http://localhost:8002/d/3--MLVZZk/istio-control-plane-dashboard?orgId=1&refresh=5s

<img src="pictures/kind-istio-grafana.png?raw=true" width="1000">


#### Jaeger
- In laptop, open chrome tab to http://localhost:8001/api/v1/namespaces/istio-system/services/http:tracing:80/proxy/jaeger/search

<img src="pictures/kind-istio-jaeger.png?raw=true" width="1000">

## Understand core resources and concepts

- Understand sidecar auto-injection via namespace label (and exclusion)
  - New pods get sidecar
  - Existing pods get sidecar on restart

```
kubectl get namespaces --label-columns=istio-injection
kubectl label namespace default  istio-injection=enabled
```

- Understand Gateway and VirtualService, etc

The VirtualService instructs the Ingress Gateway how to route the requests that were allowed into the cluster.
Note: When we apply this resource (and actually all Istio CRD resources) the Kubernetes API Server creates an event received by Istio’s Control Plane which then applies the new configuration to the envoys (istio proxies, sidecar proxies) of every pod. And the Ingress Gateway controller is another Envoy which is configured by the Control Plane, visually presented 


<img src="pictures/Configuring-Routing-for-the-Ingress-Gateway.png?raw=true" width="1000">

```
#
# 1 nginx-controller + N Ingress + M Services = 1 Gateway + N VirtualService + M Services
# https://rinormaloku.com/istio-practice-routing-virtualservices/
#
#   . ingressgateway  -  Gateway - VirtualService - [DestinationRule: subsets]     - Services - Deployment  -  Pod
#     lb: external-ip
#                        tcp/80
#                        hosts     hosts
#                                  paths
#                                 [weights/subsets]
#                                                  [subsets (host:mySvc;version:v1]                             app: myApp
#                                                                                                               [version: v1/2]
#
#    *virtualservices* glues *gateway* with *services*
#    *destinationrule* defines *subsets* (from pod-label version:) which can be used by virtualServices
#
# Resume gw, virtualservice, destinationRule: 
# https://medium.com/google-cloud/istio-routing-basics-14feab3c040e
```


- Understand app resources

```
istio-ingressgateway (lb) 

Gateway:  bookinfo-gateway 
  - host, port 

VirtualService:  bookinfo
  - host, path , [weight/subset]

[DestinationRule: bookinfo  (declare subsets v1/v2/v3) ]
  - subsets, host, version

Service: productpage, reviews, details, rating  

Deployment: productpage-v1, reviews-v1, reviews-v2, reviews-v3, details-v1, ratings-v1
```

- Show Traffic shifting: Weight-based routing
  - Add DestinationRules to define subsets, and add VirtualService to route traffic based on weights to each subset
  - Wait 2min to let traffic-average to converge
  - Kiali: review traffic distribution % on service "review"
```
$ kubectl apply -f - <<EOT
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 60
    - destination:
        host: reviews
        subset: v2
      weight: 30
    - destination:
        host: reviews
        subset: v3
      weight: 10
EOT

destinationrule.networking.istio.io/reviews created
virtualservice.networking.istio.io/reviews created


```

- Tour Kiali dashboard

<img src="pictures/kind-istio-kiali-bookinfo-traffic-shifting.png?raw=true" width="1000">


- Tour Grafana dashboard (understand underlying prometheus)

- Tour Jaeger distributed-tracing

- Understand mTLS (intra-mesh, can be mandatory)


## Istio Lab Teardown (destroy kind cluster)

```
kind delete cluster
```

Ref:
- https://rinormaloku.com/summary/
- https://medium.com/google-cloud/istio-routing-basics-14feab3c040e
- mTLS: https://istio.io/latest/docs/tasks/security/authentication/authn-policy/ & https://github.com/pydevops/istio-lab

[Credits](https://github.com/zipizap/IstioDemo)
