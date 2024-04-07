# Engine Tests

To run the entire Bulwark engine test suite:

```bash
cargo test \
  -p bulwark-build \
  -p bulwark-cli \
  -p bulwark-config \
  -p bulwark-decision \
  -p bulwark-ext-processor \
  -p bulwark-wasm-host \
  -p bulwark-wasm-sdk \
  -p bulwark-wasm-sdk-macros
```

Bulwark currently uses GitHub Actions for its continuous integration and it's recommended that contributors verify that all tests are passing before submitting a new pull request.
