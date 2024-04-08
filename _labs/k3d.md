---
title:  k3d
layout: default
---

# **k3d**

Useful, lightweight local development for Kubernetes on Docker.

## Install

```shell
$ curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

---

## **Configure k3d local registry**

Good if testing out local Docker builds prior to pushing to remote repo.

Update `/etc/hosts` with registry name

```shell
$ echo '127.0.0.1 k3d-registry.localhost' | sudo tee -a /etc/hosts
```

Create the registry

```shell
# Creates repo: k3d-registry.localhost:5002
$ k3d registry create -p 5002 registry.localhost
```

### **Run a local Kubernetes cluster (connect with local registry)**

```shell
# Create a k3d cluster named "c1" | Resulting context name: k3d-c1
$ k3d cluster create c1 --registry-use k3d-registry.localhost:5002
```

> _**Note**: using `k3d cluster create` automatically merges and switches kube-context to the latest k3d cluster built._

---

### **Pushing Docker images to k3d Local Repo**

Apply k3d repo tag name to image:

```shell
$ docker tag hashicorp/consul-dev:latest k3d-registry.localhost:5002/consul-dev:latest
```

Push image to the k3d repo:

```shell
$ docker push k3d-registry.localhost:5002/consul-dev:latest
```

---

### **Using k3d Repo Images**

Retrieve Docker image repository digest:

```shell
$ docker inspect --format='{{index .RepoDigests 0}}' "$registryPath"/consul-dev:latest
  k3d-registry.localhost:5002/consul-dev@sha256:27708036dc0496562b1f9af487f418a5685dc63540eca34f25843b6f7ae69512
```

Use Docker repo digest name for deployments, helm overrides, etc.:

```yaml
# File: values.yaml
global:
  name: consul
  image: "k3d-registry.localhost:5002/consul-dev@sha256:27708036dc0496562b1f9af487f418a5685dc63540eca34f25843b6f7ae69512"
...
```

Install/apply Kubernetes component as normal:

```shell
$ helm install consul-cluster-01 hashicorp/consul --namespace consul --values values.yaml
```

---

## **k3d Multicluster Testing**

Working with multiple Kubernetes clusters to simulate HA scenarios can get expensive 
when using cloud based resources. k3d offers seamless multicluster connectivity using 
underlying Docker networking resources.

### **Deploy multiple k3d clusters**

```shell
# Create a k3d clusters named "c1" and "c2"
# Resulting context name(s):  k3d-c1 | k3d-c2
$ k3d cluster create c1 --registry-use k3d-registry.localhost:5002 && \
  k3d cluster create c2 --registry-use k3d-registry.localhost:5002
```

### **Optionally Simulate AWS Regionally Tagging**

```shell
$ kubectl --context k3d-c1 label node k3d-c1-server-0 topology.kubernetes.io/region="us-east-1" && \
  kubectl --context k3d-c2 label node k3d-c2-server-0 topology.kubernetes.io/region="us-east-2"
```

### **Establish Docker Network Connectivity between Clusters**

```shell
# Establish connectivity between the clusters
# bidirectionally.
$ docker network connect k3d-c1 k3d-c2-server-0 && \
  docker network connect k3d-c2 k3d-c1-server-0
```