---
title:  Listeners
layout: default
---
## Envoy Listeners

Envoy exposes listeners that are named network locations, either an IP address and a port or a
Unix Domain Socket path. Envoy receives connections and requests through listeners.

The Envoy Listener Subsystem:
* Listener Filters
* Filter Chain Matching
* HTTP Inspector Listener Filter
* Original Destination Filter
* Original Source Filter
* Proxy Protocol Filter
* TLS Inspector Filter

**Static Listener**

```yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 10000
      filter_chains: [{}]
```

Referencing the above static listener configuration:
* Name: `listener_0`
* Listener Address: `0.0.0.0`
* Listener Port: `10000`

Envoy will listen on `0.0.0.0:10000` for **_incoming requests_**.

The only **_required listener configuration_** is the Listener Address.

### Listener Filters

Listener Filters are **_OPTIONAL_** filters that are included when defining a listener itself.
These filters are processed **_before_** the network level filters, and they're useful for manipulating
connection metadata for later filter or cluster process action behavior.

These filters operate on newly accepted sockets and can either stop or continue the connection execution process
in Envoy. This can help in controlling how Envoy passes the connection to the HTTP Inspection Filter when trying
to determine the underlying HTTP Protocol (HTTP/1.1 or HTTP/2), and subsequently alter which network
filter chain to use.

**_Important_**: Listener Filters are not the same as **_network filter chains_** or **_L3/L4 filters_**.

### Listener Filter Chain Matching

Envoy implements the concept of filter chain matching which allows for the **_specification of criteria_** for
**_selecting filter chains for a listener_**.

Envoy accomplishes this by extracting data from the inbound packet that contacts the listener.

**Envoy Listener Filter Chain Matching Order**
1. Destination Port (`use_original_dst` only)
2. Destination IP Address<sup>*<sup>
3. Server Name or SNI for TLS Protocol<sup>*<sup>
4. Transport Protocol
5. Application Protocols (ALPN for TLS Protocol)
6. Connection Source IP Address<sup>**</sup>
7. Source Type (any, local, external network, etc.)
8. Source IP Address<sup>*<sup>
9. Source Port

<sup>* _Allows for range or wildcard usage_</sup><br/>
<sup>** _Not to be confused with Source IP Address, as the Source IP can be overridden by the inbound listener filter if configured to do so._</sup>


### Original Destination Listener Filter

Envoy `typed_config`: `"@type": type.googleapis.com/envoy.extensions.filters.listener.original_dst.v3.OriginalDst`

The Envoy Original Destination filter reads the `SO_ORIGINAL_DST`<sup>*</sup> socket option. This is used
in connection with a cluster that has the `ORIGINAL_DST` type.

When using the`ORIGINAL_DST` Cluster Type, requests get forwarded to upstream hosts as addressed by
the redirection metadata without using any host discovery. This means that defining any endpoints in the
cluster is not required as the endpoint is derived from the original packet and Envoy does not select
it using its internal load balancing mechanisms.

Using `ORIGINAL_DST` cluster types allows Envoy to be used as a generic proxy that forwards all requests to the original
destination.

<sub>* `SO_ORIGINAL_DST`: Option that is set when a connection has been **_redirected by an iptables `REDIRECT` or `TPROXY` (`transparent` option set) target._**</sub>

```yaml
...
listener_filters:
- name: envoy.filters.listener.original_dst
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.listener.original_dst.v3.OriginalDst
...
clusters:
  - name: original_dst_cluster
    connect_timeout: 5s
    type: ORIGNAL_DST
    lb_policy: CLUSTER_PROVIDED
```

### Original Source Listener Filter

Envoy `typed_config`: `"@type: type.googleapis.com/envoy.extensions.filters.listener.original_src.v3.OriginalSrc"`

The Envoy Original Source filter replicates the downstream (host making connection to Envoy) remote address of the connection
on the upstream (host receiving requests from Envoy) side of Envoy.

~> In progress ...


### HTTP Inspector Listener Filter

Envoy `typed_config`: `"@type": type.googleapis.com/envoy.extensions.filters.listener.http_inspector.v3.HttpInspector `

The `http_inspector` filter resides under the `listener_filters` field to inspect the connection and determine the
application protocol.

Purpose(s):
* Detects incoming application request protocol type (HTTP or non-HTTP)
* Detects HTTP protocol type (HTTP/1.x or HTTP/2)

```yaml
...
    listener_filters:
    - name: envoy.filters.listener.http_inspector
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.listener.http_inspector.v3.HttpInspector
    filter_chains:
    - filter_chain_match:
        application_protocols: ["h2"]
      filters:
      - name: my_http2_filter
        ... 
    - filter_chain_match:
        application_protocols: ["http/1.1"]
      filters:
      - name: my_http1_filter
...
```
<sub>**_Example_**: If the HTTP protocol is HTTP/2 (h2c), Envoy matches the first network filter chain (starting with my_http2_filter).</sub>