# Kubernetes ambient service mesh with Istio

> Note: This tutorial is based on kind running on an ubuntu 20.04 machine with docker. 
> It should work on any other linux distribution. We also tried to run it on a 
> macbook pro with docker desktop, this did not work yet.

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
   