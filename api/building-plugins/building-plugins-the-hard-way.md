# Building Plugins (The Hard Way)

## Compiling Manually

Compiling manually is **not recommended**. The `build` subcommand should always be preferred over manual compilation. Manual compilation is documented here to demonstrate compatibility with Rust's compiler output.

When compiling a plugin manually, set the `build` command target to `wasm32-wasi`:

```sh
cargo build --target wasm32-wasi --release
```

This writes the WASM binary to `target/wasm32-wasi/release`, but this binary is not yet directly usable by Bulwark.

Bulwark uses the WASM Component Model, and the output from the Rust compiler currently cannot be directly used with it. It needs to be adapted for Bulwark's use.

First, install the WASM tools binary. Installing the latest published version of `wasm-tools` is recommended.

```sh
cargo install wasm-tools
```

> **NOTE:**
>
> This may require a nightly build of the Rust compiler to successfully install.

Once `wasm-tools` has been installed, you need an adapter. The adapter for Bulwark plugins is vendored in the Bulwark repository as `adapter/wasi_snapshot_preview1.reactor.wasm`.

1. Create the `dist` directory.

   ```sh
   mkdir -p dist/
   ```

1. Create your plugin as a new `wasm-tools` component.

   ```sh
   wasm-tools component new target/wasm32-wasi/release/<PLUGIN_NAME_HERE>.wasm --adapt wasi_snapshot_preview1=../bulwark/adapter/wasi_snapshot_preview1.reactor.wasm --output dist/<PLUGIN_NAME_HERE>.wasm
   ```

   Replace `<PLUGIN-NAME-HERE>` with your plugin name, defined in the `name` field of your `Cargo.toml` file.

You can now use the output with Bulwark.
