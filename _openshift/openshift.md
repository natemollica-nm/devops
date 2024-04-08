---
title:  "OpenShift"
layout: default
---

# ![openshift](https://github.com/natemollica-nm/devops/assets/57850649/34711e45-1e7f-40d6-a900-309195d4a26f){:width="10%"} OpenShift: Installation & Networking
{: .no_toc }

Installation and Networking concepts for OpenShift.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## OpenShift Installation Configuration


---

## OpenShift Networking: Required Ports

**Machine-to-Machine Network Communication**

| Protocol | Port        | Description                                                                                                        |
|----------|-------------|--------------------------------------------------------------------------------------------------------------------|
| ICMP     | N/A         | Network reachability tests                                                                                         |
| TCP      | 1936        | Metrics                                                                                                            |
| TCP      | 9000-9999   | Host level services, including the node exporter on ports 9100-9101 and the Cluster Version Operator on port 9099. |
| TCP      | 10250-10259 | The default ports that Kubernetes reserves                                                                         |
| UDP      | 4789        | VXLAN                                                                                                              |
| UDP      | 6081        | Geneve                                                                                                             |
| UDP      | 9000-9999   | Host level services, including the node exporter on ports 9100-9101.                                               |
| UDP      | 500         | IPsec IKE packets                                                                                                  |
| UDP      | 4500        | IPsec NAT-T packets                                                                                                |
| TCP/UDP  | 30000-32767 | Kubernetes node port                                                                                               |
| ESP      | N/A         | IPsec Encapsulating Security Payload (ESP)                                                                         |

**Machine-to-Control Plane Network Communication**

| Protocol | Port | Description    |
|----------|------|----------------|
| TCP      | 6443 | Kubernetes API |

**Control Plane-to-Control Plane Machine Network Communication**

| Protocol | Port      | Description                |
|----------|-----------|----------------------------|
| TCP      | 2379-2380 | etcd server and peer ports |

---

## OpenShift Networking: DNS Requirements

| Component              | Record                                             | Description                                                                                                                                                                                                     |
|------------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kubernetes API         | `api.<cluster_name>.<base_domain>.`                | A DNS A/AAAA or CNAME record, and a DNS PTR record, to identify the API load balancer. These records must be resolvable by both clients external to the cluster and from all the nodes within the cluster.      |
|                        | `api-int.<cluster_name>.<base_domain>.`            | A DNS A/AAAA or CNAME record, and a DNS PTR record, to internally identify the API load balancer. These records must be resolvable from all the nodes within the cluster.                                       |
| Routes                 | `*.apps.<cluster_name>.<base_domain>.`             | A wildcard DNS A/AAAA or CNAME record that refers to the application ingress load balancer. These records must be resolvable by both clients external to the cluster and from all the nodes within the cluster. |
| Bootstrap machine      | `bootstrap.<cluster_name>.<base_domain>.`          | A DNS A/AAAA or CNAME record, and a DNS PTR record, to identify the bootstrap machine. These records must be resolvable by the nodes within the cluster.                                                        |
| Control plane machines | `<control_plane><n>.<cluster_name>.<base_domain>.` | DNS A/AAAA or CNAME records and DNS PTR records to identify each machine for the control plane nodes. These records must be resolvable by the nodes within the cluster.                                         |
| Compute machines       | `<compute><n>.<cluster_name>.<base_domain>.`       | DNS A/AAAA or CNAME records and DNS PTR records to identify each machine for the worker nodes. These records must be resolvable by the nodes within the cluster.                                                |

### OpenShift DNS Validation

You can use `dig` to validate that the currently available DNS Records are
resolving as expected.

```shell
$ dig +noall +answer @dns-server.natemollica-nm.github.io api.openshift-cluster.natemollica-nm.github.io
  api.openshift-cluster.natemollica-nm.github.io.    60      IN      A       124.8.186.127
  
$ dig +noall +answer @dns-server.natemollica-nm.github.io api-int.openshift-cluster.natemollica-nm.github.io
  api-int.openshift-cluster.natemollica-nm.github.io.    60      IN      A       124.8.186.127
  
$ dig +noall +answer @dns-server.natemollica-nm.github.io openshift-cluster-console.apps.openshift-cluster.natemollica-nm.github.io
  openshift-cluster-console.apps.openshift-cluster.natemollica-nm.github.io.    60      IN      A       124.8.186.127
  
$ dig +noall +answer @dns-server.natemollica-nm.github.io my-fake-service.apps.openshift-cluster.natemollica-nm.github.io
  my-fake-service.apps.openshift-cluster.natemollica-nm.github.io.    60      IN      A       124.8.186.127
  
$ dig +noall +answer @dns-server.natemollica-nm.github.io bootstrap.openshift-cluster.natemollica-nm.github.io
  bootstrap.openshift-cluster.natemollica-nm.github.io.    60      IN      A       124.8.186.127
```

---

## OpenShift CNO (Cluster Network Operator)

OpenShift Networking configuration is managed by a customer resource (CR) object named `cluster`. The CR is
a part of the [Cluster Network Operator (CNO)][CNO] configuration spec. The CR specifies the
fields for the `Network` API in the `operator.openshift.io` API group.

The CNO configuration inherits the following fields during cluster installation from the `Network` API 
in the `Network.config.openshift.io` API group:

| Network API Field     | Description                                                                               |
|-----------------------|-------------------------------------------------------------------------------------------|
| `clusterNetwork`      | IP address pools from which pod IP addresses are allocated.                               |
| `serviceNetwork`      | IP address pool for services.                                                             |
| `defaultNetwork.type` | Cluster network plugin. `OVNKubernetes` is the only supported plugin during installation. |



## Supported Service Mesh configurations 

* This release of Red Hat OpenShift Service Mesh is only available on OpenShift Container Platform `x86_64` (and some specific [IBM products][openshift-mesh]...)
* Configurations where all Service Mesh components are contained within a single OpenShift Container Platform cluster.
* Configurations that do not integrate external services such as virtual machines.
* Red Hat OpenShift Service Mesh does not support `EnvoyFilter` configuration except where explicitly documented.


# Appendix A: OpenShift Installation Configuration Example


```yaml
apiVersion: v1
baseDomain: natemollica-nm.github.io
credentialsMode: Manual
metadata:
  name: openshift-cluster
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
  platform:
    aws:
      zones: [
        "us-east-2a",
        "us-east-2b",
        "us-east-2c"
      ]
      rootVolume:
        iops: 4000
        size: 200
        type: io1
      type: m6i.xlarge
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 3
  platform:
    aws:
      zones: var.subnet_azs
      rootVolume:
        iops: 4000
        size: 200
        type: io1
      type: m6i.xlarge
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 10.0.0.0/16
  serviceNetwork:
    - 172.30.0.0/16
platform:
  aws:
    region: "us-east-2"
    # Public and Private Subnet IP CIDRs (Overall Subnet: 10.0.0.0/16)
    subnets: [
      "10.0.0.0/19",
      "10.0.64.0/19",
      "10.0.128.0/19",
      "10.0.32.0/19",
      "10.0.96.0/19",
      "10.0.160.0/19"
    ]
    lbType: NLB
    propagateUserTags: true
    userTags:
      adminContact: natemollica
      costCenter: 0669
    amiID: ami-0c5d3e03c0ab9b19a
    serviceEndpoints:
      - name: ec2
        url: https://vpce-id.ec2.us-east-2.vpce.amazonaws.com
fips: false
pullSecret: '{"auths": ...}'
sshKey: ssh-ed25519 AAAA...
```


[CNO]: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/network-configuration.html#nw-operator-cr_post-install-network-configuration
[openshift-mesh]: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/network-configuration.html#ossm-supported-configurations-sm_post-install-network-configuration