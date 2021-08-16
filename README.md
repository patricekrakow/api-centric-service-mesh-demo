# Demo of an API-Centric Server Mesh

## API

...

## Docker & Kubernetes

### Docker

Let's first verify that docker is installed:

```text
$ docker --version
Docker version 19.03.13, build 4484c46d9d
```

Then, let's start the image `kennethreitz/httpbin` within a container:

```text
$ docker run -p 80:80 kennethreitz/httpbin
controlplane $ docker run -p 80:80 kennethreitz/httpbin
Unable to find image 'kennethreitz/httpbin:latest' locally
latest: Pulling from kennethreitz/httpbin
473ede7ed136: Pull complete 
c46b5fa4d940: Pull complete 
93ae3df89c92: Pull complete 
6b1eed27cade: Pull complete 
0373952b589d: Pull complete 
7b82cd0ee527: Pull complete 
a36b2d884a89: Pull complete 
Digest: sha256:599fe5e5073102dbb0ee3dbb65f049dab44fa9fc251f6835c9990f8fb196a72b
Status: Downloaded newer image for kennethreitz/httpbin:latest
[2021-08-13 14:20:32 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2021-08-13 14:20:32 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2021-08-13 14:20:32 +0000] [1] [INFO] Using worker: gevent
[2021-08-13 14:20:32 +0000] [9] [INFO] Booting worker with pid: 9
```

Then, within a new terminal you can test our API endpoint with `curl`:

```text
$ curl localhost/uuid
{
  "uuid": "acc28df9-959e-43e9-adc5-86a2ff5fcfaa"
}
$ curl localhost/uuid
{
  "uuid": "9246b887-2ae9-4ffc-80c8-ca58dd98d411"
}
```

Notice that we are now using `localhost` as a hostname, and not `httpbin.org` anymore as we are using the **internal** implementation. We will come back on this important point later.

### Kubernetes

## Istio

### Install Istio

1\. Download and install the latest release of the `istioctl` with `curl`:

```text
$ curl -sL https://istio.io/downloadIstioctl | sh -
Downloading istioctl-1.11.0 from https://github.com/istio/istio/releases/download/1.11.0/istioctl-1.11.0-linux-amd64.tar.gz ...
istioctl-1.11.0-linux-amd64.tar.gz download complete!

Add the istioctl to your path with:
  export PATH=$PATH:$HOME/.istioctl/bin 

Begin the Istio pre-installation check by running:
         istioctl x precheck 

Need more information? Visit https://istio.io/docs/reference/commands/istioctl/
```

2\. Add the istioctl client to your path and verify the installation:

```text
$ export PATH=$PATH:$HOME/.istioctl/bin
$ istioctl version
no running Istio pods in "istio-system"
1.11.0
```

3\. Deploy the Istio operator:

```text
$ istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.11.0
Operator controller will watch namespaces: istio-system
✔ Istio operator installed
✔ Installation complete
```

4.\ Verify the installation of the Istio operator:

```text
$ kubectl get pods -n istio-operator
NAME                              READY   STATUS    RESTARTS   AGE
istio-operator-6577b77679-hrrwg   1/1     Running   0          112s
```

4\. Create a file `one-istio-operator.yaml` to create an Istio operator resource instance:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: one-istio-operator
spec:
  profile: default
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
```

5\. Create the `one-istio-operator` IstioOperator from the `one-istio-operator.yaml` file:

```text
$ kubectl apply -f one-istio-operator.yaml
istiooperator.install.istio.io/one-istio-operator created
```

6\. Verify the installation of Istio:

```text
$ kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
istio-ingressgateway-8565f9474-cksjs   1/1     Running   0          13s
istiod-6c565d48b8-htpmm                1/1     Running   0          25s
```
