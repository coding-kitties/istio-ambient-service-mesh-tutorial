# Kubernetes ambient service mesh with Istio

> Note: This tutorial is based on kind running on an ubuntu 20.04 machine with docker. 
> It should work on any other linux distribution. We also tried to run it on a 
> macbook pro with docker desktop, this did not work yet. To see the current status 
> of all supported environments please look [here](https://github.com/istio/istio/tree/experimental-ambient#supported-environments).

## Requirements
* kubectl
* kind: with a local cluster running with configuration similiar to:
   ```bash
   kind create cluster --config=- <<EOF
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   name: ambient
   nodes:
   - role: control-plane
   - role: worker
   - role: worker
   EOF
   ```
* docker

## Getting started
To get started with the tutorial, follow the following steps:

### Installation

1. Download the experimental version of istioctl 
    ```bash
    curl https://storage.googleapis.com/istio-build/dev/0.0.0-ambient.191fe680b52c1754ee72a06b3e0d3f9d116f2e82/istio-0.0.0-ambient.191fe680b52c1754ee72a06b3e0d3f9d116f2e82-linux-amd64.tar.gz --output istio.tar.gz
    ```
2. untar the file
    ```bash
    tar -xvf istio.tar.gz
    rm istio.tar.gz
    ```
3. Move the istioctl binary to the istio folder
    ```bash
    mkdir istio
    mv istio*/*  istio
    ```
4. Install instio in ambient mode on your local cluster
    ```bash
    ./istio/bin/istioctl install --set profile=ambient -y
    ```
    You should see the following output:
    ```bash
    ✔ Istio core installed
    ✔ Istiod installed
    ✔ Ingress gateways installed
    ✔ CNI installed
    ✔ Installation complete
    ```
5. You can also see all the istio resources with the following command:
    ```bash
    kubectl -n istio-system get all
    ```
    You should see output along the line of:
    ```bash
    NAME                                       READY   STATUS    RESTARTS   AGE
    pod/istio-cni-node-5vkb6                   1/1     Running   0          12m
    pod/istio-cni-node-tc8fn                   1/1     Running   0          12m
    pod/istio-cni-node-zm8dl                   1/1     Running   0          12m
    pod/istio-ingressgateway-dd667dbb7-z64jh   1/1     Running   0          12m
    pod/istiod-6f9c757686-kw5z4                1/1     Running   0          14m
    pod/ztunnel-5qpn7                          1/1     Running   0          14m
    pod/ztunnel-fccxw                          1/1     Running   0          14m
    pod/ztunnel-mq72n                          1/1     Running   0          14m
   
    NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
    service/istio-ingressgateway   LoadBalancer   10.96.15.172   <pending>     15021:32651/TCP,80:32654/TCP,443:30106/TCP   12m
    service/istiod                 ClusterIP      10.96.83.33    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        14m
   
    NAME                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    daemonset.apps/istio-cni-node   3         3         3       3            3           kubernetes.io/os=linux   12m
    daemonset.apps/ztunnel          3         3         3       3            3           <none>                   14m
   
    NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/istio-ingressgateway   1/1     1            1           12m
    deployment.apps/istiod                 1/1     1            1           14m
   
    NAME                                             DESIRED   CURRENT   READY   AGE
    replicaset.apps/istio-ingressgateway-dd667dbb7   1         1         1       12m
    replicaset.apps/istiod-6f9c757686                1         1         1       14m
   
    NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
    horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          12m
    horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 <unknown>/80%   1         5         1          14m
    ```

### Deploying the demo apps

1. Apply all the deployments found in the istio samples folder
   ```bash
   kubectl apply -f ./istio/samples/bookinfo/platform/kube/bookinfo.yaml
   kubectl apply -f ./istio/samples/bookinfo/networking/bookinfo-gateway.yaml
   kubectl apply -f https://zhaohuabing.com/download/ambient/sleep.yaml
   kubectl apply -f https://zhaohuabing.com/download/ambient/notsleep.yaml
   ```

2. Put all the apps in ambient mode
   ```bash
   kubectl label namespace default istio.io/dataplane-mode=ambient
   ```
   
3. If deployments are correct you should be able to see one of the node "ambient" proxies logs:
   ```bash
   kubectl logs <istio-cni-node-id> -n istio-system|grep route
   ```
   You should see output along the line of:
   ```bash
   2022-11-26T19:44:13.500330Z     info    ambient Adding route for productpage-v1-7c548b785b-thqvj/default: [table 100 10.244.2.9/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   2022-11-26T19:44:13.507981Z     info    ambient Adding route for details-v1-76778d6644-fnqpt/default: [table 100 10.244.2.4/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   2022-11-26T19:44:13.513244Z     info    ambient Adding route for ratings-v1-85c74b6cb4-njpsd/default: [table 100 10.244.2.5/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   2022-11-26T19:44:13.519424Z     info    ambient Adding route for reviews-v1-6494d87c7b-vl9qf/default: [table 100 10.244.2.6/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   2022-11-26T19:44:13.525845Z     info    ambient Adding route for reviews-v2-79857b95b-nkqcz/default: [table 100 10.244.2.7/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   2022-11-26T19:44:13.531266Z     info    ambient Adding route for reviews-v3-75f494fccb-27bj5/default: [table 100 10.244.2.8/32 via 192.168.126.2 dev istioin src 10.244.2.1]
   ```