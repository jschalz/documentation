# Building Plugins the Hard Way

## Compiling Manually

Compiling manually is **not recommended**, but is still documented here to help explain compatibility with compiler output. The `build` subcommand should always be preferred over manual compilation.

When compiling a plugin manually, the build command should set its target to `wasm32-wasi`:

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

Once `wasm-tools` has been installed, you will need an adapter. The adapter for Bulwark plugins, vendored in the Bulwark repository, is `adapter/wasi_snapshot_preview1.reactor.wasm`. This assumes the plugin and Bulwark have been cloned into side-by-side directories. From the plugin directory, generate the adapted plugin:

{% tabs %}
{% tab title="Rust" %}
```
mkdir -p dist/
wasm-tools component new target/wasm32-wasi/release/example_plugin.wasm --adapt wasi_snapshot_preview1=../bulwark/adapter/wasi_snapshot_preview1.reactor.wasm --output dist/example_plugin.wasm
```
{% endtab %}
{% endtabs %}

This output may then be used with Bulwark.
