# Istio service mesh in ambient mode tutorial
> Note: This tutorial is based on kind running on an ubuntu 20.04 machine with docker. 
> It should work on any other linux distribution. We also tried to run it on a 
> macbook pro with docker desktop, this did not work yet. To see the current status 
> of all supported environments please look [here](https://github.com/istio/istio/tree/experimental-ambient#supported-environments).

This is a tutorial for the istio service mesh in ambient mode. In this 
tutorial we want to show two use cases with the use of istio ambient:

* ### L4 and L7 network processing layer with istio ambient
* ### Deploying kiali dashboard addon on istio ambient

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

1. Install the alpha version of istioctl that contains the `ambient` profile: 
    ```bash
    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.0-alpha.0 sh -
    ```
2. Install instio in ambient mode on your local cluster
    ```bash
    ./istio-1.18.0-alpha.0/bin/istioctl install --set profile=ambient -y
    ```
    You should see the following output:
    ```bash
    ✔ Istio core installed
    ✔ Istiod installed
    ✔ Ingress gateways installed
    ✔ CNI installed
    ✔ Installation complete
    ```
3. You can also see all the istio resources with the following command:
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
    In the output you can see the 3 "ambient" proxies running, one for each node in the cluster. 
    ```bash
    pod/istio-cni-node-5vkb6                   1/1     Running   0          12m
    pod/istio-cni-node-tc8fn                   1/1     Running   0          12m
    pod/istio-cni-node-zm8dl                   1/1     Running   0          12m
    ```
    If you compare this to a service mesh that is not running in ambient mode, 
    you will see that there are no proxy nodes running. Then only a proxy is installed when a pod is deployed. In 
    such a setup only the following resources are created:
    ```bash
    pod/istio-egressgateway-5bdd756dfd-2f9cq    1/1     Running   0          82s
    pod/istio-ingressgateway-67f7b5f88d-qf8vq   1/1     Running   0          82s
    pod/istiod-58c6454c57-bvv4v                 1/1     Running   0          94s
    ```
### Deploying the demo apps

1. Apply all the deployments found in the istio samples folder
   > You need to wait until all pods are running, otherwise we can't label the 
   > pods in step 2.  
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml
   kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/bookinfo-gateway.yaml
   kubectl apply -f https://raw.githubusercontent.com/linsun/sample-apps/main/sleep/sleep.yaml
   kubectl apply -f https://raw.githubusercontent.com/linsun/sample-apps/main/sleep/notsleep.yaml
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
When enabling ambient mode you’ll immediately gain mTLS communication 
among the applications in the Ambient mesh.

To verify this you could load one of the X.509 certificates for by using one 
of your ztunnels and retrieving the certificate from it.

```bash
./istio/bin/istioctl pc secret <ztunnel_id> -n istio-system -o json | jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 --decode | openssl x509 -noout -text -in /dev/stdin
```
This should give you output along the line of:

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            31:64:37:7c:3e:16:d8:b2:93:59:6a:89:a7:85:6c:2d
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O = cluster.local
        Validity
            Not Before: Nov 30 13:15:25 2022 GMT
            Not After : Dec  1 13:17:25 2022 GMT
        Subject: 
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:af:77:83:33:14:23:02:59:00:a1:f1:2a:38:38:
                    7e:0c:8f:e1:df:1c:2b:e5:9b:92:62:d9:db:c1:20:
                    13:97:16:1c:d0:35:e7:98:4a:0b:89:ec:c7:58:ad:
                    bc:78:9a:f1:e9:67:c6:18:e2:10:06:dd:1b:55:3b:
                    bd:67:31:b4:2e:b5:07:2a:af:17:5e:ed:cd:6a:2c:
                    73:22:ad:7b:b3:c5:70:ac:af:3f:87:58:2f:e3:7c:
                    4b:36:33:11:c8:2f:cb:81:c8:c1:b6:49:47:b6:5b:
                    4f:54:29:a4:1b:d6:72:08:33:c1:9c:ab:f8:7d:f8:
                    da:15:7c:2c:ab:02:33:3c:c3:11:e3:58:58:26:16:
                    65:a1:c9:bb:7c:d6:c7:b1:32:00:bd:72:70:9f:e2:
                    57:dc:ac:ce:92:4e:36:09:74:3d:82:10:db:93:d3:
                    65:55:86:f4:ac:29:98:50:69:4e:f4:ca:5f:2f:61:
                    48:38:7b:bb:37:70:da:01:61:f3:ac:72:8c:b8:3f:
                    8c:2a:b6:50:57:e3:d0:7b:f5:65:39:01:e5:eb:96:
                    ee:09:7e:a5:6a:48:53:47:c7:52:ef:ea:80:fc:4e:
                    c9:a4:94:13:8e:ad:37:f4:12:dc:76:79:82:dd:d6:
                    6f:d2:5b:c4:1e:55:9f:65:01:13:12:d5:a3:97:ba:
                    b5:f3
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                keyid:97:67:46:44:54:66:14:EB:15:4F:B4:C7:44:56:3F:68:81:39:2D:DB

            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/default/sa/notsleep
    Signature Algorithm: sha256WithRSAEncryption
         3f:89:35:57:1b:a8:33:c5:74:cc:a0:77:27:92:fc:58:79:59:
         fa:0e:ea:75:8d:6b:11:50:4f:9b:99:c9:83:94:a4:82:bb:ee:
         56:09:e9:86:1b:1f:1d:77:1c:97:1c:f3:6b:db:2e:61:1d:e5:
         29:a8:6e:0c:61:9f:41:50:47:51:2b:ae:aa:1f:4d:04:e8:7c:
         f3:ae:b4:1d:49:b8:de:70:47:a3:7a:4d:38:ae:6c:65:8a:07:
         fa:43:aa:2e:4e:ec:6e:1d:9d:e5:fb:22:7d:7e:ec:4a:d4:37:
         a2:2e:91:b2:d4:6e:a5:1a:c9:20:82:ba:8c:a2:00:4c:0c:b0:
         51:a1:f9:be:16:13:b5:2d:cc:23:e8:c8:16:e8:46:09:7e:c1:
         eb:9b:ac:0a:db:40:02:18:86:82:c3:0e:c4:52:0a:f3:22:73:
         12:da:58:aa:c9:49:ca:bf:fe:8c:5e:50:c2:12:12:ee:10:2f:
         73:fa:6d:ce:46:61:6d:6f:36:48:0f:84:6b:cb:3a:60:11:a3:
         fc:df:49:69:74:f9:bc:63:72:9d:b8:28:b5:b1:ac:cf:85:9c:
         b9:5f:d3:16:03:41:d7:94:e9:20:d4:0c:fc:db:48:4e:68:a5:
         15:84:09:33:0b:31:22:82:10:fe:40:5e:3a:b5:e2:f9:c3:9d:
         60:64:e3:80
```

### L4 processing in ambient mode
Istio service mesh ambient supports L4 processing by default. This means that
we can already apply L4 Authorization Policies out of the box.
![L4 processing](./images/ztunnel4.png)

We can verify this by using our sleep application to communicate to other 
services within our cluster.

1. Call the product page service from within our sleep application with curl.
   ```bash
   kubectl exec deploy/sleep -- curl -s http://productpage:9080/
   ```
   > We use curl to call the productpage service and to retrieve the index 
   > page.

2. Check the logs of the ztunnels that are in the same node as the product page and sleep service.
   > Note: this requires some trial and error, because we don't know which 
   > ztunnel belongs to a specific node.
   ```bash
   kubectl  -n istio-system logs <ztunnel_id> -cistio-proxy --tail 1
   ```
   You should see in the ztunnel dedicated to the productpage service output similiar:
   ```bash
   [2022-11-29T19:43:56.477Z] "- - HTTP/2" 0 - - - "-" 241 1968 91918 - "-" "-" "-" "-" "-" - - 10.244.2.4:15008 10.244.1.2:39660 - - capture inbound listener
   ```
   You should see in the ztunnel dedicated to the sleep service output similiar to:
   ```bash
   [2022-11-29T19:43:56.476Z] "- - -" 0 - - - "-" 84 1839 32 - "-" "-" "-" "-" "envoy://outbound_tunnel_lis_spiffe://cluster.local/ns/default/sa/sleep/10.244.2.4:9080" spiffe://cluster.local/ns/default/sa/sleep_to_http_productpage.default.svc.cluster.local_outbound_internal envoy://internal_client_address/ 10.96.33.162:9080 10.244.1.4:51650 - - capture outbound (no waypoint proxy)
   ```

### L7 processing in ambient mode
To enable l7 processing, you must deploy a gateway. A gateway will make
sure that the communication between the two services is monitored, and
that custom policies are enforced, such as request type limiting, network 
routing etc.

This setup can be seen in the picture below.
![L7 processing](./images/ztunnel7.png)

You can install the gateway with the following commands:

1. Deploy a gateway with the following definition:   
   > Note that the gatewayClassName in the gateway resource must be set 
   > to ‘istio-mesh’, otherwise Istio won’t create the corresponding 
   > waypoint proxy for the productpage.
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: gateway.networking.k8s.io/v1alpha2
   kind: Gateway
   metadata:
    name: productpage
    annotations:
      istio.io/service-account: bookinfo-productpage
   spec:
    gatewayClassName: istio-mesh
   EOF
   ```
2. You can check if the deployment of the productpage waypoint was successfull 
   by running the following command:
   ```bash
   kubectl get pod | grep waypoint
   ```
   Your output should be similar to:
   ```bash
   bookinfo-productpage-waypoint-proxy-fcf74c55d-j9zm5   1/1     Running   0          27s
   ```
   
3. Call the product page service from within our sleep application with curl.
   ```bash
   kubectl exec deploy/sleep -- curl -s http://productpage:9080/
   ```
   > We use curl to call the productpage service and to retrieve the index 
   > page.

4. Check the logs of the ztunnels that are in the same node as the product page and sleep service.
   > Note: this requires some trial and error, because we don't know which 
   > ztunnel belongs to a specific node.
   ```bash
   kubectl  -n istio-system logs <ztunnel_id> -cistio-proxy --tail 1
   ```
   You should see in the ztunnel dedicated to the productpage service output similiar:
   ```bash
   [2022-11-29T20:18:40.054Z] "- - HTTP/2" 0 - - - "-" 865 1977 90640 - "-" "-" "-" "-" "-" - - 10.244.2.4:15008 10.244.1.11:37182 - - capture inbound listener
   ```
   You should see in the ztunnel dedicated to the sleep service output similiar to:
   ```bash
   [2022-11-29T20:18:40.046Z] "- - -" 0 - - - "-" 84 1894 43 - "-" "-" "-" "-" "10.244.1.11:15006" spiffe://cluster.local/ns/default/sa/sleep_to_server_waypoint_proxy_spiffe://cluster.local/ns/default/sa/bookinfo-productpage 10.244.1.4:36849 10.96.33.162:9080 10.244.1.4:51652 - - capture outbound (to server waypoint proxy)
   ```
   If you look at the ztunnel dedicated to the sleep service, you can read that the 
   request will first go to a server waypoint proxy just as shown in the picture above.

### Installing a service mesh addon

#### Kiali dashboard 
> Note: ambient mode can cause issues with kiali

To deploy kiali dashboard on ambient mode, you can go to you istio installation
en install the plugim. You can do this by running the following commands.

1. Install the kiali dashboard by running the following command:
   ```bash
   kubectl apply -f ./istio/samples/addons
   kubectl rollout status deployment/kiali -n istio-system
   ```
2. Access the dashboard kiali dashboard in your browser by running the following command:
   ```bash
   ./istio/bin/istioctl dashboard kiali
   ```
   > It could be that the graph does not show anything. This means that your cluster 
   > did not handle any request in the past minute. Follow the next step to make
   > the traffic flow visible.
   In your graph view you can see an overview of all your services on the cluster and how 
   traffic flows between services.

3. Call the productpage service with your sleep service
   ```bash
   kubectl exec deploy/sleep -- curl -s http://productpage:9080/
   ```
   
4. Go back to your graph dashboard and see the traffic flow.



