---
title:  Envoy
layout: default
---

![Envoy](../assets/envoy.png){:width="25px"}

---
## Envoy Building Blocks

* **Listeners**
* **Routes**
* **Clusters**
* **Endpoints**

Envoy is a solution toward reducing developer concerns in troubleshooting microservice
networking issues. It aims to remove the networking concerns of microservices out of
the purview of the application stack itself, so developers can focus more on developing
application specific functionalities.

Envoy runs a self-contained process that runs alongside every application (Sidecar Pattern).
The collection of centrally configured Envoy processes makes up what is known as the
**_transparent mesh_**. Applications can send their requests to virtually addressed hosts
instead of actual IP addressed hosts, allowing the networking stack to become "transparent".
The application no longer has to be aware of nor bear the burden of network routing and delegates
this to Envoy.

---

## Architecture

![EnvoyArch](/assets/envoy-arch.png)

## Benefits

**Language Agnostic**: Envoy works with any programming language (Go, Java, C++, etc.)

**Deployment/Upgrade Transparent**: By removing the network stack from applications, any
changes regarding microservice networking (deployments/upgrades) are transparently delegated
to Envoy and removes the requirement of upgrading application networking libraries themselves.

## Overview

### L3/L4 Filter Architecture

Envoy uses a **_pluggable filter chain_** to write filters to perform various TCP/UDP tasks.
It acts as a L3/L4 network proxy that makes decisions based on IP addresses and ports.

### L7 Filter Architecture

Envoy supports HTTP L7 filters where HTTP filters are inserted into HTTP connection management
subsystems to perform tasks such as buffering, rate limiting, routing/forwarding, etc.

**HTTP/2**: Envoy transparently supports HTTP1.1 and HTTP/2 via proxying in both directions. This
is beneficial as it allows **_any combination_** of HTTP1.1 and HTTP/2 client and server communication.
This is good for concerns surrounding transitions from legacy to newer applications.

**HTTP Routing**: Envoy supports REST API application routing and redirection based on path, authority,
content type, and runtime values. This makes Envoy valuable for use as a front/edge proxy for building
API Gateways to the service-mesh deployment pattern.

### gRPC

Envoy supports all HTTP/2 features required to be used as the routing and load balancing substrate for
gRPC requests and responses.

> gRPC is an open-source remote procedure call (RPC) system that uses HTTP/2 for transport and protocol
buffers as the interface description language (IDL), and that provides features such as authentication,
bidirectional streaming, and flow control, blocking/nonblocking bindings, and cancellation and timeouts.

### Service Discovery | xDS Dynamic Configuration

Envoy can be configured to run as a static or dynamic proxy. In situations where statically creating
Envoy configurations is impractical, Envoy leverages a feature called **xDS** that is used to
dynamically configure Envoy through the network. This feature configures Envoy information regarding
hosts, HTTP cluster routing, listener sockets, and cryptographic material. This is all accomplished
dynamically at runtime and is reloadable at runtime.

### Health Checking

Envoy supports a health checking subsystem that performs active health checks of upstream service
clusters that play a part in internal load balancing features.

Envoy also supports passive health checking via its **_outlier detection subsystem_**.

### Load Balancing

Envoy supports:
* Automatic Retries
* Circuit Breaking
* Global Rate Limiting
* Request Shadowing (Traffic Mirroring)
* Outlier Detection
* Request Hedging

### TLS Termination

Envoy enables TLS termination (mTLS) between all services in the mesh deployment model.

### Observability

Envoy leverages `statsd` as the sink for all its subsystems, and is expandable for other statistics
providers if needed.

### HTTP/3 (Alpha)

From Envoy v1.19.0 on, HTTP/3 upstream and downstream translation between HTTP versions 1.1, 2, and
3 are supported.