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
