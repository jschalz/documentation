---
description: A quick primer on Bulwark
---

# Core Concepts

## Architecture

Bulwark's runtime has two distinct environments, the host environment and the guest environment. The host environment is responsible for handling incoming requests and processing plugin decision results. The guest environment is where user-supplied plugin logic runs, producing the decision information that the host acts upon.

The guest environment is sandboxed and can only recruit the host environment to perform special actions on its behalf if explicitly granted permission to do so. For example, by default, plugins cannot make outbound HTTP requests, or otherwise access the network at all. However the plugin may call a function that causes the host environment to make an HTTP request and return the response it receives, for the specific domains that it's been granted access to. This allows plugins to be executed with confidence, knowing that they'll do nothing more than they've been configured to do.

Bulwark currently runs only as an [Envoy external processor](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext\_proc/v3/ext\_proc.proto). Future [roadmap](../../contributing/roadmap.md) plans include support for other deployment patterns for request processing.

## WebAssembly

WebAssembly, or WASM, is a sandbox technology that Bulwark uses to isolate the plugin guest environment. A large number of programming languages offer WebAssembly as a compiler target. Currently, Bulwark only offers a first-class Rust SDK, however other languages are on the [roadmap](../../contributing/roadmap.md). It's possible to implement plugins in languages without an SDK yet, but for now, the Rust SDK is the recommended approach.

WASM enables Bulwark's detection-as-code pattern, where all detections are compiled to executable code and then run in the sandboxed guest environment. This makes Bulwark very flexible and enables a wide range of detection methods. Plugins can also accept external configuration, allowing for generic plugin implementations that become domain-specific once their configuration is applied.

## Inter-Plugin Communication

The Bulwark API offers mechanisms for plugins to extract, store, and retrieve information on a per-request basis. This may be used, for instance, to securely decrypt session cookies, extract stable user ID values, perform anti-fraud scoring lookups, and then incorporate these values into detection logic.

Bulwark executes plugins across several phases, helping it to avoid the need to explicitly declare plugin interdependencies. Plugins may reliably use information made available during an earlier execution phase. Separating responsibility for extracting information from using it within a detection allows for domain-specific information extraction to be subsequently applied to generic or third-party detection logic. To support this use-case, Bulwark defines an [extensible parameter schema](../../api/bulwark-label-schema.md).

## Decisions

Bulwark makes all of its security decisions by reading the output from plugins. Plugins primarily output a decision structure, accompanied by an optional set of tags that help to annotate the result. The decision structure is designed to allow plugins to quantitatively express uncertainty in an intuitive way. Each decision is composed of three values, an `accept` value, a `restrict` value, and an `unknown` value. All three are expected to be real numbers in the range zero to one, and have a combined sum of one. The greater the value for either the `accept` or `restrict` value, the stronger the evidence a plugin is claiming for the respective outcome. The greater the `unknown` value, the weaker a plugin is claiming its evidence is. Plugins may indicate that they have no evidence one way or the other by simply returning nothing or by setting their decision's `unknown` component to its maximum value.

Bulwark collects decisions from each plugin at the completion of a decision phase, combining all of these values together to produce a final decision that will determine how Bulwark responds. Decisions are converted into a score value when determining what a request's outcome will be.

To simplify plugin development, there are convenience functions, `set_accepted` and `set_restricted`, which are more analogous to "allow" and "deny" operations that new users may be more comfortable with.

A more advanced discussion of the decision structure and combination algorithms can be found on the [Decision Internals](decision-internals.md) page.
