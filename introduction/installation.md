# Installation

## This is a beta test.

Before covering the topic of installation, it's important to mention that this is a [beta test](../#this-is-a-beta-test.). Bulwark is not currently recommended for production use-cases.

## Cargo Install

To install Bulwark, you will need the [Rust toolchain installed first](https://www.rust-lang.org/tools/install). We generally recommend installing it via `rustup`. Then Bulwark can be installed via `cargo`:

```
cargo install bulwark-cli
```

## GitHub Releases

Precompiled CLI binaries may also be downloaded from the project's GitHub [releases](https://github.com/bulwark-security/bulwark/releases).

## Docker

Currently, no official project docker images are being published. Official docker images will be released once Bulwark publishes a version 1.0.0.

## Building

Before building Bulwark, the WebAssembly WASI target is required to be installed.

```
rustup target add wasm32-wasi
```

It is otherwise a standard build process.

```
cargo build --release
```

The `bulwark-cli` binary will be in the `target/release` directory. Additionally, it is possible to install the `bulwark-cli` binary from the repository's root directory instead of from the published crate:

```
cargo install --path .
```
