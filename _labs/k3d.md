---
title:  k3d
layout: default
---

## **k3d**

Useful, lightweight local development for Kubernetes on Docker.

### Install

```shell
$ curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```


### **Configure k3d local registry**

Good if testing out local Docker builds prior to pushing to remote repo.

**Update `/etc/hosts` with registry name**

```shell
$ echo '127.0.0.1 k3d-registry.localhost' | sudo tee -a /etc/hosts
```

**Create the registry**

```shell
# Creates repo: k3d-registry.localhost:5002
$ k3d registry create -p 5002 registry.localhost
```

### **Run a local Kubernetes cluster (connect with local registry)**

```shell
$ k3d cluster create c1 --registry-use k3d-registry.localhost:5002
```

> Note: using `k3d cluster create` automatically merges and switches kube-context to the latest k3d cluster built.


### **Pushing Docker images to k3d Local Repo**

Tag Docker image with k3d repo name

```shell
$ docker tag hashicorp/consul-dev-image:local k3d-registry.localhost:5002/consul:latest
```

Push image to the k3d repo

```shell
$ docker push k3d-registry.localhost:5002/consul:latest
```