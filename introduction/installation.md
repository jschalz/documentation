# Installation

## Install Rust

Before installing Bulwark, we'll need the Rust toolchain available. You can skip this step if it's already available in your environment. [Full installation instructions](https://www.rust-lang.org/tools/install) cover additional details and environments, but for most \*nix based systems this is all you need:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

If you prefer to avoid shell pipe installs, there are [signed standalone installers](https://forge.rust-lang.org/infra/other-installation-methods.html#standalone-installers).

## Install Bulwark

Once the Rust toolchain is installed, we can obtain the Bulwark CLI via [Cargo](https://doc.rust-lang.org/cargo/), which is included with the toolchain.

```bash
cargo install bulwark-cli
```

## GitHub Releases

Precompiled CLI binaries may also be downloaded from the project's GitHub [releases](https://github.com/bulwark-security/bulwark/releases).

## Docker

Currently, no official project Docker images are being published. Official Docker images will be released once Bulwark publishes a version 1.0.0.

## Building

Before building Bulwark, the WebAssembly WASI target should be installed.

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
