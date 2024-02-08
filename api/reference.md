---
description: Common plugin API reference definitions
---

# Reference

## Compatibility

The Bulwark API is currently unstable. Prior to a 1.0.0 release, API signatures [may change from one release to the next](https://semver.org/#spec-item-4). Prior to 1.0.0, minor version numbers will be incremented for breaking API changes. After 1.0.0, Bulwark's API will become stable. At that point, intentionally breaking changes will be uncommon and only occur for major API revisions, with new versions tested against the [Community Ruleset](../contributing/community-ruleset.md).

## Guest handler functions

At least one of these functions must be declared within the guest plugin. Other than the `main` function, handlers are optional and may be omitted if the plugin does not need to make use of a particular handler. The `main` function must be declared, but may be empty. Each handler corresponds to one of Bulwark's execution phases and they will be executed in order. Handler functions do not take parameters nor do they return any values, instead relying on Host API functions for their inputs and outputs. For each handler type, all plugins processing a request will have their handlers executed concurrently.

<details>

<summary><code>main</code></summary>

The WASM entry point. If compiled as a [WASI target](https://wasi.dev/), this will be the `main()` function. If compiled as a WASM function without WASI, this function will be named `_start()` instead. Generally, plugin authors should be [compiling as a WASI target](building-plugins.md).

The `main` function may be used to perform any plugin initialization as needed, but it is commonly empty. Host API functions, including the `get_request` function are callable from within the `main` function, but by convention, only initialization logic and logic that must complete prior to the `on_request` handler should run in the main function.

</details>

<details>

<summary><code>on_request</code></summary>

This handler is called after a plugin has completed initialization and a request has become available for processing, but prior to a [decision](../introduction/core-concepts/#decisions) being made for the request phase. This handler is most commonly used for plugins that need to perform information extraction on the request. A typical pattern would be to call the [`set_param_value`](reference.md#set\_param\_value) function from this handler, passing a key selected from the [parameter schema](bulwark-parameter-schema.md) for subsequent use by other plugins. Since this handler is guaranteed to complete prior to any [`on_request_decision`](reference.md#on\_request\_decision) handler being executed for a request, other plugins can rely upon this information being available.

</details>

<details>

<summary><code>on_request_decision</code></summary>

This is the most common handler for plugins to implement. All initialization and information extraction operations will have completed prior to this handler's execution. In this handler, plugins will generally call [`get_request`](reference.md#get\_request) to retrieve the request for processing, perform their detection logic on the request, and then call [`set_decision`](reference.md#set\_decision) with their [decision](../introduction/core-concepts/#decisions) result. Plugins may optionally omit the call to [`set_decision`](reference.md#set\_decision), implicitly returning a decision indicating full uncertainty, equivalent to an explicit `set_decision(0.0, 0.0, 1.0)` call.

After all plugins complete the execution of their `on_request_decision` handlers, a combined decision is produced. The outcome of this decision determines whether to proceed with sending the request onwards to the interior service or to immediately reject the request without proxying it.

</details>

<details>

<summary><code>on_response_decision</code></summary>

If a request was not blocked by a combined decision from the [`on_request_decision`](reference.md#on\_request\_decision) handlers, the request will be proxied to the interior service, which will return a response. Prior to sending this response back to the client, it will be made available to the `on_response_decision` handlers for processing.

In some cases, a plugin's handler may take no action during this phase other than analyzing the response and possibly storing some state or incrementing a counter based on the response status, usable when processing future requests. If a plugin takes no action during this phase or if the handler isn't defined at all, the decision made during the [`on_request_decision`](reference.md#on\_request\_decision) phase will be carried forward.

If a plugin does call `set_decision` during the `on_response_decision` handler, it will overwrite the previous decision for the plugin.

After all plugins complete the execution of their `on_response_decision` handlers, a second, final combined decision is produced. The outcome of this decision determines whether to send the response onwards to the exterior client or to immediately reject the request despite the interior service having already processed it.

Notably, for authentication scenarios, if a response has been blocked by this phase, it may be prudent to use the `on_decision_feedback` handler to invalidate the session. Otherwise a client may receive an access denied response but may still possess a session which is now logged in. This may be achieved by calling a session logout endpoint via `send_request` with the same session.

</details>

<details>

<summary><code>on_decision_feedback</code></summary>

After a final combined decision has been reached, the `on_decision_feedback` handler will execute. The `get_combined_decision` and `get_combined_tags` host functions may be used to create feedback loops. The handler may also be used to perform additional post-decision mitigations, like invalidating a session.

A feedback loop pattern will typically use one of the remote state host functions, such as `increment_remote_state`, to provide additional contextual information that will be available when processing subsequent requests.

Mitigation patterns, like session invalidation, will typically respond to a blocked request by calling `send_request` to an interior service, and preventing future malicious activity by the client. This might be a call to the logout endpoint on the authentication service or a call to an anti-fraud API, as appropriate. Plugins using the `on_decision_feedback` handler in this way should be selective in which requests trigger additional mitigations. It would generally not be appropriate to respond to all blocked requests by invalidating a session for example.

</details>

## API functions

These functions may be called from within a plugin handler function to produce a desired detection behavior. Some functions may require permissions to be configured in order to execute successfully. Other functions may only be callable from within specific handler functions. In both cases, these restrictions will be specifically noted below.

<details>

<summary><code>get_request</code></summary>

Arguments: None

Returns: `Request`

Returns the incoming request.

The SDK current always returns an empty body. Currently, detections are limited to reading request metadata and headers only. This is on the [roadmap](../contributing/roadmap.md).

</details>

<details>

<summary><code>get_response</code></summary>

Arguments: None

Returns: `Response`

Returns the response received from the interior service.

The SDK current always returns an empty body. Currently, detections are limited to reading response metadata and headers only. This is on the [roadmap](../contributing/roadmap.md).

</details>

<details>

<summary><code>get_client_ip</code></summary>

Arguments: None

Returns: `Option<IpAddr>`

Returns the originating client's IP address, if available. If there is no `Forwarded` or `X-Forwarded-For` header present on the incoming request or if [`proxy_hops`](../ops/configuration.md#proxy\_hops) has been configured incorrectly, this may not return an IP.

</details>

<details>

<summary><code>get_param_value</code></summary>

Arguments:

* `key` - The parameter key name. Should be named according to the [Bulwark Parameter Schema](bulwark-parameter-schema.md).

Returns: `Value`

Returns a named value which is scoped to the incoming request.

</details>

<details>

<summary><code>set_param_value</code></summary>

Arguments:

* `key` - The parameter key name. Should be named according to the [Bulwark Parameter Schema](bulwark-parameter-schema.md).
* `value` - The value to record.

Sets a named value which will be scoped to the incoming request. Other plugins may read this value. This function is typically called from within the [`on_request`](reference.md#on\_request) handler.

</details>

<details>

<summary><code>get_config</code></summary>

Arguments: None

Returns: `Value`

Returns the guest environment's configuration. By convention, this would return an `Object`. Most plugins will use [`get_config_value`](reference.md#get\_config\_value) instead, unless a plugin needs to iterate over its entire configuration.

</details>

<details>

<summary><code>get_config_value</code></summary>

Arguments:

* `key` - A key indexing into a configuration `Object`.

Returns: `Option<Value>`

Returns a named guest environment configuration value as a `Value`.  A shortcut for calling `get_config`, reading it as an `Object`, and then retrieving a named `Value` from it. Returns nothing if the key does not exist.

</details>

<details>

<summary><code>get_env</code></summary>

Arguments:

* `key` - The environment variable name. Case-sensitive.

Returns: `String`

Returns a named environment variable value as a `String`.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#env) for the environment variable being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>get_env_bytes</code></summary>

Arguments:

* `key` - The environment variable name. Case-sensitive.

Returns: `Vec<u8>`

Returns a named environment variable value as a bytes.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#env) for the environment variable being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>set_decision</code></summary>

Arguments:

* `decision` - The `Decision` output of the plugin.

Returns: `Result<(), ValidationErrors>`

Records the [decision](../introduction/core-concepts/#decisions) value the plugin wants to return. All components of a `Decision` must be in the range 0.0 to 1.0 and must sum to 1.0. An `accept` value of 1.0 indicates that a request should be accepted with maximum confidence, a `restrict` value of 1.0 indicates that a request should be restricted with maximum confidence, and an `unknown` value of 1.0 indicates full uncertainty or a complete lack of evidence for an outcome in either direction. All lower values indicate progressively lower weight for their respective components, down to the lower limit of zero.

The [`set_accepted`](reference.md#set\_accepted) and [`set_restricted`](reference.md#set\_restricted) functions offer convenience wrappers for calling this function. An `unknown` value of 1.0 is the default decision and omitting a call to `set_decision` is equivalent unless a plugin has already set a decision during another phase.

</details>

<details>

<summary><code>set_accepted</code></summary>

Arguments:

* `value` - An `accept` value. Should between 0.0 and 1.0 and clamped otherwise.

Records a [decision](../introduction/core-concepts/#decisions) indicating how much a plugin wants to accept a request, assigning any remainder to uncertainty. The `unknown` component value will be derived from the supplied `accept` value. For example, if called with a 0.4 parameter value, it will derive a 0.6 `unknown` component value.

</details>

<details>

<summary><code>set_restricted</code></summary>

Arguments:

* `value` - A `restrict` value. Should between 0.0 and 1.0 and clamped otherwise.

Records a [decision](../introduction/core-concepts/#decisions) indicating how much a plugin wants to restrict a request, assigning any remainder to uncertainty. The `unknown` component value will be derived from the supplied `restrict` value. For example, if called with a 0.4 parameter value, it will derive a 0.6 `unknown` component value.

</details>

<details>

<summary><code>set_tags</code></summary>

Arguments:

* `tags` - The list of tags to associate with a `Decision`.

Records the tags the plugin wants to associate with its decision. Tags have no direct effect on a `Decision` outcome, but may be used to help understand why a particular outcome occurred, or to provide a queryable value that may be used to locate related request decisions.

</details>

<details>

<summary><code>get_combined_decision</code></summary>

Arguments: None

Returns: `Decision`

Returns the combined decision. Typically used within the [`on_decision_feedback`](reference.md#on\_decision\_feedback) handler.

</details>

<details>

<summary><code>get_combined_tags</code></summary>

Arguments: None

Returns: `Vec<String>`

Returns the combined set of tags associated with a decision. Typically used within the [`on_decision_feedback`](reference.md#on\_decision\_feedback) handler.

</details>

<details>

<summary><code>get_outcome</code></summary>

Arguments: None

Returns: `Outcome`

Returns the outcome of the combined decision. Typically used within the [`on_decision_feedback`](reference.md#on\_decision\_feedback) handler.

</details>

<details>

<summary><code>send_request</code></summary>

Arguments:

* `request` - The HTTP request to send.

Returns: `Response`

Sends an outbound HTTP request.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#http) for the host being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>get_remote_state</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.

Returns: `Vec<u8>`

Returns the named state value retrieved from Redis. Also used to retrieve a counter value. Counters are stored as strings.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>set_remote_state</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.
* `value` - The value to record. Values are byte strings, but may be interpreted differently by Redis depending on context.

Set a named value in Redis.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>increment_remote_state</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.

Increments a named value in Redis by one.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>increment_remote_state_by</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.
* `delta` - The amount to increase the counter by.

Increments a named value in Redis by the delta.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>set_remote_ttl</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.
* `ttl` - The time-to-live for the value in seconds.

Sets an expiration on a named value in Redis.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>increment_rate_limit</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.
* `delta` - The amount to increase the counter by.
* `window` - How long each period should be in seconds.

Returns: `Rate`

Increments a rate limit, returning the number of attempts so far and the expiration time.

The rate limiter is a counter over a period of time. At the end of the period, it will expire, beginning a new period. Window periods should be set to the longest amount of time that a client should be locked out for. The plugin is responsible for performing all rate-limiting logic with the counter value it receives. Rate-limiting expirations will happen automatically, there is no need to manage them via [`set_remote_ttl`](reference.md#set\_remote\_ttl).

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>check_rate_limit</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.

Returns: `Rate`

Checks a [rate limit](reference.md#increment\_rate\_limit), returning the number of attempts so far and the expiration time.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>increment_breaker</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.
* `delta` - The amount to increase the success or failure counter by.
* `window` - How long each period should be in seconds.

Returns: `Breaker`

Increments a circuit breaker, returning the generation count, success count, failure count, consecutive success count, consecutive failure count, and expiration time.

The plugin is responsible for performing all circuit-breaking logic with the counter values it receives. The host environment does as little as possible to maximize how much control the plugin has over the behavior of the breaker. Circuit breaker expirations will happen automatically, there is no need to manage them via [`set_remote_ttl`](reference.md#set\_remote\_ttl).

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>

<details>

<summary><code>check_breaker</code></summary>

Arguments:

* `key` - The key name corresponding to the state value.

Returns: `Breaker`

Checks a circuit breaker, returning the generation count, success count, failure count, consecutive success count, consecutive failure count, and expiration time.

In order for this function to succeed, a plugin's configuration must explicitly declare a [permission grant](../ops/configuration.md#state) for the prefix of the key being requested. This function will panic, terminating the plugin's execution, if permission has not been granted.

</details>
