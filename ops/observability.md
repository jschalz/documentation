# Observability

## Request Enrichment

One of the most valuable capabilities that Bulwark introduces for improving visibility for security into your traffic is the concept of request enrichment. Aside from taking accept and restrict actions on traffic, Bulwark plugins also have the ability to annotate traffic with useful tags and labels. Plugins have access to information beyond what comes over the wire. Much more useful contextual information can be injected as a result. These annotations can then be queried using standard observability tools after the fact. Depending on the plugins deployed, this can enable use-cases such as threat hunting, more efficient incident response, and better forensic analysis.

## Structured Logs

Bulwark currently relies on a mix of its own logs and metrics and also [Envoy's metrics](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http\_conn\_man/stats) for observability. Bulwark supports both Prometheus-compatible metrics scraping and StatsD for metrics collection. There are a number of future [roadmap items](../contributing/roadmap.md) related to improving Bulwark's capabilities in this area. Since Bulwark is intended to function as a security observability tool in its own right, this is a development area that will receive significant attention.

Bulwark currently offers two log formats. The first is a structured newline-delimited JSON format that implements the [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-reference.html) (ECS) specification and is intended for use with centralized log stores and other consumers of [high-cardinality](https://www.cncf.io/blog/2022/05/23/what-is-high-cardinality/) event data. The second is a human-readable multi-line log format intended for debugging use-cases. Other log formats may be introduced in the future, as needed.

{% tabs %}
{% tab title="ECS" %}
```json
{
  "@timestamp": "2023-05-04T22:17:22.174144Z",
  "message": "GET /hello [403]",
  "log": {
    "level": "info"
  },
  "http": {
    "request": {
      "method": "GET"
    },
    "response": {
      "status": 403
    }
  },
  "url": {
    "original": "/hello"
  },
  "user_agent": {
    "original": "curl/7.79.1"
  },
  "event": {
    "kind": "event",
    "category": [
      "web"
    ],
    "type": [
      "denied"
    ]
  },
  "risk": {
    "calculated_level": "restricted",
    "calculated_risk_score": 1
  },
  "bulwark": {
    "plugins": {
      "example_plugin": {
        "accept": 0,
        "restrict": 1,
        "score": 1,
        "unknown": 0
      }
    }
  }
}
```

The output above has been piped to `jq -r` for readability. It is emitted as a condensed single line by the Bulwark process.
{% endtab %}

{% tab title="Forest" %}
```
INFO     handle request [ 258ms | 0.00% / 100.00% ]
INFO     ┝━ ｉ [info]: "process request" | method: "GET" | uri: "/hello" | user_agent: "curl/7.79.1"
INFO     ┕━ route request [ 258ms | 0.00% / 100.00% ]
INFO        ┝━ execute on_request [ 257ms | 99.75% ]
INFO        ┝━ execute on_request_decision [ 32.7µs | 0.01% ]
INFO        │  ┕━ ｉ [info]: "plugin decision" | name: "example_plugin" | accept: 0.0 | restrict: 0.0 | unknown: 1.0 | score: 0.5
INFO        ┝━ ｉ [info]: "combine decision" | accept: 0.0 | restrict: 0.0 | unknown: 1.0 | score: 0.5 | outcome: "accepted" | tags: ""
INFO        ┝━ execute on_response_decision [ 100µs | 0.04% ]
INFO        │  ┕━ ｉ [info]: "plugin decision" | name: "example_plugin" | accept: 0.0 | restrict: 1.0 | unknown: 0.0 | score: 1.0
INFO        ┝━ ｉ [info]: "combine decision" | accept: 0.0 | restrict: 1.0 | unknown: 0.0 | score: 1.0 | outcome: "restricted" | tags: ""
INFO        ┝━ ｉ [info]: "process response" | status: 403
INFO        ┕━ execute on_decision_feedback [ 517µs | 0.20% ]
```

Timestamps and request IDs have been elided for brevity.
{% endtab %}
{% endtabs %}
