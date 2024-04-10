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
$ ssh-keygen -t ed25519 -N '' -f .openshift-local/openshift-key.pem
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
export OFFLINE_API_TOKEN="eyJhbG...
```

---

### Generate RedHat SSO Authorization Bearer Token

```shell
## SSO Redhat Authorization Bearer Token
$ export TOKEN="$(curl \
    --silent \
    --data-urlencode "grant_type=refresh_token" \
    --data-urlencode "client_id=cloud-services" \
    --data-urlencode "refresh_token=${OFFLINE_API_TOKEN}" \
    https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token | \
    jq -r .access_token)"
```

---

### Establish OpenShift local domain variables

```shell
# Determine localhost Network Interface IP
export HOST_IP="$(ifconfig en4 | grep -oE 'inet [0-9.]+' | tr -d 'inet\s' | awk '{print $2}' | sed 's/^[[:space:]]*//')"
## OpenShift Cluster Settings
export OPENSHIFT_SSH_KEY="$(cat $(pwd)/.openshift-local/openshift-key.pem)"
export OPENSHIFT_VERSION=4.14.12
export OPENSHIFT_DOMAIN=openshift.local.io
export OPENSHIFT_CLUSTER=consul-openshift-sno
export OPENSHIFT_HOSTNAME=consul-openshift-sno
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
$ podman init && \
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
local=/$OPENSHIFT_DOMAIN/
domain=$OPENSHIFT_DOMAIN
address=/apps.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/api.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/api-int.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
address=/$OPENSHIFT_HOSTNAME.$OPENSHIFT_DOMAIN/$OPENSHIFT_IP
ptr-record=$PTR.in-addr.arpa,$OPENSHIFT_HOSTNAME.$OPENSHIFT_DOMAIN
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
$ echo "$OPENSHIFT_IP    $OPENSHIFT_DOMAIN" | sudo tee -a /etc/hosts
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

**File**: `ai-installer-template.json`
```json
{
  "name": "$OPENSHIFT_CLUSTER",
  "openshift_version": "$OPENSHIFT_VERSION",
  "ocp_release_image": "$OCP_RELEASE_IMAGE",
  "base_dns_domain": "$OPENSHIFT_DOMAIN",
  "hyperthreading": "all",
  "user_managed_networking": true,
  "vip_dhcp_allocation": false,
  "high_availability_mode": "None",
  "ssh_public_key": "$SSH_KEY_LOCAL",
  "pull_secret": "$PULL_SECRET",
  "network_type": "OVNKubernetes"
}
```

Use template file `ai-installer-template.json` and `envsubst` to render payload:

```shell
$ envsubst < ai-installer-template.json > assisted-installer-deployment.json
```

Deploy OpenShift Cluster & Obtain Cluster ID:

```bash
# Create NMSTATE tmp file
$ DATA=$(mktemp)

# Deploy cluster and set Cluster ID environment variable
$ CLUSTER_ID=$(curl --silent \
    --request POST \
    --header "Content-Type: application/json" \
    --header "Authorization: Bearer $TOKEN" \
    --data @./assisted-installer-deployment.json \
    "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters" \
    | jq -r '.id' )
```

## Configure SNO Static Networking NMSTATE:

Configure `nmstate.yaml` for static networking:

```yaml
dns-resolver:
  config:
    server:
      - 192.168.0.40
interfaces:
  - ipv4:
      enabled: true
      address:
        - ip: 192.168.0.40
          prefix-length: 24
      dhcp: false
    name: enp0s1
    state: up
    type: ethernet
routes:
  config:
    - destination: 0.0.0.0/0
      next-hop-address: 192.168.0.1
      next-hop-interface: enp0s1
      table-id: 254
```

Generate Static Networking Data:

```shell
jq --null-input \
  --arg OPENSHIFT_SSH_KEY "$OPENSHIFT_SSH_KEY" \
  --arg NMSTATE_YAML "$(cat nmstate.yaml)" \
  --arg MAC_ADDRESS "$SNO_NIC_MAC_ADDRESS" \
  --arg NIC "$SNO_NIC" \
'{
  "ssh_public_key": $OPENSHIFT_SSH_KEY,
  "image_type": "full-iso",
  "static_network_config": [
    {
      "network_yaml": $NMSTATE_YAML,
      "mac_interface_map": [{"mac_address": "$MAC_ADDRESS", "logical_nic_name": "$NIC"}]
    }
  ]
}' >> "$DATA"
```

Generate AI installer `iso` from `https://api.openshift.com/api/assisted-install/`:

```shell
$ curl -X POST \
  "https://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/downloads/image" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d @"$DATA"
```

Copy AI installer `iso` file:

```shell
# Download iso named discovery-image-"$OPENSHIFT_CLUSTER".iso
$ curl -L \
  "http://$ASSISTED_SERVICE_API/api/assisted-install/v1/clusters/$CLUSTER_ID/downloads/image" \
  -o discovery-image-"$OPENSHIFT_CLUSTER".iso \
  -H "Authorization: Bearer $TOKEN"
```

## Create OpenShift Agent Installation ISO

This step configures the final Agent Installation ISO with the properly pre-populated network
settings to run on local Virtual Machine.

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