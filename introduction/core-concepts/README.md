# Core concepts

A quick primer on Bulwark, including architecture, WebAssembly, and plugin communication.

## Architecture

Bulwark is split between two environments: the **host** environment and the **guest** environment.

The **host** environment handles incoming requests and processes plugin decision results. The **guest** environment produces the decision results the host receives by running user-supplied logic. The guest environment is "sandboxed" and only uses the host environment to perform special actions on its behalf if explicitly granted permission to do so.

For example, plugins cannot make outbound HTTP requests or access the network by default. To interact with a network, the plugin can make API calls so the host environment sends and receives HTTP requests and responses on its behalf, but only for domains the plugin has access to. This allows you to execute plugins with confidence, knowing they're limited to their configured access.

Bulwark currently only runs as an [Envoy external processor](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext\_proc/v3/ext\_proc.proto). Future [roadmap](../../contributing/roadmap.md) plans include support for other deployment patterns for request processing.

## WebAssembly (WASM)

WebAssembly (or WASM) is a sandbox technology that Bulwark uses to isolate the plugin guest environment. A large number of programming languages offer WASM as a compiler target. Bulwark currently only offers a [first-class Rust SDK](https://docs.rs/bulwark-wasm-sdk/latest/bulwark_wasm_sdk/), but support for other languages is on the [roadmap](../../contributing/roadmap.md). It's possible to implement plugins in languages without an SDK, but the [Rust SDK](https://docs.rs/bulwark-wasm-sdk/latest/bulwark_wasm_sdk/) is the recommended approach.

WASM enables Bulwark's flexible and customizable detection-as-code pattern by compiling detections as executable code before running them in sandboxed guest environments. Plugins also accept external configuration, which allows for generic plugin implementations with domain-specific configuration applied as needed.

## Inter-plugin communication

Bulwark executes plugins across several phases, which eliminates the need to declare plugin interdependencies. Plugins use information made available during an earlier execution phase. By separating plugin responsibilities into phases, domain-specific information extraction can be applied to generic or specific third-party detection logic. To support this use-case, Bulwark defines an [extensible parameter schema](../../api/bulwark-parameter-schema.md).

The Bulwark API allows plugins to extract, store, and retrieve information on a per-request basis. An example use case: securely decrypt session cookies, extract stable user ID values, perform anti-fraud scoring lookups, and then incorporate these values into detection logic.

## Decisions

Bulwark makes all of its security decisions by reading the detection output from plugins. Plugins use a decision object to quantitatively express uncertainty around detection results in a user-intuitive format.

Each decision is composed of three values: an `accept` value, a `restrict` value, and an `unknown` value. All three must be positive real numbers (including whole, decimal, and irrational numbers) in the range zero to one, and have a combined sum of one. These values must be in a decimal format.

| Acceptable values ✅  | Unacceptable values ❌ |
| -------------------- | --------------------- |
| 0.0                  | 0                     |
| 0.7402               | - 0.2                 |
| 0.001                | "0.33"                |

The values `accept` and `restrict` indicate if the plugin believes the detected threat should be accepted or restricted for your application. The greater the value for either `accept` or `restrict`, the stronger the evidence a plugin claims for the value's respective outcome. A high `accept` value and a low `restrict` value indicates the plugin believes your application should allow the detected traffic. Likewise, a high `restrict` value and a low `accept` value indicates the plugin believes your application should restrict the detected traffic.

If the plugin has poor evidence for the detected traffic, the plugin uses the `unknown` value. The greater the `unknown` value, the weaker the evidence a plugin claims for either allowing or restricting the traffic. Plugins may indicate it has no evidence by simply returning nothing (an empty decision object), or by setting the decision's `unknown` component to its maximum value (`1.0`).

Bulwark collects decision outputs from each plugin at the completion of a decision phase, and then combines all of these values together to produce a final decision that will determine how Bulwark responds. Decisions are converted into a score value when determining what a request's outcome will be.

To simplify plugin development, Bulwark provides the convenience functions `set_accepted` and `set_restricted`. These functions are analogous to "allow" and "deny" operations that new users may be more familiar with.

Decision structures also include an optional set of tags to annotate results, which enables easier logging and searching.

Read more about decision structures and combination algorithms on the [Decision Internals](decision-internals.md) page.
