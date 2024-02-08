# Configuration

Bulwark uses the [TOML](https://toml.io/) file format for its configuration. Settings are organized into related sections or tables, with some tables being repeatable. The path to the configuration file may be supplied via the `--config` [CLI](cli.md) parameter.

## Example Config Files

{% tabs %}
{% tab title="main.toml" %}
```toml
[service]
port = 8089
admin_port = 8090
remote_state = "redis://127.0.0.1:6379"

[thresholds]
restrict = 0.75

[[include]]
path = "include.toml"

[[plugin]]
ref = "evil_bit"
path = "bulwark-evil-bit.wasm"

[[preset]]
ref = "default"
plugins = ["evil_bit", "starter_preset"]

[[resource]]
route = "/"
plugins = ["default"]
timeout = 25

[[resource]]
route = "/*params"
plugins = ["default"]
timeout = 25
```
{% endtab %}

{% tab title="include.toml" %}
```toml
[[plugin]]
ref = "blank_slate"
path = "bulwark-blank-slate.wasm"
config = {}

[[preset]]
ref = "starter_preset"
plugins = ["blank_slate"]
```
{% endtab %}
{% endtabs %}

## `service`&#x20;

All of the settings for Bulwark's services are organized under the `service` table.

### `port`

Type: `u16`

Default: `8089`

The port number for the Bulwark primary service. Bulwark's primary service is determined by the subcommand supplied to the [CLI](cli.md).

### `admin_port`

Type: `u16`

Default: `8090`

The port number for the Bulwark admin service. The admin service hosts the health check endpoints (`/live`, `/started`, and `/ready`). Further details may be found on the [Deployment](../introduction/deployment.md) page.

### `admin_enabled`

Type: `bool`

Default: `true`

Determines whether the admin service will be enabled. If set to `false`, only the primary service will be available.

### `remote_state_uri`

Type: `String`

An optional setting that enables the use of Bulwark's remote state feature. It should be set to a Redis connection URI. When a plugin makes a host call that uses remote state, the corresponding Redis commands will be sent here.

{% hint style="warning" %}
While this setting is optional and disabled by default, it is highly recommended that Bulwark be used with it enabled. Much of Bulwark's capabilities are derived from it.
{% endhint %}

### `remote_state_pool_size`

Type: `u32`

Default: `16`

The size of the connection pool to use when accessing remote state.

### `proxy_hops`

Type: `u8`

Default: `0`

The number of trusted proxy hops in front of, or exterior to, the Bulwark primary service. Bulwark's own proxy hop or the hop of the proxy hosting Bulwark would not be counted. This setting determines which client IP will be reported to the plugin guest environment when parsing the `Forwarded` or `X-Forwarded-For` HTTP header.

{% hint style="danger" %}
If Bulwark is deployed in conjunction with a load balancer or with another proxy exterior to it and the `proxy_hops` setting is not updated to reflect this, some Bulwark plugins may attempt to block traffic from trusted proxy sources whenever malicious traffic from any source is detected.
{% endhint %}

## `metrics`

The settings for how Bulwark provides and manages its metrics are organized under the `metrics` table. By default, Bulwark uses Prometheus metrics and does not require any configuration other than the `admin_enabled` setting to use them.

### `statsd_host`

Type: `String`

This is the hostname of the StatsD server to push metrics to. All other StatsD settings are ignored if this value is not set.

{% hint style="warning" %}
If set, disables Prometheus metrics support and enables StatsD support instead. Enabling both at once is not supported.
{% endhint %}

### `statsd_port`

Type: `u16`

Default: `8125`

Determines the port number that will be used for the StatsD server that metrics are pushed to.

### `statsd_queue_size`

Type: `usize`

Default: `5000`

Determines the size of the queue used when pushing metrics to StatsD.

### `statsd_buffer_size`

Type: `usize`

Default: `1024`

Determines the size of the buffer metrics are sent to before being sent to the StatsD server.

### `statsd_prefix`

Type: `String`

A prefix to apply to StatsD metrics names.

## `thresholds`

The configurable decision thresholds are organized under the `thresholds` table.

### `observe_only`

Type: `bool`

Default: `false`

Disables blocking of requests. If set, Bulwark will allow all requests through, regardless of the outcome determined by its plugins. It will still calculate a verdict, it will still inject headers readable by interior services, plugin-specific side-effects may still occur, but Bulwark will not otherwise take action on an incoming request. This can be used to perform initial evaluation of Bulwark without concern for false positives.

### `restrict`

Type: `f64`

Default: `0.8`

The decision score lower-bound for blocking a request. Any decision risk score above this threshold will be blocked if Bulwark is in blocking mode.

### `suspicious`

Type: `f64`

Default: `0.6`

The decision score lower-bound for marking a request as suspicious. Any decision risk score above this threshold will be marked as suspicious. Suspicious requests are not blocked, but they may be queried and manually inspected and plugins may use them in the context of feedback loops to inform future decision-making.

### `trust`

Type: `f64`

Default: `0.2`

The decision score upper-bound for marking a request as trusted. Any decision risk score value below this threshold will be marked as trusted. Trusted requests are not handled differently than accepted requests, but plugins may use them in the context of feedback loops to inform future decision-making.

## `include`

An include directive used during initial configuration loading. Allows configuration to span multiple files. Multiple `include` tables may appear in a configuration file.

### `path`

Type: `String`

The relative path of the configuration file to be included. The path is resolved relative to the configuration file that is performing the include.

## `plugin`

Configuration for an individual plugin. Multiple `plugin` tables may appear in a configuration file. Additionally, multiple `plugin` tables that point to the same file `path` may appear, as long as they have different `reference` values. This is typically used to supply different guest configurations to the same plugin implementation.

### `reference`

Type: `String`

A unique identifier that should be both human-readable and machine-readable, for a specific plugin and its associated configuration. Should only include ASCII lowercase a-z and underscore (\_) characters.

### `path`

Type: `String`

The relative path to the compiled Web Assembly file for a plugin. The path is resolved relative to the file in which it appears.

{% hint style="info" %}
This may eventually change to a URI field to enable future [roadmap](../contributing/roadmap.md) items.
{% endhint %}

### `weight`

Type: `f64`

Default: `1.0`

A weight factor which this plugin's decisions will be multiplied by. This may be used to tune results from a plugin without recompiling it. In particular, values below 1.0 may be used to scale down the output of the `set_accepted` or `set_restricted` host calls, which would otherwise only output a maximum value.

### `config`

Type: `Map<String, Value>`

Default: `{}`

An opaque configuration structure that will be passed to the plugin guest environment. It will be JSON-serialized when sent to the plugin. Language-specific SDKs may provide convenience functions to automatically deserialize this value within the guest.

### `permissions`

Type: `Map<String, Vec<String>>`

Default: `{}`

A set of permission grants for the plugin. If a plugin attempts to perform a privileged operation without the necessary permission grant, it will exit immediately with a panic.

#### `env`

The `env` permission grant is a list of environment variables that this plugin is allowed to read. This is typically used to grant fine-grained access to a secret.

#### `http`

The `http` permission grant is a list of URI host values that may be used when sending outbound HTTP requests from a plugin. Host values must match exactly and wildcards are not currently supported.

#### `state`

The `state` permission grant is a list of Redis key prefixes that may be accessed when using Bulwark's remote state feature.

## `preset`

A preset is a reusable grouping of plugins and other presets that can be used to reduce repetition within Bulwark's configuration. A good practice is to create a default preset that provides generic protection, and then use more specialized plugin lists on a per-resource basis, where needed.

### `reference`

Type: `String`

A unique identifier that should be both human-readable and machine-readable, for a specific preset and its associated configuration. Should only include ASCII lowercase a-z and underscore (\_) characters.

### `plugins`

Type: `Vec<String>`

A list of plugin reference values or other preset references that this preset should represent. Circular references are not allowed. Reference types may be a mix of plugin and preset references.

## `resource`

A resource in Bulwark's configuration is a mapping between an endpoint and the set of plugins that will be applied to incoming requests for it. Resource tables may also be used to control how latency-sensitive Bulwark should be on a per-endpoint basis.

### `route`

The route pattern that this resource should match. Route patterns may capture parameters, which will be made accessible to plugins via the `get_param_value` function in the API. Single path parameters may be captured with a `:` sigil character and multiple path parameters may be captured with a `*` sigil character. Query strings are ignored for matching, but may be inspected by plugins.

Routes within Bulwark match only the request. An interior service does not need to respond to the same or even similar route patterns, and in some cases, Bulwark's configured routes may correspond as much to threat patterns as they do to the shape of the interior services.

#### Example patterns

The `/` pattern will match only the apex path. It is not a prefix match. The `/*params` pattern will match everything except the apex path, because `*params` portion of the pattern will not match if it is empty. A `/:user/profile` pattern would match the request path `/alice/profile`, capturing the value `alice` into the `user` parameter.

### `plugins`

Type: `Vec<String>`

A list of plugin reference values or other preset references that this resource should execute in response to a matching request. Circular references are not allowed. Reference types may be a mix of plugin and preset references.

### `timeout`

Type: `u64`

Default: `10`

A timeout duration, specified in milliseconds. Each plugin handler phase will have this long to complete execution. If a plugin times out prematurely, the results of its execution will not be considered for that phase. If a plugin registers multiple handlers across both the request and response phase, subsequent handlers will still execute even if earlier handlers timed out. Therefore if the maximum tolerable latency for an endpoint is 30ms, this timeout would be set to 15ms. Remote state becoming unavailable is one scenario where both phase timeouts might be hit.
