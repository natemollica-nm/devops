---
title:  "OpenShift: Installation"
layout: default
parent: OpenShift
nav_order: 2
---

# ![openshift][openshift-logo]{:width="10%"} OpenShift: Installation
{: .no_toc }

Installation concepts for OpenShift.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## OpenShift Installation Configuration

* Agent-based Installer `install-config.yaml` Parameters: [Docs][install-config-agent-installer] 

---

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


[openshift-logo]: https://github.com/natemollica-nm/devops/assets/57850649/34711e45-1e7f-40d6-a900-309195d4a26f
[install-config-agent-installer]: https://docs.openshift.com/container-platform/4.15/installing/installing_with_agent_based_installer/installation-config-parameters-agent.html