---
title:  "OpenShift: Troubleshooting"
layout: default
---

# **OpenShift: Troubleshooting**
{: .no_toc }

Tips and tricks for troubleshooting networking issues on OpenShift.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## OpenShift: `tcpdump`

**When to Use**: There's a need to inspect network traffic from an OpenShift pod.

> _**Note**: OpenShift leverages the use of `nsenter` when running `tcpdump`. Nsenter is a Utility for running commands in the context of different namespaces. Required for CoreOS Rhel running the underlying containers and managing kernel level namespaces in OpenShift._

**Available Options**:
* (OpenShift v4.x.x) Run `tcpdump` directly from OpenShift Node
* (OpenShift v4.x.x) Run `tcpdump` from OpenShift managed **_debug pod_**
* (OpenShift v4.x.x) Run `tcpdump` via [ksniff][A001]

---

## Prerequisite(s) for all `tcpdump` Capture Methods

### Determine OpenShift NodeName, Pod Namespace, and Pod Name

Determine OpenShift Namespace that pod under inspection resides in (user should know this information).

Determine OpenShift node name or IP Address that is hosting pod under inspection:

```shell
# Ran from: Local machine with OpenShift Kubeconfig and Admin User privileges
$ oc get nodes --no-headers
ip-10-0-1-101.us-east-2.compute.internal   Ready    master,worker   3d18h   v1.20.0+bd9e442   10.0.1.101        <none>        Red Hat Enterprise Linux CoreOS 47.84.202009041830-0 (Ootpa)   4.18.0-193.28.1.el8_2.x86_64   cri-o://1.19.0-34.rhaos4.6.git0baa9a2.el8
ip-10-0-2-202.us-east-2.compute.internal   Ready    worker          3d18h   v1.20.0+bd9e442   10.0.2.202        <none>        Red Hat Enterprise Linux CoreOS 47.84.202009041830-0 (Ootpa)   4.18.0-193.28.1.el8_2.x86_64   cri-o://1.19.0-34.rhaos4.6.git0baa9a2.el8
ip-10-0-3-33.us-east-2.compute.internal    Ready    worker          3d17h   v1.20.0+bd9e442   10.0.3.33         <none>        Red Hat Enterprise Linux CoreOS 47.84.202009041830-0 (Ootpa)   4.18.0-193.28.1.el8_2.x86_64   cri-o://1.19.0-34.rhaos4.6.git0baa9a2.el8
ip-10-0-4-44.us-east-2.compute.internal    Ready    worker          3d16h   v1.20.0+bd9e442   10.0.4.44         <none>        Red Hat Enterprise Linux CoreOS 47.84.202009041830-0 (Ootpa)   4.18.0-193.28.1.el8_2.x86_64   cri-o://1.19.0-34.rhaos4.6.git0baa9a2.el8
```

Determine pod name from Namespace and Node Name:

```shell
# Ran from: Local machine with OpenShift Kubeconfig and Admin User privileges
$ oc get pods -n "$SELECTED_NAMESPACE" --field-selector=status.phase=Running -o json | jq -r --arg NODE "${SELECTED_NODE}" '.items[] | select(.spec.nodeName==$NODE) | .metadata.name'
my-application-pod-daffead-dfkn
my-other-app-pod-daffead-dfkn
another-app-pod-daffead-dfkn
```

Save this information for later refernce during one of the methods of capture below. Each of the following
variables are referenced in the examples below:

```shell
$ export POD_NAME=my-application-pod-daffead-dfkn
$ export SELECTED_NAMESPACE=my-app-namespace
$ export SELECTED_NODE=ip-10-0-4-44.us-east-2.compute.internal
```

---

## Run `tcpdump` directly from OpenShift Node

SSH to Selected OpenShift Node:

```shell
$ ssh -i ssh-keys/my-openshift-cluster.key.pem.pub core@"$SELECTED_NODE"
```

Start OpenShift toolbox Container:

```shell
$ toolbox
```

Set the required `nsenter` parameters for reaching the pod's interface for inspection:

```shell
# Ran from: OpenShift Debug Pod Shell Session (runs as Root by default)
# Retrieve selected pod crictl ID Number
$ pod_id=$(chroot /host crictl pods --namespace ${SELECTED_NAMESPACE} --name ${POD_NAME} -q)

# Retrieve selected pod network namespace path
$ ns_path="/host/$(chroot /host bash -c "crictl inspectp $pod_id | jq '.info.runtimeSpec.linux.namespaces[]|select(.type==\"network\").path' -r")"

# Set 'nsenter' parameters to 
$ NSENTER_PARAMS="--net=${ns_path}"
```

Determine the selected pod's available network interfaces and select applicable device (`eth0` is usually a good default to choose):

```shell
# Ran from: OpenShift Debug Pod Shell Session
$ nsenter $NSENTER_PARAMS -- tcpdump -D
```

(Optional) Determine and set additional `tcpdump` parameters for filtering:

```shell
# Example sets monitoring for ssh and application port connections only
# as well as with the most verbose output.
$ export TCPDUMP_EXTRA_PARAMS="port 22 or port 8443 -vvvXx"
```

Start `tcpdump` capture (will run until process is terminating):

> _**Note**: This example saves the `tcpdump` packet capture to `/host/var/tmp/${POD_NAME}_$(date +\%d_%m_%Y-%H_%M_%S-%Z).pcap` for retrieval from the node later._

```shell
# Ran from: OpenShift Debug Pod Shell Session
nsenter $NSENTER_PARAMS -- tcpdump -nn -i ${INTERFACE} -w /host/var/tmp/${POD_NAME}_$(date +\%d_%m_%Y-%H_%M_%S-%Z).pcap ${TCPDUMP_EXTRA_PARAMS}
```

Retrieve the `tcpdump` packet capture file from the OpenShift node under inspection:

```shell
# Ran from: Local machine with OpenShift Kubeconfig and Admin User privileges
$ scp core@"${SELECTED_NODE}":/host/var/tmp/<tcpdump_capture_filename>.pcap <local_machine/file_path/file_name>.pcap
```

---

## Run `tcpdump` from OpenShift managed **_debug pod_**

Deploy OpenShift debug pod to selected node:

```shell
# Ran from: Local machine with OpenShift Kubeconfig and Admin User privileges
$ oc debug node/"${SELECTED_NODE}"
```

Set the required `nsenter` parameters for reaching the pod's interface for inspection:

```shell
# Ran from: OpenShift Debug Pod Shell Session (runs as Root by default)
# Retrieve selected pod crictl ID Number
$ pod_id=$(chroot /host crictl pods --namespace ${SELECTED_NAMESPACE} --name ${POD_NAME} -q)

# Retrieve selected pod network namespace path
$ ns_path="/host/$(chroot /host bash -c "crictl inspectp $pod_id | jq '.info.runtimeSpec.linux.namespaces[]|select(.type==\"network\").path' -r")"

# Set 'nsenter' parameters to 
$ NSENTER_PARAMS="--net=${ns_path}"
```

Determine the selected pod's available network interfaces and select applicable device (`eth0` is usually a good default to choose):

```shell
# Ran from: OpenShift Debug Pod Shell Session
$ nsenter $NSENTER_PARAMS -- tcpdump -D
```

(Optional) Determine and set additional `tcpdump` parameters for filtering:

```shell
# Example sets monitoring for ssh and application port connections only
# as well as with the most verbose output.
$ export TCPDUMP_EXTRA_PARAMS="port 22 or port 8443 -vvvXx"
```

Start `tcpdump` capture (will run until process is terminating):

> _**Note**: This example saves the `tcpdump` packet capture to `/host/var/tmp/${POD_NAME}_$(date +\%d_%m_%Y-%H_%M_%S-%Z).pcap` for retrieval from the node later._

```shell
# Ran from: OpenShift Debug Pod Shell Session
nsenter $NSENTER_PARAMS -- tcpdump -nn -i ${INTERFACE} -w /host/var/tmp/${POD_NAME}_$(date +\%d_%m_%Y-%H_%M_%S-%Z).pcap ${TCPDUMP_EXTRA_PARAMS}
```

Retrieve the `tcpdump` packet capture file from the OpenShift node under inspection:

```shell
# Ran from: Local machine with OpenShift Kubeconfig and Admin User privileges
$ oc cp "${SELECTED_NODE}":/host/var/tmp/<tcpdump_capture_filename>.pcap <local_machine/file_path/file_name>.pcap
```

---

## Run `tcpdump` via ksniff

Install [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/):

```shell
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Update `$PATH` environment variable with krew and **_restart shell session_**:

```shell
$ export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

Install `ksniff` via krew:

```shell
$ kubectl krew install sniff
```

Start packet capture using `ksniff` on selected pod:

```shell
$ oc sniff -p "$POD_NAME" -n "$SELECTED_NAMESPACE" -o ${POD_NAME}_$(date +\%d_%m_%Y-%H_%M_%S-%Z).pcap
```

End capture via **_CRTL+C_**



---

[A001]: https://access.redhat.com/bounce/?externalURL=https%3A%2F%2Fgithub.com%2Feldadru%2Fksniff