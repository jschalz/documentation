---
description: Common plugin API reference definitions
---

# Reference

## Compatibility

The Bulwark API is currently unstable. Prior to a 1.0.0 release, API signatures [may change from one release to the next](https://semver.org/#spec-item-4). Prior to 1.0.0, minor version numbers will be incremented for breaking API changes. After 1.0.0, Bulwark's API will become stable. At that point, intentionally breaking changes will be uncommon and should only occur for major API revisions, with new versions tested against the [Community Ruleset](../../contributing/community-ruleset.md).

## Guest handler functions

Bulwark plugins define handler functions to perform their detection logic. The Bulwark plugin API has five handler functions that may be used. When using an SDK, these handlers will perform a no-op if omitted. Most plugins will primarily use the `handle_request_decision` handler. All plugins are guaranteed to complete execution of each phase's corresponding handlers before beginning the subsequent phase, however no order is guaranteed within a phase. Plugins should be assumed to execute concurrently within a phase.

### `handle_init`

Performs any one-time initialization logic needed by the plugin. Most plugins will omit this handler.

### `handle_request_enrichment`

This handler is called after a plugin has completed initialization and a request has become available for processing, but prior to a [decision](../../introduction/core-concepts/#decisions) being made for the request phase. This handler receives a request and a set of optional [parameters](../bulwark-label-schema.md) extracted from the [routes](../../ops/configuration.md#route) in the config file. It returns a set of new parameters to merge with the existing set. Bulwark will perform a merge of the new parameters onto the existing ones after all enrichment handlers return.

### `handle_request_decision`

This is the most common handler for plugins to implement. All initialization and request enrichment will have completed prior to this handler's execution. This handler will receive the same arguments as `handle_request_enrichment`, but it returns a handler output structure instead, which wraps a decision, a set of tags, as well as any new parameters to further enrich the request with.

After all plugins complete the execution of their `handle_request_decision` handlers, a combined decision is produced. The outcome of this decision determines whether to proceed with sending the request onwards to the interior service or to immediately restrict the request. If a request is marked as restricted, the response phase will be skipped, but the feedback phase will still occur. A request marked as restricted will be blocked unless Bulwark is in observe-only mode.

### `handle_response_decision`

If a request was not blocked by a combined decision from the previous phase, the request will be proxied to the interior service, which will return a response. Prior to this response being sent onwards to the client, it will be sent first to each plugin's `handle_response_decision` function for processing.

In some cases, a plugin's handler may take no action during this phase other than analyzing the response and possibly storing some state or incrementing a counter based on the response status, usable when processing future requests. If a plugin takes no action during this phase or if the handler isn't defined at all, the decision made during the preceding phase will be carried forward. This prevents a small number of plugins having outsized influence during the response phase due to no-op implementations in the other plugins.

After all plugins complete the execution of their `handle_response_decision` functions, a second, final combined decision is produced. The outcome of this decision determines whether to send the response onwards to the exterior client or to immediately reject the request despite the interior service having already processed it.

Notably, for authentication scenarios, if a response has been blocked by this phase, it may be prudent to use the `handle_decision_feedback` handler to invalidate the session. Otherwise a client may receive an access denied response but may still possess a session which is now logged in. This may be achieved by calling a session logout endpoint with the same session.

### `handle_decision_feedback`

After a final combined decision has been reached, the `handle_decision_feedback` function is called. It takes the request, response, parameters, and a verdict as arguments and does not need to return anything. A verdict contains the combined decision and the combined set of all tags applied to the request by any of the plugins. These values are intended to allow plugins to create feedback loops. Alternatively, the handler may be used to perform additional post-decision mitigations, like invalidating a session.

A feedback loop pattern will typically use one of the Redis API functions, such as `incr` or `incr_rate_limit`, to provide additional contextual information that will be available when processing subsequent requests.

Mitigation patterns typically respond to a restricted request verdict by sending an outgoing request to an interior service or possibly a vendor API, in order to prevent future malicious activity by the client. This might be a call to the logout endpoint on an authentication service or a call to an anti-fraud API, as appropriate. Plugins using the `handler_decision_feedback` handler in this way should be selective in which requests trigger additional mitigations. It would generally not be appropriate to respond to all restricted requests by invalidating a session for example.
