---
title: "Supporting HTTP/3 at the ingress gateway in Istio"
date: 2021-10-23T20:00:17+05:30
draft: false
---

A few months ago I contributed experimental HTTP/3 support at the Istio ingress gateway
([PR](https://github.com/istio/istio/pull/33817)). For the HTTPS gateway
servers which terminate the TLS at the gateway, it automatically adds a mirror HTTP/3
listener over QUIC provided that there is an open UDP port. If there is an HTTPS server
on TCP port 443, then an HTTP/3 server is automatically created on UDP port 443 as
illustrated in the diagram below. All you need to do is to flip a switch on pilot and
open UDP ports.

TODO: Add diagram

## Prerequistes
HTTP/3 over QUIC is so new as of writing this. QUIC was standardized this May end and HTTP/3 is not yet
standardized (There is an IETF draft [here](https://datatracker.ietf.org/doc/html/draft-ietf-quic-http-34)). 
So creating custom builds, turning on alpha features are expected. I'm running Linux.

### Kubernetes
This feature requires the supporting both TCP and UDP on the same port at the gateway. By
default for Kubernetes services of type `LoadBalancer` this is not allowed. Since Kubernetes
1.20 there is alpha support. As documented [here](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancers-with-mixed-protocol-types), it can be turned on with `MixedProtocolLBSupport`
feature gate. 

For this demo, I use Kind to provision a cluster with the following config
{{< highlight yaml "hl_lines=4" >}}
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  MixedProtocolLBService: true # Very important
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta2
  kind: ClusterConfiguration
  metadata:
    name: config
  apiServer:
    extraArgs:
      "service-account-issuer": "kubernetes.default.svc"
      "service-account-signing-key-file": "/etc/kubernetes/pki/sa.key"
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
      endpoint = ["http://kind-registry:5000"]
{{< / highlight >}}

For the load balancer support, I'm using [MetalLB](https://metallb.universe.tf/) with the following configuration.
For setup and usage instructions, please refer to their documentation.

### cURL
Testing the setup requires support for `--http3` flag. If it is not supported out of the box,
then you have to build it with Cloudflare Quiche support as documented [here](https://github.com/curl/curl/blob/master/docs/HTTP3.md).


### Istio
As of writing this Istio 1.12 is not yet released. So the pilot and proxy images should be
built from the master branch.

## Setup
Demo setup consists of 
1. Setting up Istio
2. Deploying httpbin application
3. Setting up gateway TLS certificates
4. Configuring ingress gateway

### Setting up Istio
There are two important things
1. Set `PILOT_ENABLE_QUIC_LISTENERS` environment variable to `true` on **Istiod** to turn on generating HTTP/3 mirror listener on the gateway
2. Exposing both 443/UDP and 443/TCP - **same port, different transport protocol**

Here is an example `IstioOperator` spec. 
{{< highlight yaml "hl_lines=23-29 36" >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: install
spec:
  # Replace HUB and TAG to point to your
  # image repository and the custom image tag
  hub: localhost:5000
  tag: master
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          # Both HTTPS and HTTP/3 must have
          # the same port number. Here it
          # is 443/TCP and 443/UDP
          - name: https
            port: 443
            targetPort: 8443
          - name: http3
            port: 443
            targetPort: 8443
            protocol: UDP
  values:
    pilot:
      # This is required to create mirror QUIC
      # listeners for TLS-terminated HTTPS listeners
      # on the gateway
      env:
        PILOT_ENABLE_QUIC_LISTENERS: true
{{< / highlight >}}
Then install Istio
{{< highlight bash >}}
$ istioctl install -f istio.yaml -y
{{< / highlight >}}

### Deploying httpbin
I am using the sample httpbin provided in the main Istio repository ([Link](https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml)). Deploy it with good old `kubectl`. Make sure that Istio sidecars are injected.
You can turn on istio sidecar injection at the namespace level by labeling it `istio-injection=enabled`
or if you are using revisions (please use it to make upgrades smoother), then it is `istio.io/rev=<revision-name>`.

{{< highlight bash >}}
$ kubectl create namespace httpbin
$ kubectl label namespace httpbin istio-injection=enabled
$ kubectl -n httpbin apply -f \
  https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
{{< / highlight >}}

### Setting up TLS certificates for the gateway
HTTP/3 is over TLS only. In fact, using TLS is baked into QUIC, the underlying transport protocol
as described in [RFC-9000](https://datatracker.ietf.org/doc/html/rfc9000) and [RFC-9001](https://datatracker.ietf.org/doc/html/rfc9001).
So we need to generate and install certificates. For the demo, I'm using self-signed certificates.

1. Parameters for the certificate (Like SAN). Save it as `httpbin.cfg`
{{< highlight cfg >}}
[req]
default_bits       = 2048
prompt             = no
distinguished_name = req_distinguished_name
req_extensions     = san_reqext

[ req_distinguished_name ]
countryName         = XX
stateOrProvinceName = YY
organizationName    = QuicCorp

[ san_reqext ]
subjectAltName      = @alt_names

[alt_names]
DNS.0   = httpbin.quic-corp.com
{{< / highlight >}}

2. Generate TLS certificates and keys with the following script. Here, I'm using `openssl`. You can
use other tools like `cfssl` as well.
{{< highlight shell >}}
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:4096 -subj \
    "/C=XX/ST=YY/O=QuicCorp" -keyout quiccorp-ca.key -out quiccorp-ca.crt

openssl req -out httpbin.csr -newkey rsa:2048 -nodes \
    -keyout httpbin.key -config httpbin.cnf

openssl x509 -req -days 365 -CA quiccorp-ca.crt -CAkey quiccorp-ca.key \
    -set_serial 0 -in httpbin.csr -out httpbin.crt \
    -extfile httpbin.cnf -extensions san_reqext
{{< / highlight >}}

3. Install the certificates
{{< highlight shell >}}
$ kubectl -n istio-system create secret tls httpbin-cred \
    --key=httpbin.key --cert=httpbin.crt
{{< / highlight >}}

### Istio configuration
Remember, HTTP/3 listener is generated **automatically** for TLS-terminated HTTPS listener. Currently,
specifying that a port is HTTP/3-only is not yet supported. This is because HTTP/3 support is not widespread
yet and the most common use case is for the clients like Google Chrome to become aware of HTTP/3 support 
with `alt-svc` HTTP header in the response when they first use HTTP/1.1 or HTTP/2 over TCP.

Configure the gateway. Notice that there is no explicit HTTP/3 related configuration
{{< highlight yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - httpbin.quic-corp.com
    tls:
      mode: SIMPLE
      credentialName: httpbin-cred
{{< / highlight >}}

Configure the route
{{< highlight yaml >}}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin-route
spec:
  hosts:
  - httpbin.quic-corp.com
  gateways:
  - httpbin-gateway
  http:
  - name: httpbin-default-route
    route:
    - destination:
        host: httpbin.httpbin.svc.cluster.local
        port: 
          number: 8000
{{< / highlight >}}

Once these routes are applied, check if two listeners are created
{{< highlight shell "hl_lines=3-4" >}}
$ istioctl proxy-config listeners istio-ingressgateway-6fdcddbb8b-9rpkx.istio-system
ADDRESS PORT  MATCH                      DESTINATION
0.0.0.0 8443  SNI: httpbin.quic-corp.com Route: https.443.https.httpbin-gateway.istio-system
0.0.0.0 8443  SNI: httpbin.quic-corp.com Route: https.443.https.httpbin-gateway.istio-system
0.0.0.0 15021 ALL                        Inline Route: /healthz/ready*
0.0.0.0 15090 ALL                        Inline Route: /stats/prometheus*
{{< / highlight >}}
Where did the other inbound listener come from? Let us dive deeper into the config.
Let us start with checking listener names
{{< highlight bash "hl_lines=4" >}}
$ istioctl proxy-config listeners istio-ingressgateway-6fdcddbb8b-9rpkx.istio-system \
    --address 0.0.0.0 --port 8443 -o json | jq -r '.[].name'
0.0.0.0_8443
udp_0.0.0.0_8443
{{< / highlight >}}
UDP?? Remember that QUIC uses UDP. So could this be related to QUIC? Let us dive deeper.
The output is huge and only the relevant part is shown here. So pipe it to a pager like `less`
{{< highlight json "hl_lines=3-5" >}}
{
  "transportSocket": {
      "name": "envoy.transport_sockets.quic",
      "typedConfig": {
          "@type": "type.googleapis.com/envoy.extensions.transport_sockets.quic.v3.QuicDownstreamTransport",
          "downstreamTlsContext": {
              "commonTlsContext": {
                  "tlsCertificateSdsSecretConfigs": [
                      {
                          "name": "kubernetes://httpbin-cred",
                          "sdsConfig": {
                              "ads": {},
                              "resourceApiVersion": "V3"
                          }
                      }
                  ],
                  "alpnProtocols": [
                      "h3"
                  ]
              },
              "requireClientCertificate": false
          }
      }
  }
}
{{< / highlight >}}
Well...The one with `udp` prefix is a QUIC listener!

## Demo time
First note down the address of the Ingress gateway
{{< highlight bash >}}
$ kubectl -n istio-system get svc
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                       AGE
istio-ingressgateway   LoadBalancer   10.96.112.53    172.18.200.1   15021:32534/TCP,443:31597/TCP,443:31597/UDP   79m
istiod                 ClusterIP      10.96.255.123   <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP         79m

$ export INGRESS_IP=172.18.200.1
{{< / highlight >}}

I have a custom build of curl called `qcurl` which supports sending HTTP/3 request with
`--http3` flag. `curl` here is the standard curl available in the software repositories.
For demo purposes, I'm skipping TLS certificate verification. Don't this if it is not a demo.

Let us first send HTTP/2 request
{{< highlight bash "hl_lines=9 17" >}}
$ curl -svk --http2 --resolve httpbin.quic-corp.com:443:$INGRESS_IP https://httpbin.quic-corp.com/headers
[ Output truncated ]
...
> GET /headers HTTP/2
> Host: httpbin.quic-corp.com
> user-agent: curl/7.76.1
> accept: */*
...
< HTTP/2 200 
< server: istio-envoy
< date: Wed, 27 Oct 2021 05:05:59 GMT
< content-type: application/json
< content-length: 601
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 1
< alt-svc: h3=":443"; ma=86400
< 
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.quic-corp.com", 
    "User-Agent": "curl/7.76.1", 
    "X-B3-Parentspanid": "819c2b2cab59bbe8", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "0462ac631afb5f84", 
    "X-B3-Traceid": "3fa44e3bbbcd21a3819c2b2cab59bbe8", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Envoy-Internal": "true", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/httpbin/sa/httpbin;Hash=97bf9c90d4a5b9bb8f5da3e825dfa34f04631400420649394741807a320aa0a1;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
{{< / highlight >}}
Yayyy! It is working. In particular notice this header
{{< highlight shell >}}
alt-svc: h3=":443"; ma=86400
{{< / highlight >}}
It indicates that HTTP/3 is supported. `h3` is the ALPN sent with the TLS handshake.
Clients supporting HTTP/3 can use this information to connect with QUIC for the future
connections. **Now, let us send an HTTP/3 request**
{{< highlight shell "hl_lines=14-15 20" >}}
$ qcurl -svk --http3 --resolve httpbin.quic-corp.com:443:$INGRESS_IP https://httpbin.quic-corp.com/headers
* Added httpbin.quic-corp.com:443:172.18.200.1 to DNS cache
* Hostname httpbin.quic-corp.com was found in DNS cache
*   Trying 172.18.200.1:443...
* Connect socket 5 over QUIC to 172.18.200.1:443
* Sent QUIC client Initial, ALPN: h3,h3-29,h3-28,h3-27
* Connected to httpbin.quic-corp.com () port 443 (#0)
* h3 [:method: GET]
* h3 [:path: /headers]
* h3 [:scheme: https]
* h3 [:authority: httpbin.quic-corp.com]
* h3 [user-agent: curl/7.78.0-DEV]
* h3 [accept: */*]
* Using HTTP/3 Stream ID: 0 (easy handle 0xa64260)
> GET /headers HTTP/3
> Host: httpbin.quic-corp.com
> user-agent: curl/7.78.0-DEV
> accept: */*
> 
< HTTP/3 200
< server: istio-envoy
< date: Wed, 27 Oct 2021 05:08:46 GMT
< content-type: application/json
< content-length: 642
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 2
< alt-svc: h3=":443"; ma=86400
< 
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.quic-corp.com", 
    "Transfer-Encoding": "chunked", 
    "User-Agent": "curl/7.78.0-DEV", 
    "X-B3-Parentspanid": "d9f3d78e3f6ddc5c", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "3f6c920b02a92b73", 
    "X-B3-Traceid": "d5c22cd31ddec119d9f3d78e3f6ddc5c", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Envoy-Internal": "true", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/httpbin/sa/httpbin;Hash=97bf9c90d4a5b9bb8f5da3e825dfa34f04631400420649394741807a320aa0a1;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
{{< / highlight >}}
And.....There it goes! **IT WORKS!**
