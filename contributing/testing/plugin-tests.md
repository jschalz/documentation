# Plugin Tests

## Running Plugin Unit Tests

{% tabs %}
{% tab title="Rust" %}
Rust plugins may be tested like any other Rust crate:

```
cargo test
```
{% endtab %}
{% endtabs %}

## Example Plugin Unit Tests

The following is a plugin example that checks request body size against a soft and a hard limit. It is recommended that plugin logic primarily lives outside the handler functions to facilitate more effective testing.

{% tabs %}
{% tab title="Rust" %}
```rust
use bulwark_wasm_sdk::*;

/// A soft limit will be applied to requests above 15 MiB
const SOFT_LIMIT: u64 = 15 * 1048576;

/// Maximum acceptable request body length is 50 MiB
const HARD_LIMIT: u64 = 50 * 1048576;

#[derive(Debug, PartialEq, Eq)]
enum BodyLimit {
    Normal,
    SoftLimit,
    HardLimit,
}

/// Checks if the request is above either the hard or soft limit
fn body_limit(size: u64) -> BodyLimit {
    match size {
        x if x > HARD_LIMIT => BodyLimit::HardLimit,
        x if x > SOFT_LIMIT => BodyLimit::SoftLimit,
        _ => BodyLimit::Normal,
    }
}

struct SizeLimitPlugin;

#[bulwark_plugin]
impl Handlers for SizeLimitPlugin {
    fn on_request_decision() -> Result {
        let request = get_request();
        let content_length = request
            .headers()
            .get("Content-Length")
            .and_then(|hv| hv.to_str().ok())
            .and_then(|hv| hv.parse().ok());
        if let Some(content_length) = content_length {
            match body_limit(content_length) {
                BodyLimit::HardLimit => set_restricted(1.0),
                // This generally will not result in a restrict decision in isolation
                BodyLimit::SoftLimit => set_restricted(0.15),
                BodyLimit::Normal => (),
            }
        }
        Ok(())
    }

    fn on_request_body_decision() -> Result {
        let request = get_request();
        match body_limit(request.body().size) {
            BodyLimit::HardLimit => set_restricted(1.0),
            // This generally will not result in a restrict decision in isolation
            BodyLimit::SoftLimit => set_restricted(0.15),
            BodyLimit::Normal => (),
        }
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_body_limit() -> std::result::Result<(), Box<dyn std::error::Error>> {
        let test_cases = [
            (
                // This request has no body
                0,
                BodyLimit::Normal,
            ),
            (
                // This request has a body that is exactly at the soft limit
                SOFT_LIMIT,
                BodyLimit::Normal,
            ),
            (
                // This request has a body that is exactly at the hard limit
                HARD_LIMIT,
                BodyLimit::SoftLimit,
            ),
            (
                // This request has a body above the upper limit
                HARD_LIMIT + 1,
                BodyLimit::HardLimit,
            ),
        ];

        for (size, expected) in test_cases {
            let limit = body_limit(size);
            assert_eq!(limit, expected);
        }

        Ok(())
    }
}
```
{% endtab %}
{% endtabs %}
