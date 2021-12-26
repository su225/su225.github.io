---
title: "Understanding Istio configuration and envoy config generation"
date: 2021-12-19T20:00:17+05:30
draft: true
tags: [istio,envoy]
---

Istio is one of the most popular service meshes out there with lots of features.
However, configuring the mesh can get complex very soon. In this post, I will
explore configuring bottom-up, right from configuring Envoyproxy to how the Istio API
is translated to Envoy configuration.<!--more-->

{{< toc >}}

## Envoyproxy
[Envoyproxy](https://www.envoyproxy.io/) is a cloud-native L4-L7 proxy designed from
group up for dynamic configuration. Like any other proxy, it sits between the client
and the actual server as follows

{{< mermaid >}}
    graph LR
        client
        envoyproxy
        subgraph servers
            server-1
            server-2
        end
        client --> envoyproxy
        envoyproxy --> server-1
        envoyproxy --> server-2

{{< /mermaid >}}
Hence, configuring Envoyproxy mainly consists of two parts
1. Server-side configuration (configuring `listeners`, `filter_chains` and `filters`)
2. Client-side configuration (configuring `clusters`)

### Experiment setup
I will create a simple service and explore Envoy is configured for proxying. There is particular
emphasis on configuring TLS. I will illustrate passthrough, simple and mutual TLS modes.

### Server-side configs
Server side configuration involves specifying
1. **Where to receive traffic?** - Listener configuration (under the covers - Linux Sockets API)
    * Specify IP address and port or the Unix Domain Socket
    * Specify whether to bind the socket to IP and port
2. **What to do with the traffic received?** - This is the part where the chain of transformations
   on the traffic, which service to send traffic to are configured.
    * `listener_filters` are used to sniff the raw traffic to determine their properties. An example
       is `tls_inspector` which checks for TLS traffic and extracts parameters to be used for matching
       filter chains. This is crucial for various reasons
       * Implementing `PERMISSIVE` mTLS mode
       * Configuring multiple hostnames on the same port. SNI (Server Name Indication) is used to
         distinguish between different hostnames and process them differently.
    * `filter_chains` - they determine the chain of transformations and routing decisions based on
      the protocol. A single `listener` can be configured with multiple filter chains. The 
      `filter_chain_match` criteria is used to pick the single filter chain. **SNI is one such criteria**
    * `filter` - a `filter_chain` consists of multiple `filter`s, each doing a specific task. Here
      are some important ones
      * `tcp_proxy` - as the name suggests, forwards TCP traffic to the upstream hosts
      * `http_connection_manager` - decodes HTTP traffic and provides various HTTP specific features.
        This filter also has a sub-filter chain to process HTTP traffic. The entire list can be found
        [here](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters).
        * `router_filter` - route based on HTTP authority/host, header and paths, add/remove headers
        * `jwt_authn` - configure JWT authentication (validate claims, issuers, verify signature). In
          Istio, this is specified as `RequestAuthentication`.
        * `ext_authz` - configure External authorization backend. For each request, Envoy calls the
          configured backend to make authorization decisions. Providers like Open Policy Agent can
          be integrated. 

In picture,

{{< mermaid >}}
graph TD
    port8080(8080/TCP)
    port8443(8443/TCP)

    subgraph HTTP
        subgraph HTTP
            raw0(raw_transport_socket) --> hcm0(http_connection_manager)
        end
    end

    subgraph HTTPS
        subgraph TLS-terminated: foo.com
            tls1(tls_downstream_socket) --> hcm1(http_connection_manager)
        end

        subgraph TLS-passthrough: bar.com
            raw2(raw_transport_socket) --> tcp_proxy
        end
    end 

    port8080 --> raw0

    port8443 --> tls_inspector
    tls_inspector -->|sni:foo.com| tls1
    tls_inspector -->|sni:bar.com| raw2
{{< /mermaid >}}

Here is a sample server-side configuration
TODO: fill this one
```yaml
socket_address:
    address: 0.0.0.0
    port: 8080
```

From Istio perspective, the following points are important
* For the gateways, server-side configuration is defined by `Gateway` resource. The filter chain
  generated depends on the port protocol.
* For the sidecars, it depends on the `workload_port` they are listening on. For inbound ports, 
  the address would be `0.0.0.0`. For the outbound traffic, address would be `service VIP` and the port would be the `service_port`.

### Client-side configs
Once traffic transformation is performed and we reach a place where traffic has to be forwarded
to an upstream host, we have to pick which endpoint to send traffic to, the protocol specific
settings (something like use HTTP/2), connection pool settings and so on. For each `service` or
a `version` of the service (called `subset` in Istio), an envoy `cluster` is configured. The
`cluster` consists of `endpoints` which could be either IP addresses or DNS hostnames.

The full list of configuration options can be seen [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto)

> **NOTE**: If DNS names are specified then Envoy asynchronously resolves DNS 
> names according to the specified DNS settings and the cache-TTL.

#### Client-side Load-balancing
This is a crucial feature to enable intelligent traffic routing in Istio. Here, the set of endpoints
for a service are classified according to various user-specified labels denoting version of the service
and the locality information like region, zone, network, cluster.

1. **Simple load-balancing** - Simple load-balancing policies are called so because as they are 
   simple and quick to setup. There are few knobs to turn.
2. **Locality and priority-based load-balancing** - Locality and version information can be added
   to divide the set of endpoints for a service into a subsets based on locality or some other user
   specified labels and route traffic within the subset. See [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#config-endpoint-v3-localitylbendpoints)
   for more information.

{{< mermaid >}}
graph LR
    envoy --> endpoint-0
    envoy --> endpoint-1
    envoy --> endpoint-2

    subgraph cluster: foo.com
        subgraph us-east
            endpoint-0
            endpoint-1
        end

        subgraph eu-central
            endpoint-2
        end
    end
{{< /mermaid >}}

## Understanding Istio traffic capture
Istio treats gateway and sidecar envoys differently while configuring. Gateway envoy is a standalone
envoy instance which listens on the ports specified in the `Gateway` configuration applicable to it.
By default, the sidecar envoys capture all the traffic arriving at a workload hosting a service and
route to `VirtualInbound` listener. All outgoing traffic is also captured and routed to `VirtualOutbound`
listener. These listeners in turn send it over to the actual listeners processing the traffic.

As of writing this, Istio uses the good-old `iptables` for traffic capture. There are two ways to
configure these rules in the Kubernetes environment.
1. An init-container `istio-init` is injected through the webhook which runs `iptables`.
2. (alpha) Istio CNI plugin chained to the main CNI plugin sets up the rules in the pod's network namespace

More reading
* [Networking changes in Istio 1.10](https://istio.io/latest/blog/2021/upcoming-networking-changes/)

## Exploring config generation in Istio

### Experiment setup
TODO: fill this section

### Configuration

#### Exposing the app through the gateway
In this section, I will illustrate how `Gateway` configuration affects Envoy configuration
at the gateways deployed.
TODO: fill this section

#### Configuring routes

#### Configuring subsets for canarying

#### Configuring mutual TLS

#### Configuring JWT authentication

#### Configuring authorization policy