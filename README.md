# Demo of an API-Centric Server Mesh

## Prerequisites

You need a working environment with a connection to a  Kubernetes cluster, or you can use our [Katacoda Kubernetes Playground](https://www.katacoda.com/patrice1972/scenarios/kubernetes-hello-world).

Verify that Kubernetes is running properly:

```text
$ kubectl version --short
Client Version: v1.18.0
Server Version: v1.18.0
```

Then, download all the files needed for this demo from GitHub:

```text
$ git clone https://github.com/patricekrakow/api-centric-service-mesh-demo.git
Cloning into 'api-centric-service-mesh-demo'...
remote: Enumerating objects: 49, done.
remote: Counting objects: 100% (49/49), done.
remote: Compressing objects: 100% (49/49), done.
remote: Total 49 (delta 26), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (49/49), done.
$ cd api-centric-service-mesh-demo/
```

You are ready for our **API-Centric Server Mesh** demo!

## API

Let's start by desiging an awesome API that allows the creation of [**U**niversally **U**nique **ID**entifiers (UUIDs)](https://en.wikipedia.org/wiki/Universally_unique_identifier).

### Noun(s)

`/uuid`

### Verb(s)

`GET /uuid`

### Hostname

`GET httpbin.org/uuid`

### OpenAPI

```yaml
swagger: '2.0'
info:
  title: Awesome UUID API
  description: This awesome API allows you to create a UUID (Universally Unique IDentifier) version 4.
  contact:
    name: Patrice Krakow
    email: patrice.krakow@ing.com
  version: 1.0.0
host: httpbin.org
basePath: /
schemes:
  - http
consumes:
  - application/json
produces:
  - application/json
paths:
  /uuid:
    get:
      summary: Return a UUID4.
      responses:
        '200':
          description: OK
          schema:
            $ref: '#/definitions/UUID'
definitions:
  UUID:
    type: object
    required:
      - uuid
    properties:
      uuid:
        type: string
```

This file is avaible from GitHub using the address <https://raw.githubusercontent.com/patricekrakow/api-centric-service-mesh-demo/main/Awesome-UUID-API.v1.0.0.oas2.0.yaml>, you can use this URL to test online tools like [ReDoc](https://redocly.github.io/redoc/) or [Swagger UI](https://petstore.swagger.io/).

### Bezos Mandate

_5. All service interfaces, without exception, must be designed from the ground up to be externalizable. That is to say, the team must plan and design to be able to expose the interface to developers in the outside world. No exceptions._

```text
$ curl httpbin.org/uuid
{
  "uuid": "51b517aa-e8eb-442c-8aae-2b93e771916a"
}
```

## Docker & Kubernetes

But, before this API was live on the Internet, we had to implement it on our laptop... Let's go back to that stage.

### Docker

Thanks to Docker, we will speed up the implementation phase ;-)

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

Let's now use Kubernetes, so we can orchestrate containers for the server side, the client side, and much more...

```text
$ kubectl apply -f httpbin.1st.yaml
namespace/httpbin-ns created
serviceaccount/httpbin-sa created
deployment.apps/httbin-deploy created
```

```text
$ kubectl apply -f curl.1st.yaml
namespace/curl-ns created
serviceaccount/curl-sa created
deployment.apps/curl-deploy created
```

```text
$ kubectl apply -f httpbin.2nd.yaml
namespace/httpbin-ns unchanged
serviceaccount/httpbin-sa unchanged
deployment.apps/httbin-deploy unchanged
service/httpbin created
```

```text
$ kubectl apply -f curl.2nd.yaml
namespace/curl-ns unchanged
serviceaccount/curl-sa unchanged
deployment.apps/curl-deploy configured
```

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

> It can take some time after the creation of the Istio operatror, for the `istiod` and `istio-ingressgateway` to be created.

#### Deploy the Envoy sidecar proxies

```text
$ kubectl apply -f httpbin.3rd.yaml
namespace/httpbin-ns configured
serviceaccount/httpbin-sa unchanged
deployment.apps/httbin-deploy unchanged
service/httpbin unchanged
```

```text
$ kubectl apply -f curl.3rd.yaml
namespace/curl-ns configured
serviceaccount/curl-sa unchanged
deployment.apps/curl-deploy unchanged
```

```text
$ kubectl delete pods httbin-deploy-6bf8fcb5c5-k6kmn -n httpbin-ns
pod "httbin-deploy-6bf8fcb5c5-k6kmn" deleted

$ kubectl delete pods curl-deploy-758c5bf7f7-xcsch -n curl-ns
pod "curl-deploy-758c5bf7f7-xcsch" deleted
```

#### Install and Congigure Prometheus

```text
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/addons/prometheus.yaml
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

#### Install and Congigure Kiali

```text
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/addons/kiali.yaml
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
```

```text
export INGRESS_DOMAIN=2886795341-80-ollie02.environments.katacoda.com
```

```text
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kiali-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-kiali
      protocol: HTTP
    hosts:
    - "kiali.${INGRESS_DOMAIN}"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-vs
  namespace: istio-system
spec:
  hosts:
  - "kiali.${INGRESS_DOMAIN}"
  gateways:
  - kiali-gateway
  http:
  - route:
    - destination:
        host: kiali
        port:
          number: 20001
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: kiali
  namespace: istio-system
spec:
  host: kiali
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF
gateway.networking.istio.io/kiali-gateway created
virtualservice.networking.istio.io/kiali-vs created
destinationrule.networking.istio.io/kiali created
```

```text
echo http://kiali.${INGRESS_DOMAIN}
```

## API Centric Service Mesh

```text
$ kubectl apply -f mesh.A.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry created
virtualservice.networking.istio.io/httpbin-org-virtual-service created
```

```text
$ kubectl apply -f curl.API.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry created
virtualservice.networking.istio.io/httpbin-org-virtual-service created
```

```text
$ kubectl apply -f node-uuid.yaml
...
```

```text
$ kubectl apply -f mesh.A.80-20.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry unchanged
virtualservice.networking.istio.io/httpbin-org-virtual-service configured
```

```text
$ kubectl apply -f mesh.A.50-50.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry unchanged
virtualservice.networking.istio.io/httpbin-org-virtual-service configured
```

```text
$ kubectl apply -f mesh.A.20-80.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry unchanged
virtualservice.networking.istio.io/httpbin-org-virtual-service configured
```

```text
$ kubectl apply -f mesh.B.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry created
virtualservice.networking.istio.io/httpbin-org-virtual-service created
```