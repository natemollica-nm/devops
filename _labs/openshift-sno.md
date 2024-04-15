---
title:  "OpenShift SNO Cluster"
layout: default
nav_order: 3
---

# ![openshift](https://github.com/natemollica-nm/devops/assets/57850649/34711e45-1e7f-40d6-a900-309195d4a26f){:width="10%"} OpenShift: Installation on Single Node
{: .no_toc }

Installation guide for OpenShift on Single Node.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## References

* [OpenShift Assisted Installer](https://developers.redhat.com/api-catalog/api/assisted-install-service#content-operations-group-installer)
* [Installing with Assisted Installer API](https://access.redhat.com/documentation/fr-fr/assisted_installer_for_openshift_container_platform/2023/html/assisted_installer_for_openshift_container_platform/installing-with-api)
* [Deploying Single Node OpenShift via Assisted Installer API](https://schmaustech.blogspot.com/2021/08/deploying-single-node-openshift-via.html)

## Requirements

**Resource Requirements**

| Profile | vCPU         | Memory | Storage |
|:--------|--------------|--------|---------|
| Minimum | 8 vCPU cores | 16GB   | 120 GB  |


**Required DNS Records**

| Usage          | FQDN                                   | Description                                                                                                                             |
|:---------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Kubernetes API | `api.<cluster_name>.<base_domain>`     | Add a DNS `A/AAAA` or `CNAME` record. This record must be resolvable by clients external to the cluster.                                |
| Internal API   | `api-int.<cluster_name>.<base_domain>` | Add a DNS `A/AAAA` or `CNAME` record when creating the ISO manually. This record must be resolvable by nodes within the cluster.        |
| Ingress Route  | `*.apps.<cluster_name>.<base_domain>`  | Add a wildcard DNS `A/AAAA` or `CNAME` record that targets the node. This record must be resolvable by clients external to the cluster. |

## Prerequisites

### Install [podman](https://podman.io/)

```shell
$ brew install podman
```

---

### Install UTM for mac

Visit [UTM download](https://mac.getutm.app/) site to obtain UTM dmg package:

![UTM](https://github.com/natemollica-nm/devops/assets/57850649/6df485c4-a867-45ee-b030-a4bcf6059894)

Run the installer package and complete application installation.

---

### Generate openshift ssh-keys for OpenShift local node

```shell
$ ssh-keygen -t ed25519 -N '' -f openshift-key.pem
```

---

### Obtain RedHat CoreOS Live ISO

```shell
# Obtain proper versioning URL for openshift-installation binary
$ export URL="$(openshift-install coreos print-stream-json | grep location | grep aarch64 | grep iso | cut -d\" -f4)"

# Download RHCOS Live ISO as rhcos-live.iso
$ curl -sL "$URL" -o rhcos-live.iso
```

---

### Build OpenShift Installer Podman Container Image



**Filename**: `Containerfile`

```Dockerfile
# Preparation Build
FROM quay.io/centos/centos:stream9-minimal as prep

# ARG
ARG VERSION

# ENV
ENV VERSION=${VERSION}
ENV ARCH=aarch64

# Download binary
RUN microdnf install tar gzip -y
RUN curl -sLO https://mirror.openshift.com/pub/openshift-v4/${ARCH}/clients/ocp/${VERSION}/openshift-install-linux.tar.gz
RUN curl -sLO https://mirror.openshift.com/pub/openshift-v4/${ARCH}/clients/ocp/${VERSION}/openshift-client-linux.tar.gz
RUN tar xzf openshift-install-linux.tar.gz
RUN tar xzf openshift-client-linux.tar.gz

# Build image
FROM quay.io/centos/centos:stream9-minimal as build

RUN microdnf update -y \
    && microdnf install nmstate -y \
    && microdnf clean all

COPY --from=prep openshift-install /usr/local/bin
COPY --from=prep oc /usr/local/bin

ENTRYPOINT ["/usr/local/bin/openshift-install"]
```

Build `openshift-install` agent `iso` image:

```shell
$ podman build -f Containerfile -t openshift-install --build-arg VERSION="${OPENSHIFT_VERSION}"
```

---

### Obtain OpenShift Offline API Token

* Visit your [RedHat Hybrid Cloud Console](https://console.redhat.com/openshift/token/show) and obtain offline token value:

![redhat-offline-api-token](https://github.com/natemollica-nm/devops/assets/57850649/39788dc3-f3b1-45c7-bd7e-de635fdac7e1){:width="100%"}

Set `OFFLINE_API_TOKEN` environment variable to this value.

```shell
export OFFLINE_API_TOKEN="eyJhbG..."
```

---

### Obtain Assisted Installer REST API Token 

> **_Note_**: _This token expires every 15 minutes._

```shell
## Redhat Authorization Bearer Token
$ export TOKEN=$( \
      curl \
      --silent \
      --header "Accept: application/json" \
      --header "Content-Type: application/x-www-form-urlencoded" \
      --data-urlencode "grant_type=refresh_token" \
      --data-urlencode "client_id=cloud-services" \
      --data-urlencode "refresh_token=${OFFLINE_API_TOKEN}" \
      "https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token" \
      | jq --raw-output ".access_token" \
)
```

Verify API Access:

```shell
$ curl -s https://api.openshift.com/api/assisted-install/v2/component-versions -H "Authorization: Bearer ${TOKEN}" | jq
{
  "release_tag": "v2.30.2",
  "versions": {
    "assisted-installer": "registry.redhat.io/rhai-tech-preview/assisted-installer-rhel8:v1.0.0-326",
    "assisted-installer-controller": "registry.redhat.io/rhai-tech-preview/assisted-installer-reporter-rhel8:v1.0.0-404",
    "assisted-installer-service": "quay.io/app-sre/assisted-service:c9c2dbf",
    "discovery-agent": "registry.redhat.io/rhai-tech-preview/assisted-installer-agent-rhel8:v1.0.0-312"
  }
}
```

---

### Establish OpenShift local domain variables

```shell
# Determine localhost Network Interface IP
export HOST_IP="$(ifconfig en4 | grep -oE 'inet [0-9.]+' | tr -d 'inet\s' | sed 's/^[[:space:]]*//')"
## OpenShift Cluster Settings
export PULL_SECRET="$(jq -rc . < pull-secret.txt | sed 's/\"/\\"/g')"
export OPENSHIFT_SSH_KEY="$(cat openshift-key.pem.pub)"
export OPENSHIFT_VERSION=4.13.38
export OPENSHIFT_DOMAIN=openshift.local.io
export OPENSHIFT_CLUSTER=consul-openshift-sno
export OPENSHIFT_HOSTNAME=conri-openshift-sno
## OpenShift Static IP Settings
export SNO_NIC=enp0s1
export SNO_NIC_MAC_ADDRESS="4E:B5:D2:58:F8:56"
export CIDR="$(printf '%s\n' "$HOST_IP" | cut -d'.' -f1-3).0/24"
export GATEWAY="$(printf '%s\n' "$HOST_IP" | cut -d'.' -f1-3).1"
export OPENSHIFT_IP="$(printf '%s\n' "$HOST_IP" | cut -d'.' -f1-3).40"
export PTR="$(echo "$OPENSHIFT_IP" | awk -F '.' '{print $4"."$3"."$2"."$1}')"
export OCP_RELEASE_IMAGE="$(openshift-install version | awk '/release image/ {print $3}')"
## OpenShift API
export ASSISTED_SERVICE_API="api.openshift.com"
```

---

### Configure OpenShift DNS via Podman `dnsmasq`

Initialize and configure podman virtual machine to run in `root` user mode:

```shell
$ podman machine init && \
  podman machine set --rootful
```

Start podman virtual machine:

```shell
$ podman machine start
```

Disable podman `DNSStubListener` for podman `dnsmasq` container:

```shell
$ podman machine ssh sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
```

Restart podman systemd-resolved:

```shell
$ podman machine ssh systemctl restart systemd-resolved
```

---

### Configure `dnsmasq`

**Filename**: `dnsmasq-template.conf`
{% highlight cfg %}
user=root
port=53
expand-hosts
log-queries
log-facility=-
local=/local.io/
domain=local.io
address=/apps.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/api.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/api-int.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/bootstrap.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/$OPENSHIFT_HOSTNAME.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
ptr-record=$PTR.in-addr.arpa,$OPENSHIFT_HOSTNAME.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN
rev-server=$CIDR,127.0.0.1
{% endhighlight %}

Generate `dnsmasq` configuration file using template:

```shell
$ envsubst < dnsmasq-template.conf > dnsmasq.conf
```

### Confirm `DNSStubListener` disabled for `dnsmasq`

```shell
$ podman machine ssh ss -ltnup
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess                                    
udp   UNCONN 0      0            0.0.0.0:5355      0.0.0.0:*    users:(("systemd-resolve",pid=2335,fd=10))
udp   UNCONN 0      0          127.0.0.1:323       0.0.0.0:*    users:(("chronyd",pid=1675,fd=5))         
udp   UNCONN 0      0               [::]:5355         [::]:*    users:(("systemd-resolve",pid=2335,fd=12))
udp   UNCONN 0      0              [::1]:323          [::]:*    users:(("chronyd",pid=1675,fd=6))         
tcp   LISTEN 0      4096         0.0.0.0:5355      0.0.0.0:*    users:(("systemd-resolve",pid=2335,fd=11))
tcp   LISTEN 0      128          0.0.0.0:22        0.0.0.0:*    users:(("sshd",pid=1820,fd=3))            
tcp   LISTEN 0      4096            [::]:5355         [::]:*    users:(("systemd-resolve",pid=2335,fd=13))
tcp   LISTEN 0      128             [::]:22           [::]:*    users:(("sshd",pid=1820,fd=4))
```

Confirm port `53` isn't being used by `dnsmasq` podman container and enter `ctrl+c` when satisfied.

---

### Run podman `dnsmasq` container

Use `dnsmasq.conf` for `dnsmasq` podman container runtime DNS configuration:

```shell
$ podman run -d --rm -p 53:53/udp -v ./dnsmasq.conf:/etc/dnsmasq.conf --name dnsmasq quay.io/crcont/dnsmasq:latest
```

---

### Update local systemd resolver and `/etc/hosts` file with OpenShift domain

Configure nameserver entry for OpenShift local domain.

```shell
$ echo "nameserver 127.0.0.1" | sudo tee "/etc/resolver/$OPENSHIFT_DOMAIN"
```

Update `/etc/hosts` with OpenShift domain:

```shell
$ echo "$OPENSHIFT_IP    api.$OPENSHIFT_CLUSTER.$OPENSHIFT_DOMAIN" | sudo tee -a /etc/hosts
```

---

### Verify OpenShift SNO local domain resolution

```shell
$ dig +noall +answer @127.0.0.1 api."$OPENSHIFT_CLUSTER"."$OPENSHIFT_DOMAIN"
  api.consul-openshift-sno.openshift.local.io.    60      IN      A       192.168.0.40
  
$ dig +noall +answer @127.0.0.1 api-int."$OPENSHIFT_CLUSTER"."$OPENSHIFT_DOMAIN"
  api-int.consul-openshift-sno.openshift.local.io.    60      IN      A       192.168.0.40
  
$ dig +noall +answer @127.0.0.1 openshift-cluster-console.apps."$OPENSHIFT_CLUSTER"."$OPENSHIFT_DOMAIN"
  openshift-cluster-console.apps.consul-openshift-sno.openshift.local.io.    60      IN      A       192.168.0.40
  
$ dig +noall +answer @127.0.0.1 fake-service.apps."$OPENSHIFT_CLUSTER"."$OPENSHIFT_DOMAIN"
  my-fake-service.apps.consul-openshift-sno.openshift.local.io.    60      IN      A       192.168.0.40
  
$ dig +noall +answer @127.0.0.1 bootstrap."$OPENSHIFT_CLUSTER"."$OPENSHIFT_DOMAIN"
  bootstrap.consul-openshift-sno.openshift.local.io.    60      IN      A       192.168.0.40
```

---

## Deploy OpenShift Cluster via Assisted Installer API

Create OpenShift SNO Cluster API Payload (leave variable syntax in place):

**File**: `assisted-installer-deployment-template.json`
```json
{
  "name": "$OPENSHIFT_CLUSTER",
  "cpu_architecture" : "arm64",
  "openshift_version": "$OPENSHIFT_VERSION",
  "ocp_release_image": "$OCP_RELEASE_IMAGE",
  "base_dns_domain": "$OPENSHIFT_DOMAIN",
  "hyperthreading": "all",
  "user_managed_networking": true,
  "vip_dhcp_allocation": false,
  "high_availability_mode": "None",
  "ssh_public_key": "$OPENSHIFT_SSH_KEY",
  "pull_secret": "$PULL_SECRET",
  "network_type": "OVNKubernetes"
}
```

Use template file `assisted-installer-deployment-template.json` and `envsubst` to render payload:

```shell
$ envsubst < assisted-installer-deployment-template.json > assisted-installer-deployment.json
```

Deploy OpenShift Cluster & Obtain Cluster ID:

```shell
# Register cluster and obtain ClusterID
$ export CLUSTER_ID=$(curl --silent \
    --request POST \
    --header "Content-Type: application/json" \
    --header "Authorization: Bearer $TOKEN" \
    --data @./assisted-installer-deployment.json \
    "https://api.openshift.com/api/assisted-install/v2/clusters" \
    | jq -r '.id' )
```

Check status of cluster:

```shell
$ curl \
  --silent \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer $TOKEN" \
  --request GET "https://api.openshift.com/api/assisted-install/v2/clusters/$CLUSTER_ID" \
  | jq
```

## Register Cluster Infrastructure Environment

Create Infrastructure template payload:

**File**: `ai-infra-template.json`
```json
{
  "name": "$OPENSHIFT_CLUSTER-infra",
  "image_type":"full-iso",
  "cluster_id": "$CLUSTER_ID",
  "cpu_architecture" : "arm64",
  "pull_secret": "$PULL_SECRET"
}
```

Use template file `ai-infra-template.json` and `envsubst` to render payload:

```shell
$ envsubst < ai-infra-template.json > ai-infra-deployment.json
```

Register OpenShift Cluster Infrastructure & Obtain InfrastructureID:

```shell
$ export INFRA_ENV_ID=$(curl \
    --silent \
    --header "Authorization: Bearer ${TOKEN}" \
    --header "Content-Type: application/json" \
    --data @./ai-infra-deployment.json \
    "https://api.openshift.com/api/assisted-install/v2/infra-envs" \
    | jq -r '.id' )
```

Verify Infrastructure Registration:

```shell
$ curl \
  --silent \
  --request GET \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/json" \
  --url "https://api.openshift.com/api/assisted-install/v2/infra-envs/$INFRA_ENV_ID" \
  | jq .
```

## Configure Infrastructure Static Networking

Configure `nmstate-template.yaml` for static networking:

```yaml
dns-resolver:
  config:
    server:
      - "$OPENSHIFT_IP"
interfaces:
  - ipv4:
      enabled: true
      address:
        - ip: "$OPENSHIFT_IP"
          prefix-length: 24
      dhcp: false
    name: "$SNO_NIC"
    state: up
    type: ethernet
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: "$GATEWAY"
      next-hop-interface: "$SNO_NIC"
      table-id: 254
```

Use template file `nmstate-template.yaml` and `envsubst` to render yaml for payload:

```shell
$ envsubst < nmstate-template.yaml > nmstate.yaml
```


```shell
$ curl \
  --silent \
  --request PATCH \
  --header "Authorization: Bearer $TOKEN" \
  --header 'Content-Type: application/json' \
  --url "https://api.openshift.com/api/assisted-install/v2/infra-envs/$INFRA_ENV_ID" \
  --data "$(jq \
      --null-input \
      --arg NMSTATE_YAML "$(cat nmstate.yaml)" \
      --arg MAC_ADDRESS "$SNO_NIC_MAC_ADDRESS" \
      --arg NIC "$SNO_NIC" '{
  "static_network_config": [
    {
      "mac_interface_map": [
        {
          "logical_nic_name": $NIC,
          "mac_address": $MAC_ADDRESS
        }
      ],
      "network_yaml": $NMSTATE_YAML
    }
  ]
}')" | jq -r '.static_network_config'
```

**Sample Return**

```json
[
  {
    "mac_interface_map": [
      {
        "logical_nic_name": "enp0s1",
        "mac_address": "4E:B5:D2:58:F8:56"
      }
    ],
    "network_yaml": "dns-resolver:\n..."
  }
]
```

## Download Assisted Installer Discovery ISO

Obtain Infrastructure Download URL for Discovery ISO Image:

```shell
$ export DISCOVERY_URL=$(curl \
      --silent \
      --header "Authorization: Bearer ${TOKEN}" \
      "https://api.openshift.com/api/assisted-install/v2/infra-envs/${INFRA_ENV_ID}/downloads/image-url" \
      | jq -r '.url')
```

Download from URL:

```shell
$ wget -qq "$DISCOVERY_URL" -O discovery.iso
```

## Update `discovery.iso` bootstrap kernel network settings:

```shell
$ export KERNEL_ARGS="console=ttyS0 rd.neednet=1 ip=${OPENSHIFT_IP}::${GATEWAY}:255.255.255.0:${OPENSHIFT_HOSTNAME}.${OPENSHIFT_CLUSTER}.${OPENSHIFT_DOMAIN}:enp0s1:off::[4E:B5:D2:58:F8:56] nameserver=${HOST_IP} nameserver="${GATEWAY}" nameserver=8.8.8.8"

$ coreos-installer iso kargs modify -a "$KERNEL_ARGS" discovery.iso
```

```shell
$ coreos-installer iso kargs show discovery.iso
  coreos.liveiso=rhcos-415.92.202311241643-0 ignition.firstboot ignition.platform.id=metal console=ttyS0 rd.neednet=1 ip=192.168.0.40::192.168.0.1:255.255.255.0:conri-openshift-sno.consul-openshift-sno.openshift.local.io:enp0s1:off::[4E:B5:D2:58:F8:56] nameserver=192.168.0.100 nameserver=192.168.0.1 nameserver=8.8.8.8
```

## Create OpenShift Agent Installation ISO

This step configures the final Agent Installation ISO with the properly pre-populated network
settings to run on local Virtual Machine.

### Create `install-config.yaml` and `agent-config.yaml`

Create the following template files:

**Filename**: `install-config-template.yaml`
```yaml
apiVersion: v1
metadata:
  name: "$OPENSHIFT_CLUSTER"
baseDomain: "$OPENSHIFT_DOMAIN"
compute:
  - name: worker
    replicas: 0
controlPlane:
  name: master
  replicas: 1
  architecture: arm64
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: "$CIDR"
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
bootstrapInPlace:
  installationDisk: /dev/vda
fips: false
pullSecret: $PULL_SECRET
sshKey: "$OPENSHIFT_SSH_KEY"
```

**Filename**: `agent-config-template.yaml`
```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: "$OPENSHIFT_CLUSTER"
rendezvousIP: "$OPENSHIFT_IP"
hosts:
    - hostname: "$OPENSHIFT_HOSTNAME"
      interfaces:
        - name: "enp0s1"
          macAddress: "4E:B5:D2:58:F8:56"
      rootDeviceHints:
        deviceName: "/dev/vda"
      networkConfig:
        interfaces:
          - name: "enp0s1"
            type: ethernet
            state: up
            mac-address: "4E:B5:D2:58:F8:56"
            ipv4:
              enabled: true
              address:
                - ip: "$OPENSHIFT_IP"
                  prefix-length: 23
              dhcp: false
        dns-resolver:
          config:
            server:
              - "$HOST_IP"
        routes:
          config:
            - destination: "0.0.0.0/0"
              next-hop-address: "$GATEWAY"
              next-hop-interface: "enp0s1"
              table-id: 254
```

Create OpenShift installer directory:

```shell
$ mkdir --parents ./ocp
```

Use template files and `envsubst` to generate install directory configs:

```shell
$ envsubst <agent-config-template.yaml >ocp/agent-config.yaml && \
  envsubst <install-config-template.yaml >ocp/install-config.yaml
```

### Build `agent.aarch64.iso`

```shell
$ podman run -\
    -privileged \
    --rm \
    -v "$(pwd)":/data \
    -v ./.installer_cache/image_cache:/root/.cache/agent/image_cache \
    -v ./.installer_cache/files_cache:/root/.cache/agent/files_cache \
    -w /data \
    openshift-install:latest --dir ./ agent create image
```

This process will take a few minutes to complete, and will be indicated when you 
see the `Writing manifest to image destination` message.

### Configure Static IP Addressing on Agent Installer ISO image

> _This method of OpenShift SNO installation to install the RHCOS is by using an ISO image file.
Adjusting RHCOS system boot kernel arguments is required since we're implementing static network
settings._

> _RHCOS networking options are adjusted via the `dracut` tool during boot. Detailed explanation of
all available options can be found in the Linux manual page for [`dracut`](https://www.man7.org/linux/man-pages/man7/dracut.cmdline.7.html)._

Set updated `agent.aarch64.ido` Kernel Arguments for anticipated Static Network:

| Component              | Value                                              | Description                                                                                                             |
|------------------------|----------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Host IP                | `$HOST_IP`                                         | Sets localhost's IP Address to user's selected NIC (i.e., en0 or en4, etc)                                              |
| Gateway                | `$GATEWAY`                                         | Configures OpenShift Node's routable gateway IP to host's CIDR prefixed address at `.1`                                 |
| Node IP                | `$OPENSHIFT_IP`                                    | Sets OpenShift Single Node IP to host's CIDR prefixed address at `.40`                                                  |
| Hostname               | `$OPENSHIFT_HOSTNAME`                              | Sets OpenShift Node's hostname to user-defined variable name.                                                           |
| DNS Server             | `$HOST_IP` \| `$HOST_IP` \|`$GATEWAY` \| `8.8.8.8` | Configures OpenShift Node DNS Nameserver to localhost's instance of `dnsmasq` as well as publicly available DNS Servers |
| IPv6 Autoconfiguration | `off` (i.e., `... :$SNO_NIC:off::...`)             | No `auto-configuration` is required when IP networking is configured statically.                                        |
| `netroot`              | `rd.neednet=1`                                     | Brings up network even without `netroot` set                                                                            |

* Set an individual static IP address (`ip="$HOST_IP"`). 
* If setting a static IP, you must then identify the DNS server IP address (`nameserver="$HOST_IP"`, etc.) on each node. 

```shell
$ export KERNEL_ARGS="console=ttyS0 rd.neednet=1 ip=${OPENSHIFT_IP}::${GATEWAY}:255.255.255.0:${OPENSHIFT_HOSTNAME}.${OPENSHIFT_DOMAIN}:$SNO_NIC:off::[$SNO_NIC_MAC_ADDRESS] nameserver=${HOST_IP} nameserver=${GATEWAY} nameserver=8.8.8.8"
```


```shell
$ coreos-installer iso kargs modify -a "$KERNEL_ARGS" agent.aarch64.iso
```

## Deploy OpenShift SNO on UTM

The agent.aarch64.iso installer ISO image is now configured for deploying OpenShift in its entirety on a
single virtual machine Linux node. 

| Step                                                                                                                                                                                                                 | Description                                                                                                                    |
|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Open UTM Application and select **Create a New Virtual Machine**:                                                                                                                                                    | ![UTM_VM](https://github.com/natemollica-nm/devops/assets/57850649/f1cf2623-53fd-4801-93ef-c01e68394d92){:width="75%"}         |
| From the **Start** pop-up menu, select the **Virtualize** option:                                                                                                                                                    | ![UTM_START](https://github.com/natemollica-nm/devops/assets/57850649/b833f35c-503c-47f0-bb5e-16b3d097c327){:width="75%"}      |
| From the **Operating System** pop-up menu, select the **Linux** option:                                                                                                                                              | ![UTM_OS](https://github.com/natemollica-nm/devops/assets/57850649/17f07c01-c342-4bbc-98b4-e59d0888f3d4){:width="75%"}         |
| From the **Linux** pop-up menu, select the **Browse** option and locate the `agent.aarch64.iso` image:<br/>> **_Note_**: Leave the **Use Apple Virtualization** and **Boot from kernel image** checkboxes unchecked. | ![UTM_BOOT](https://github.com/natemollica-nm/devops/assets/57850649/4b197561-1b87-478e-8f94-a8b8cafdb1e6){:width="75%"}       |
| Locate the `agent.aarch64.iso` from the Finder pop-up window and select:                                                                                                                                             | ![UTM_ISO](https://github.com/natemollica-nm/devops/assets/57850649/dfeb6d5d-df47-46b4-9c6e-2ac662e7bb9a){:width="75%"}        |
| From the **Hardware** pop-up window, adjust the VM's memory to at least `16392 MB` (16Gi):                                                                                                                           | ![UTM_MEM](https://github.com/natemollica-nm/devops/assets/57850649/625947a3-555a-42cc-a09d-98ccc7aa9432){:width="85%"}        |
| From the **Storage** pop-up window, adjust the VM's storage size to at least `120 GB`:                                                                                                                               | ![UTM_STORAGE](https://github.com/natemollica-nm/devops/assets/57850649/59c9c454-417a-411e-bb70-02bbd2aa9fac){:width="75%"}    |
| Select **Continue** from the **Shared Directory** window:                                                                                                                                                            | ![UTM_SHARED_DIR](https://github.com/natemollica-nm/devops/assets/57850649/e50db7fd-e7c8-47ac-b3a5-7e7c639508e0){:width="75%"} |
| From the **Summary** pop-up window, set the VM's name to the value of `$OPENSHIFT_HOSTNAME` and check the **Open VM Settings** checkbox:                                                                             | ![UTM_SUMMARY](https://github.com/natemollica-nm/devops/assets/57850649/54da2425-bf6a-42df-98a9-401a09120caa){:width="75%"}    |
| Select **Save** and proceed.                                                                                                                                                                                         |                                                                                                                                |
| From the **VM Settings** window, navigate to **Network** in the left-pane and adjust the **MAC Address** and **Network Mode** settings to match `$SNO_NIC_MAC_ADDRESS` and `Bridged (Advanced)` respectively:        | ![UTM_NETWORK](https://github.com/natemollica-nm/devops/assets/57850649/908e24d7-a200-4813-8d9a-9d86b29dbe36){:width="75%"}    |
| Monitor Virtual Machine deployment after booting:                                                                                                                                                                    |                                                                                                                                | 