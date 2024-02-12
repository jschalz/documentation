# Building Plugins

Bulwark's plugins target the WebAssembly (WASM) instruction format and use the WebAssembly System Interface (WASI) API to communicate with their host environment.

Bulwark supports a [Rust SDK](https://docs.rs/bulwark-wasm-sdk/latest/bulwark\_wasm\_sdk/). Other language support is planned in the [project roadmap](../contributing/roadmap.md).

## Prerequisites

Before building a Bulwark plugin, ensure you've done the following:

1) Follow the [Installation](https://docs.bulwark.security/introduction/installation) guide.

## Create a Cargo.toml file

The Rust SDK requires a `Cargo.toml` file in the plugin directory. This declares a plugin's dependencies and includes useful metadata like the plugin name and author information. Read more about `Cargo.toml` files in the Rust [Cargo Book Manifest](https://doc.rust-lang.org/cargo/reference/manifest.html) reference.

The following configuration file shows the base template for a Bulwark plugin:

{% code title="Cargo.toml" %}

```toml
# Plugin definition and metadata.
[package]
name = "example-plugin" # required
edition = "2021" # required
version = "0.1.0" #recommended
publish = false # recommended

# Plugin dependencies.
[dependencies] # required
bulwark-wasm-sdk = "0.3.0"

# Library type configuration.
[lib] # required
crate-type = ["cdylib"]

# Use this section to optimize file size for release builds.
[profile.release] # recommended
lto = true
opt-level = 3
codegen-units = 1
panic = "abort"
strip = "debuginfo"
```

{% endcode %}

The configuration fields are defined in the sections below.

### [package]

This section defines the plugin and its metadata.

Read more about the `package` section [here](https://doc.rust-lang.org/cargo/reference/manifest.html#the-package-section).

| Field name   | Type    | Description                 | Required or recommended? |
| ---------------------- | ------- | --------------------------- | ------------------------ |
| `name`       | string  | The plugin name.            | Required    |
| `edition`    | string  | The plugin edition.         | Required    |
| `version`    | string  | The plugin version.         | Recommended |
| `publish`    | boolean | If the plugin is published. | Recommended |

### [dependencies]

This section defines the required version for the Bulwark WASM SDK.

> **NOTE:**
>
> Because Bulwark is in an open beta state, the SDK version may change frequently. Check the Bulwark project [releases](https://github.com/bulwark-security/bulwark/releases) for the most recent SDK version.

Read more about the `dependencies` section [here](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html).

| Field name         | Type   | Description | Required or recommended? |
| ------------------ | ------ | ----------- | ------------------------ |
| `bulwark-wasm-sdk` | string | The Bulwark WASM SDK version for the plugin. | Required |

### [lib]

This section defines the library type for the plugin.

Read more about the `lib` section [here](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#library).

| Field name   | Type   | Description | Required or recommended? |
| ------------ | ------ | ----------- | ------------------------ |
| `crate-type` | string | The type of library output. Must be `"cdylib"`. | Required |

### [profile.release]

This section defines options to optimize the file size for release builds.

Read more about `profile` settings [here](https://doc.rust-lang.org/cargo/reference/profiles.html#profile-settings).

| Field name      | Type    | Description | Required or recommended? |
| --------------- | ------- | ----------- | ------------------------ |
| `lto`           | boolean |             | Recommended              |
| `opt-level`     | integer |             | Recommended              |
| `codegen-units` | integer |             | Recommended              |
| `panic`         | string  |             | Recommended              |
| `strip`         | string  |             | Recommended              |

## Write the plugin logic

Write plugin logic under the `src` folder in a file named `lib.rs`.

The following code shows how to call the `bulwark_wasm_sdk` library:

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

## Compile the plugin

Use the built-in `build` subcommand from the `bulwark-cli` to compile Bulwark plugins:

```sh
bulwark-cli build
```

The default location for `build` output is the `dist/` directory. To change the output location, use the `-o`/`--output` option.

```sh
bulwark-cli build -o/--output <YOUR PATH HERE>
```

To learn more about `bulwark-cli` options, refer to the [CLI reference](https://docs.bulwark.security/ops/cli) page.

### Compile the plugin manually (aka the hard way)

Compiling manually is **not recommended**. Manual compilation is documented here to demonstrate compatibility with Rust's compiler output.

When compiling a plugin manually, the `build` command should set the `wasm32-wasi` environment as its target:

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

To generate the adapted plugin, run the following commands:

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
