# Deployment

## Envoy

Bulwark currently runs only as an external processor for [Envoy](https://www.envoyproxy.io/). Other deployment models are [planned](../contributing/roadmap.md), but currently Envoy is required to use Bulwark.

### Deployment with Envoy

Bulwark should generally be configured to run at the front of the HTTP filter chain and in front of the filter for the router. It will process incoming requests, responding to them as directed by its plugins. Bulwark may be scaled as its own service, independent of Envoy itself.

### Example Reverse Proxy Envoy Config

```yaml
static_resources:
  listeners:
    - name: http
      address:
        socket_address:
          address: 127.0.0.1
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                codec_type: AUTO

                # All HTTP traffic should route to the interior cluster.
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: interior

                # Filtering should apply the Bulwark external processing filter before sending to the interior cluster.
                http_filters:
                  - name: envoy.filters.http.ext_proc
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_proc.v3.ExternalProcessor
                      message_timeout:
                        seconds: 2
                      grpc_service:
                        timeout:
                          seconds: 300
                        envoy_grpc:
                          cluster_name: bulwark
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    # The interior service that Bulwark is protecting.
    - name: interior
      connect_timeout: 0.25s
      type: STATIC
      lb_policy: ROUND_ROBIN
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: interior
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 8000
    # The Bulwark external processor.
    - name: bulwark
      connect_timeout: 0.25s
      type: STATIC
      lb_policy: ROUND_ROBIN
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: bulwark
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 8089
```

## Remote State

Bulwark offers plugins the ability to store state via a number of its API functions. This capability is accomplished via Redis. Bulwark may be used without Redis to reduce deployment complexity, however in this configuration, all plugins must be stateless.

{% hint style="info" %}
Remote state is a powerful feature and may contribute significantly to the accuracy of the results Bulwark produces. Enabling it is recommended.
{% endhint %}

## Injected HTTP Headers

Bulwark injects two headers into every request sent onwards to interior services: `Bulwark-Decision` and `Bulwark-Tags`. Both headers are formatted according to the [Structured Field Values for HTTP](https://www.rfc-editor.org/rfc/rfc8941.html) specification and may be parsed by any of its implementations. The `Bulwark-Decision` header contains the values for the combined decision output for the corresponding request. The `Bulwark-Tags` header contains the tags that were associated with the combined decision. There is a [planned improvement](../contributing/roadmap.md) to introduce a signature for these and any other headers Bulwark may emit in the future.

Notably, these headers represent the verdict rendered by the first phase of processing only. At the point where headers are rendered, the request body will not have been processed yet.

Interior services may use this information to enrich business event data, for logging purposes, or to relocate the point of decision-making. This may be particularly relevant if Bulwark is configured in observe-only mode.

### Example Header Values

```
Bulwark-Decision: accept=0.0, restrict=0.55, unknown=0.45
Bulwark-Tags: fraud, card-testing
```
