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
$ ls
Awesome-UUID-API.v1.0.0.oas2.0.yaml  httpbin.2nd.yaml   mesh.A.yaml
curl.1st.yaml                        httpbin.3rd.yaml   mesh.B.yaml
curl.2nd.yaml                        LICENSE            node-uuid.yaml
curl.3rd.yaml                        mesh.A.20-80.yaml  one-istio-operator.yaml
curl.API.yaml                        mesh.A.50-50.yaml  README.md
httpbin.1st.yaml                     mesh.A.80-20.yaml
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

Then, within a new terminal you can test our local API endpoint with `curl`:

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

Notice that we are now using `localhost` as a hostname, and not `httpbin.org` anymore as we are using it **internally**. We will come back on this important point later.

### Kubernetes

Let's now use Kubernetes, so we can orchestrate containers for the server side, the client side, and much more...

Let's first configure the server side:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: httpbin-ns
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin-sa
  namespace: httpbin-ns
```  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httbin-deploy
  namespace: httpbin-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: 1.0.0
  template:
    metadata:
      labels:
        app: httpbin
        version: 1.0.0
    spec:
      serviceAccountName: httpbin-sa
      containers:
      - name: httpbin-co
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
```

We can configure all that in one go using the `httpbin.1st.yaml` file with the following command:

```text
$ kubectl apply -f httpbin.1st.yaml
namespace/httpbin-ns created
serviceaccount/httpbin-sa created
deployment.apps/httbin-deploy created
```

```text
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
httpbin-ns    httbin-deploy-6bf8fcb5c5-nl4jn             1/1     Running   0          39s
...
```

```text
$ kubectl logs -n httpbin-ns httbin-deploy-6bf8fcb5c5-nl4jn 
[2021-08-18 08:32:46 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2021-08-18 08:32:46 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2021-08-18 08:32:46 +0000] [1] [INFO] Using worker: gevent
[2021-08-18 08:32:46 +0000] [8] [INFO] Booting worker with pid: 8
```

```text
$ kubectl describe pod -n httpbin-ns httbin-deploy-6bf8fcb5c5-nl4jn
Name:         httbin-deploy-6bf8fcb5c5-nl4jn
Namespace:    httpbin-ns
...
Status:       Running
IP:           10.244.1.3
...
Events:
  ...
  Normal  Started    3m4s   kubelet, node01    Started container httpbin-co
```

Let's now start the client side:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: curl-ns
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: curl-sa
  namespace: curl-ns
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deploy
  namespace: curl-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
      version: 1.0.0
  template:
    metadata:
      labels:
        app: curl
        version: 1.0.0
    spec:
      serviceAccountName: curl-sa
      containers:
      - name: curl-co
        image: curlimages/curl
        command:
        - /bin/sh
        - -c
        - sleep 2h
```

We can configure all that in one go using the `curl.1st.yaml` file with the following command:

```text
$ kubectl apply -f curl.1st.yaml
namespace/curl-ns created
serviceaccount/curl-sa created
deployment.apps/curl-deploy created
```

```text
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS             RESTARTS   AGE
curl-ns       curl-deploy-549648b7d6-kszw8               1/1     Running            0          11s
httpbin-ns    httbin-deploy-6bf8fcb5c5-nl4jn             1/1     Running            0          10m
...
```

```text
$ kubectl exec -it -n curl-ns curl-deploy-549648b7d6-kszw8 -- /bin/sh 
/ $
```

```text
/ $ curl http://10.244.1.3/uuid
{
  "uuid": "231a1b2a-2c77-48fc-bec9-6db3fb2e3e8f"
} 
```

```text
/ $ while curl http://10.244.1.3/uuid; do sleep 1.0; done;
{
  "uuid": "c0d668e8-de91-413d-b1cf-ab457dc2204d"
}
{
  "uuid": "6302c15b-8eed-4133-8e21-fbc7a59b686b"
}
{
  "uuid": "9c2b4f86-9aad-474d-8e5e-a33a2f4ecea8"
}
...
```

Using IP address is not really convenient and is even an issue if the _Pod_ crashes as the _ReplicaSet_ will create a new one with a different IP address. That's why you have _Service_ in Kubernetes which will create a hostname for the _Pod_ and automatically configure and update the internal DNS service of the Kubernetes cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin-ns
spec:
  selector:
    app: httpbin
    version: 1.0.0
  ports:
  - name: httpbin-http-port
    protocol: TCP
    port: 80
    targetPort: 80
```

We can configure that using the `httpbin.2nd.yaml` file with the following command:

```text
$ kubectl apply -f httpbin.2nd.yaml
namespace/httpbin-ns unchanged
serviceaccount/httpbin-sa unchanged
deployment.apps/httbin-deploy unchanged
service/httpbin created
```

```text
/ $ curl http://httpbin.httpbin-ns/uuid
{
  "uuid": "3327448a-0a2c-4455-bae4-8950f41ddf55"
}
```

Finally, let's change the configuration of `curl-co` container in order to fire 5 requests/second:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deploy
  namespace: curl-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
      version: 1.0.0
  template:
    metadata:
      labels:
        app: curl
        version: 1.0.0
    spec:
      serviceAccountName: curl-sa
      containers:
      - name: curl-co
        image: curlimages/curl
        command:
        - /bin/sh
        - -c
        - while curl --no-progress-meter http://httpbin.httpbin-ns/uuid; do sleep 0.2; done;
```

We can configure that using the `curl.2nd.yaml` file with the following command:

```text
$ kubectl apply -f curl.2nd.yaml
namespace/curl-ns unchanged
serviceaccount/curl-sa unchanged
deployment.apps/curl-deploy configured
```

```text
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS             RESTARTS   AGE
curl-ns       curl-deploy-549648b7d6-kszw8               1/1     Terminating        0          15m
curl-ns       curl-deploy-64d9c774c6-zs2jf               1/1     Running            0          9s
httpbin-ns    httbin-deploy-6bf8fcb5c5-nl4jn             1/1     Running            0          26m
...
```

```text
$ kubectl logs -n curl-ns --follow curl-deploy-64d9c774c6-zs2jf 
...
{
  "uuid": "ecadb5d5-ecbe-42f5-b74c-074f9586c665"
}
{
  "uuid": "98d53ac4-b1ef-4090-a0fa-9d76facd1f3d"
}
...
```

## Istio

### Install Istio

1\. Download and install the latest release of the `istioctl` with `curl`:

```text
curl -sL https://istio.io/downloadIstioctl | sh -
```

```text
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

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: httpbin-ns
  labels:
    istio-injection: enabled
```

We can configure that using the `httpbin.3rd.yaml` file with the following command:

```text
$ kubectl apply -f httpbin.3rd.yaml
namespace/httpbin-ns configured
serviceaccount/httpbin-sa unchanged
deployment.apps/httbin-deploy unchanged
service/httpbin unchanged
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: curl-ns
  labels:
    istio-injection: enabled
```

We can configure that using the `curl.3rd.yaml` file with the following command:

```text
$ kubectl apply -f curl.3rd.yaml
namespace/curl-ns configured
serviceaccount/curl-sa unchanged
deployment.apps/curl-deploy unchanged
```

```text
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS             RESTARTS   AGE
curl-ns       curl-deploy-549648b7d6-kszw8               1/1     Terminating        0          15m
curl-ns       curl-deploy-64d9c774c6-zs2jf               1/1     Running            0          9s
httpbin-ns    httbin-deploy-6bf8fcb5c5-nl4jn             1/1     Running            0          26m
...
```

```text
$ kubectl delete pods -n httpbin-ns httbin-deploy-6bf8fcb5c5-k6kmn 
pod "httbin-deploy-6bf8fcb5c5-k6kmn" deleted

$ kubectl delete pods -n curl-ns curl-deploy-758c5bf7f7-xcsch 
pod "curl-deploy-758c5bf7f7-xcsch" deleted
```

```text
$ kubectl get pods -A
NAMESPACE        NAME                                       READY   STATUS             RESTARTS   AGE
curl-ns          curl-deploy-64d9c774c6-gfvpb               2/2     Running            0          9s
httpbin-ns       httbin-deploy-6bf8fcb5c5-bxs9p             2/2     Running            0          21s
...
```

```text
$ kubectl logs -n curl-ns curl-deploy-64d9c774c6-zs2jf 
...
{
  "uuid": "ecadb5d5-ecbe-42f5-b74c-074f9586c665"
}
{
  "uuid": "98d53ac4-b1ef-4090-a0fa-9d76facd1f3d"
}
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

1\. Select "Graph" in the left pane.

2\. From the "Select Namespaces" dropbox, select "curl-ns" and "httpbin-ns".

3\. From the "Display" dropbox, select "Traffic Rate", "Namespace Boxes" and "Traffic Animation".

## API Centric Service Mesh

```text
$ kubectl apply -f mesh.A.yaml
serviceentry.networking.istio.io/httpbin-org-service-entry created
virtualservice.networking.istio.io/httpbin-org-virtual-service created
```

```text
$ kubectl apply -f curl.API.yaml
namespace/curl-ns unchanged
serviceaccount/curl-sa unchanged
deployment.apps/curl-deploy configured
```

1\. From the "Display dropbox", select "Traffic Distribution" and unselect "Traffic Rate" and "Service Nodes.

```text
$ kubectl apply -f node-uuid.yaml
namespace/node-uuid-ns created
serviceaccount/node-uuid-sa created
deployment.apps/node-uuid-deploy created
service/node-uuid created
```

```text
$ kubectl get pods -A
NAMESPACE        NAME                                       READY   STATUS    RESTARTS   AGE
curl-ns          curl-deploy-68b698bf7c-tqhhl               2/2     Running   0          3m29s
httpbin-ns       httbin-deploy-6bf8fcb5c5-bxs9p             2/2     Running   0          13m
...
node-uuid-ns     node-uuid-deploy-5b798b5888-wb84f          2/2     Running   0          41s
```

1\. From the "Display dropbox", select "Idle Nodes".

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
