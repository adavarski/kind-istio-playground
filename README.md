# Kind Istio Playground

Isto playground in a kind-cluster

Pre:

- Docker
- Kind
- kubectl


# Create kubernetes cluster with kind (over docker) and metallb
```
kind create cluster
kubectl get pod -A

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
LB_NET_PREFIX=$(docker network inspect -f '{{range .IPAM.Config }}{{ .Gateway }}{{end}}' kind | cut -d '.' -f1,2)
LB_NET_IP_FIRST=${LB_NET_PREFIX}.255.200
LB_NET_IP_LAST=${LB_NET_PREFIX}.255.250
kubectl apply -f - <<EOT
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

## Install istio and addons

```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.19
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y

# Enable namespace-auto-injection for default ns
kubectl label namespace default  istio-injection=enabled
kubectl get namespaces --label-columns=istio-injection

# Install addons: kiali, prometheus, etc
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
ISTIOINGRESSGW_LB_IP=$(kubectl -n istio-system get svc/istio-ingressgateway -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$ISTIOINGRESSGW_LB_IP istioigw" | sudo tee -a /etc/hosts
```

## Install an app prepared for istio

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Check app services, and pods with auto-injected sidecar 2/2
kubectl get services
timeout 60 watch kubectl get pods
#kubectl describe pod pod-with-sidecar

# Check: pod can connect to the app via service productpage:9080
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# Expose app via Gateway + VirtualService
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get service,gateway,virtualservice

# Check: from outside the cluster (from VM), we can connect to the app http://...../productpage
# via istio-ingressgateway > Gateway > VirtualService > Service > Deployment > Pod
curl -s http://istioigw/productpage | grep -o "<title>.*</title>"

# Take note of the following ip: <istio-ingressgateway_ExternalIp>, should be 172.18.255.200
getent hosts istioigw

```

And then in your laptop open chrome tab to the application webpage http://127.0.0.1:20080/productpage
and the web-application should load in the laptop



# Istio demo

## Generate traffic
- In VM, leave shell open running:
```
while :; do echo -n "$(date)  "; curl -sv http://istioigw/productpage  2>&1 | grep '< HTTP' ; sleep 0.5; done
```

## Open Istio dashboards

### Kiali
- In VM, leave a shell open running: `kubectl proxy`
- In laptop ssh-session, add Local-port-forwarding-tunnel:
  - source-port:          8001
  - destination ip:port   127.0.0.1:8001
- In laptop, open chrome tab to http://localhost:8001/api/v1/namespaces/istio-system/services/http:kiali:20001/proxy/


### Grafana
- In VM, leave a shell open running: `kubectl -n istio-system port-forward service/grafana 8002:3000`
- In laptop, open chrome tab to http://localhost:8002/d/G8wLrJIZk/istio-mesh-dashboard?orgId=1&refresh=5s&from=now-1h&to=now

### Jaeger
- In laptop, open chrome tab to http://localhost:8001/api/v1/namespaces/istio-system/services/http:tracing:80/proxy/jaeger/search



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
kubectl apply -f - <<EOT
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
```


- Tour Kiali dashboard

- Tour Grafana dashboard (understand underlying prometheus)

- Tour Jaeger distributed-tracing

- Understand mTLS (intra-mesh, can be mandatory)


# Istio Lab Teardown (destroy kind cluster)


```
kind cluster delete
```

