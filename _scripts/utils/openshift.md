---
layout: default
title: "OpenShift"
parent: utils
grand_parent: Scripts
nav_order: 2
---

# Random Utilities
{: .no_toc }

Openshift specific script utilities.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## OpenShift Cluster Assisted Installer (v2): Offline API Token Refresh

Prerequisites

* jq
* OpenShift [Red Hat Hybrid Cloud Account](https://console.redhat.com/)

* Visit [OpenShift Cluster Manager API Token](https://console.redhat.com/openshift/token/show) page and obtain offline token value:

![redhat-offline-api-token](https://github.com/natemollica-nm/devops/assets/57850649/39788dc3-f3b1-45c7-bd7e-de635fdac7e1){:width="100%"}

Configure `OFFLINE_API_TOKEN` environment variable:

```shell
export OFFLINE_API_TOKEN="eyJhbG..."
```

## `openshift-offline-token-refresh.sh`

```shell
export API_TOKEN=$( \
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

## Usage:

```shell
# Refresh token
$ source openshift-offline-token-refresh.sh
```

```shell
# Verify API Access
$ curl -s https://api.openshift.com/api/assisted-install/v2/component-versions -H "Authorization: Bearer ${API_TOKEN}" | jq
{
  "release_tag": "v2.11.3",
  "versions": {
    "assisted-installer": "registry.redhat.io/rhai-tech-preview/assisted-installer-rhel8:v1.0.0-211",
    "assisted-installer-controller": "registry.redhat.io/rhai-tech-preview/assisted-installer-reporter-rhel8:v1.0.0-266",
    "assisted-installer-service": "quay.io/app-sre/assisted-service:78d113a",
    "discovery-agent": "registry.redhat.io/rhai-tech-preview/assisted-installer-agent-rhel8:v1.0.0-195"
  }
}
```
