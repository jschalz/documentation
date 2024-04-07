# Building Plugins

Bulwark's plugins target the WebAssembly (WASM) instruction format and use the WebAssembly System Interface (WASI) API to communicate with their host environment. Currently, Bulwark only offers support for a [Rust SDK](https://docs.rs/bulwark-wasm-sdk/latest/bulwark\_wasm\_sdk/), however other language support is [planned](../../contributing/roadmap.md).

The Rust SDK requires a `Cargo.toml` file in the plugin directory. This declares a plugin's dependencies and includes useful metadata like the plugin name and author information.

{% tabs %}
{% tab title="Rust" %}
{% code title="cargo.toml" %}
```toml
[package]
name = "example-plugin"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
bulwark-wasm-sdk = "0.3.0"

[lib]
crate-type = ["cdylib"]

# These settings may help optimize file size for release builds
[profile.release]
lto = true
opt-level = 3
codegen-units = 1
panic = "abort"
strip = "debuginfo"
```
{% endcode %}
{% endtab %}
{% endtabs %}

Plugin logic will be written in a `src/lib.rs` file.

{% tabs %}
{% tab title="Rust" %}
{% code title="src/lib.rs" %}
```rust
use bulwark_wasm_sdk::*;

struct ExamplePlugin;

#[bulwark_plugin]
impl Handlers for ExamplePlugin {
    fn on_request_decision() -> Result {
        // Plugin logic goes here
        Ok(())
    }
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

A [`build` subcommand](../../ops/cli.md#build) is included in the `bulwark-cli` binary and should generally be used to compile Bulwark plugins.

```
bulwark-cli build
```

The default location for build output is the `dist/` directory.
