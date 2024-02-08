# Building Plugins

Bulwark's plugins target the WebAssembly (WASM) instruction format and use the WebAssembly System Interface (WASI) API to communicate with their host environment. Currently, Bulwark only offers support for a [Rust SDK](https://docs.rs/bulwark-wasm-sdk/latest/bulwark\_wasm\_sdk/), however other language support is [planned](../contributing/roadmap.md).

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

A `build` subcommand is included in the `bulwark-cli` binary and should generally be used to compile Bulwark plugins.

```
bulwark-cli build
```

The default location for build output is the `dist/` directory.

## Compiling Manually

Compiling manually is not recommended, but is documented to explain compatibility with Rust's compiler output. When compiling a plugin manually, the build command should set this environment as its target:

{% tabs %}
{% tab title="Rust" %}
`cargo build --target wasm32-wasi --release`
{% endtab %}
{% endtabs %}

This will write the WebAssembly binary to `target/wasm32-wasi/release`, however this binary is not directly usable by Bulwark. Bulwark uses the WebAssembly Component Model and the output from the Rust compiler currently cannot be directly used with it. It needs to be adapted for usage by Bulwark. To do this, first install the WebAssembly tools binary:

{% tabs %}
{% tab title="Rust" %}
```
cargo install wasm-tools
```
{% endtab %}
{% endtabs %}

Note that this may require a nightly build of the Rust compiler to successfully install!

Once `wasm-tools` has been installed, you will need an adapter. The adapter for Bulwark plugins, vendored in the Bulwark repository, is `adapter/wasi_snapshot_preview1.reactor.wasm`. To generate the adapted plugin:

{% tabs %}
{% tab title="Rust" %}
```
mkdir -p dist/
wasm-tools component new target/wasm32-wasi/release/example_plugin.wasm --adapt wasi_snapshot_preview1=../bulwark/adapter/wasi_snapshot_preview1.reactor.wasm --output dist/example_plugin.wasm
```
{% endtab %}
{% endtabs %}

This output may then be used with Bulwark.
