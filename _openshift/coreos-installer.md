---
title:  "CoreOS Installer"
layout: default
nav_order: 3
---

# ![openshift](https://github.com/natemollica-nm/devops/assets/57850649/34711e45-1e7f-40d6-a900-309195d4a26f){:width="10%"} CoreOS Installer
{: .no_toc }

CoreOS Installer concepts and info as it relates to OpenShift.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## CoreOS Installer

[coreos-installer][coreos-installer]: An installation assistance program that aims to assist with
Federoa CoreOS (FCOS) and RedHat Enterprise Linux CoreOS (RHCOS).


## Run from a container

You can run `coreos-installer` from a container. You’ll need to bind-mount `/dev` and `/run/udev`, 
as well as a data directory if you want to access files in the host. 

```shell
$ sudo podman run \
    --pull=always \
    --privileged \
    --rm \
    -v /dev:/dev \
    -v /run/udev:/run/udev \
    -v .:/data \
    -w /data \
    quay.io/coreos/coreos-installer:release \
    "${COREOS_INSTALLER_PARAMETERS}"
```

### CoreOS Installer Script Usage

When working on platforms that don't have architecture builds for `coreos-installer`, you can
implement the container run as a function within a script.

```shell
## Podman image used to run coreos-installer as command from container
## Ref: https://coreos.github.io/coreos-installer/getting-started/#run-from-a-container
coreos-installer() {
  podman run --privileged --rm -v "$PWD":/data -w /data quay.io/coreos/coreos-installer:release "$@"
}
```


## References

1. [CoreOS Installer Docs][coreos-installer]


[coreos-installer]: https://coreos.github.io/coreos-installer/