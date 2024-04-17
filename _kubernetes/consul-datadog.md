---
title:  Datadog Consul Integration
layout: default
---

## Consul on Kubernetes | Datadog DogstatsD Configuration

![image](https://github.com/natemollica-nm/devops/assets/57850649/ce673115-2bd6-47fc-b384-e128b5f4494f)

For user running **_Consul versions less than 1.16.x_**, Datadog DogstatsD integration requires manual 
configuration for collecting Consul Server specific runtime metrics. This article aims to outline the 
manual steps required to configure a consul-k8s environment for metrics collection via [Datadog DogstatsD](https://docs.datadoghq.com/developers/dogstatsd/?tab=kubernetes).

For users running **_Consul versions >= 1.16.x_**, the Consul helm chart provides [datadog-specific overrides](https://github.com/hashicorp/consul-k8s/blob/02b8d33719e1958f5b272ef685b611414093a974/charts/consul/values.yaml#L663-L785)
that make automating this process easier based on runtime Consul deployment configurations. This method 
of implementation is recommended for these users in this case. See the Configure Datadog metrics collection
for [Consul on Kubernetes: DogstatsD](https://developer.hashicorp.com/consul/docs/k8s/deployment-configurations/datadog#dogstatsd) documentation for more information regarding this feature.

### Consul Server Helm Overrides

* `server.extraLabels`: Configures [Datadog Unified Service Tagging](https://docs.datadoghq.com/getting_started/tagging/unified_service_tagging/?tab=kubernetes)
    * With these three tags, you can:
        * Identify deployment impact with trace and container metrics filtered by version
        * Navigate seamlessly across traces, metrics, and logs with consistent tags
        * View service data based on environment or version in a unified fashion
* `server.annotations`:
    * `'ad.datadoghq.com/consul.logs'`:
        * Configures tags for enabling Kubernetes [log collection for Consul](https://docs.datadoghq.com/integrations/consul/?tab=containerized#log-collection)
    * `'ad.datadoghq.com/consul.metrics_exclude'`: When `true`, excludes metric collection from the entire consul-server pod.
        * Required when using DogstatsD, Consul is emitting metrics _**to**_ Datadog (vice Datadog reaching out to scrape Consul)
        * Required as Datadog may attempt to reach out to Consul to scrape prometheus style metrics when `prometheus_retention_time` is set to `true`
        * [DogstatsD](https://docs.datadoghq.com/developers/dogstatsd/?tab=kubernetes), [Consul Integration](https://docs.datadoghq.com/integrations/consul/?tab=containerized), and [Openmetrics Prometheus](https://docs.datadoghq.com/containers/kubernetes/prometheus/?tab=kubernetesadv2) Metrics by design share the same metric name syntax for collection, and would therefore cause a conflict. The [consul.py](https://github.com/DataDog/integrations-core/blob/07c04c5e9465ba1f3e0198830896d05923e81283/consul/datadog_checks/consul/consul.py#L55-L61) integration source code prohibits the enablement of more that one integration at a time.
* `server.extraConfig["telemetry"]`:
    *  `dogstatsd_tags`: This provides a list of global tags that will be added to all telemetry packets sent to DogStatsD.
    *  `dogstatsd_addr`: This provides the address of a DogStatsD instance in the format `host:port` to enable Consul to send various telemetry information to that instance for aggregation.
    *  `disable_hostname`: When `true`, stops hostname prepending for Datadog gauge-type metrics (better for troubleshooting)
    *  `enable_host_metrics`: Enables reporting of host metrics about system resources (more helpful info for troubleshooting)
    *  `prefix_filter`: Enables emitting [Consul Server Workload](https://developer.hashicorp.com/consul/docs/agent/telemetry#server-workload) metrics (v1.12.x and above)

_**Datadog Agent Running in Kube Cluster (namespace: datadog)**_
```yaml
server:
  enabled: true
  replicas: 3
  extraLabels:
    'tags.datadoghq.com/version': '1.6.2-ent'
    'tags.datadoghq.com/env': 'vault'
    'tags.datadoghq.com/service': 'consul-server'
  annotations:
    'ad.datadoghq.com/consul.logs': '[{"source": "consul","consul_service": "consul-server"}]'
    'ad.datadoghq.com/consul.metrics_exclude': true
  extraConfig: |
    {
      "telemetry": {
        "prometheus_retention_time": "10s",
        "disable_hostname": true,
        "prefix_filter": ["+consul.rpc.server.call"],
        "enable_host_metrics": true,
        "dogstatsd_tags": ["source:consul","service:consul-server"],
        "dogstatsd_addr": "datadog-agent.datadog.svc.cluster.local"
      }
    }
```

_**Datadog Agent Running in Outside Kube Cluster**_

> _**Note**: IP/Hostname is routable from internal Kubernetes Cluster networking_
```yaml
# Example DNS Name: "datadog-agent.my-domain.com:8125"
server:
  enabled: true
  replicas: 3
  extraLabels:
    'tags.datadoghq.com/version': '1.6.2-ent'
    'tags.datadoghq.com/env': 'vault'
    'tags.datadoghq.com/service': 'consul-server'
  annotations:
    'ad.datadoghq.com/consul.logs': '[{"source": "consul","consul_service": "consul-server"}]'
    'ad.datadoghq.com/consul.metrics_exclude': true
  extraConfig: |
    {
      "telemetry": {
        "prometheus_retention_time": "10s",
        "disable_hostname": true,
        "prefix_filter": ["+consul.rpc.server.call"],
        "enable_host_metrics": true,
        "dogstatsd_tags": ["source:consul","service:consul-server"],
        "dogstatsd_addr": "172.20.180.10:8125"
      }
    }
```

### Helm Upgrade

The above Helm value overrides demonstrate how to configure consul-k8s consul-server pod metrics emission to a Datadog Agent leveraging DogstatsD.

After applying the following additional override settings, the users are then expected to apply the updates using either the [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) or [consul-k8s](https://developer.hashicorp.com/consul/docs/k8s/k8s-cli#upgrade) upgrade commands to update their environments.