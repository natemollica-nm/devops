---
title:  "OpenShift: Networking"
layout: default
parent: OpenShift
nav_order: 2
---

# ![openshift](https://github.com/natemollica-nm/devops/assets/57850649/34711e45-1e7f-40d6-a900-309195d4a26f){:width="10%"} OpenShift: Networking
{: .no_toc }

Networking concepts for OpenShift.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## OpenShift Networking: Requirements

* Connectivity between all cluster nodes
* Connectivity for each node to the internet
* Access to an NTP server for time synchronization between the cluster nodes
* A DHCP server unless using static IP addressing.
* A base domain name. You must ensure that the following requirements are met:
  * There is no wildcard, such as `*.<cluster_name>.<base_domain>`, or the installation will not proceed.
  * A DNS `A/AAAA` record for `api.<cluster_name>.<base_domain>`.
  * A DNS `A/AAAA` record with a wildcard for `*.apps.<cluster_name>.<base_domain>`.
* Port `6443` is open for the API URL if you intend to allow users outside the firewall to access the cluster 
  via the oc CLI tool.
* Port `443` is open for the console if you intend to allow users outside the firewall to access the console.
* A DNS `A/AAAA` record for each node in the cluster when using User Managed Networking, or the installation 
  will not proceed. DNS `A/AAAA` records are required for each node in the cluster when using Cluster Managed 
  Networking after installation is complete in order to connect to the cluster, but installation can proceed 
  without the `A/AAAA` records when using Cluster Managed Networking.
* A DNS PTR record for each node in the cluster if you want to boot with the preset hostname when using static 
  IP addressing. Otherwise, the Assisted Installer has an automatic node renaming feature when using static IP 
  addressing that will rename the nodes to their network interface MAC address.

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


[CNO]: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/network-configuration.html#nw-operator-cr_post-install-network-configuration
[openshift-mesh]: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/network-configuration.html#ossm-supported-configurations-sm_post-install-network-configuration