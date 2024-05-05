# Installation

Follow this guide to install Bulwark.

> :warning: **WARNING: This is a beta test.**
>
> Bulwark is currently in a public [beta test](../#this-is-a-beta-test.). It is not currently recommended for production use-cases.

## Prerequisites

Before installing Bulwark, complete the following:

* Install [rustup](https://rustup.rs/).
* Install the [Rust toolchain](https://www.rust-lang.org/tools/install).

## Install

### Cargo install

Install Bulwark via `cargo`:

```sh
cargo install bulwark-cli
```

To install the `bulwark-cli` binary from the repository's root directory instead of from the published crate, run the following `cargo` command:

```
cargo install --path .
```

### GitHub releases

Download a precompiled CLI binary from the Bulwark project's GitHub [releases](https://github.com/bulwark-security/bulwark/releases).

### Docker

There are currently no official Bulwark docker images published. 

Official docker images will be released once Bulwark publishes version 1.0.0.

## Build

1. Install the WebAssembly WASI target via `rustup`:

   ```
   rustup target add wasm32-wasi
   ```

1. Build Bulwark via `cargo`:

   ```
   cargo build --release
   ```

   The `bulwark-cli` binary is built in the `target/release` directory.
