# Plugin Tests

Plugins should have a unit test suite, like any other code you might run in production. Bulwark plugins use standard test suites for their corresponding language.

## Running Plugin Unit Tests

{% tabs %}
{% tab title="Rust" %}
Rust plugins may be tested like any other Rust crate:

```
cargo test
```

In Rust, typically unit tests live in the same module as the code they're testing and Bulwark plugins are no different.

```rust
use bulwark_wasm_sdk::*;

pub struct DetectionPlugin;

#[bulwark_plugin]
impl HttpHandlers for DetectionPlugin {
    // Your detection logic goes here.
}

#[cfg(test)]
mod tests {
    use super::*;

    // Your unit tests go here.
}
```

Currently, due to the way the handler macros work, it's better to extract the logic you intend to test into helper functions rather than trying to test handler functions directly. Simplified testing of handler functions will be supported in a future release.
{% endtab %}
{% endtabs %}
