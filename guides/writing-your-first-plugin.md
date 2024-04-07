# Writing Your First Plugin

This tutorial will try to avoid assuming the reader is already familiar with any particular programming language, although some degree of coding knowledge is assumed.

{% tabs %}
{% tab title="Rust" %}
## Install Rust

Before writing a Rust-based plugin, we'll need the Rust toolchain available. You can skip this step if it's already available in your environment. [Full installation instructions](https://www.rust-lang.org/tools/install) cover additional details and environments, but for most \*nix based systems this is all you need:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

If you prefer to avoid shell pipe installs, there are [signed standalone installers](https://forge.rust-lang.org/infra/other-installation-methods.html#standalone-installers).

## Install Bulwark

Bulwark has its own build command, which we'll need to build our plugin. Once the Rust toolchain is installed, we can obtain the Bulwark CLI via [Cargo](https://doc.rust-lang.org/cargo/), which is included with the toolchain. Again, you can skip this step if the CLI is already installed.

```bash
cargo install bulwark-cli
```

## Blank Slate

We're going to start our plugin from a "blank slate" example. The [full example](https://github.com/bulwark-security/bulwark/tree/main/crates/wasm-sdk/examples/blank-slate) can be obtained from the Bulwark repository by cloning it. You will need [git installed](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) if it isn't already. We're going to be writing a simple plugin that treats excessively long `Content-Type` headers as suspicious. A number of historical exploit payloads have used this header and typically require much longer values than a legitimate client would send. A length check thus becomes a very simple form of anomaly detection.

```bash
# Inside your preferred workspace
git clone https://github.com/bulwark-security/bulwark.git
cp -rp bulwark/crates/wasm-sdk/examples/blank-slate ./long-content-type
cd long-content-type
```

This is the starting point for our plugin. All of the handlers we might use already have boilerplate we can start from. In this case, we're going to delete each handler we don't intend to use. In the code below, `HttpHandlers` is the [Rust trait](https://doc.rust-lang.org/book/ch10-02-traits.html) (essentially an interface) that is required for each Bulwark plugin. We use a special macro, `bulwark_plugin`, that makes it so that we don't need to implement every member of the trait and we can just focus on the main functionality of our plugin.

{% code title="src/lib.rs" %}
```rust
use bulwark_wasm_sdk::*;
use std::collections::HashMap;

pub struct BlankSlate;

#[bulwark_plugin]
impl HttpHandlers for BlankSlate {
    fn handle_request_enrichment(
        _request: Request,
        _params: HashMap<String, String>,
    ) -> Result<HashMap<String, String>, Error> {
        // Cross-plugin communication logic goes here, or leave as a no-op.
        Ok(HashMap::new())
    }

    fn handle_request_decision(
        _request: Request,
        _params: HashMap<String, String>,
    ) -> Result<HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
        // Main detection logic goes here.
        output.decision = Decision::restricted(0.0);
        output.tags = vec!["blank-slate".to_string()];
        Ok(output)
    }

    fn handle_response_decision(
        _request: Request,
        _response: Response,
        _params: HashMap<String, String>,
    ) -> Result<HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
        // Process responses from the interior service here, or leave as a no-op.
        output.decision = Decision::restricted(0.0);
        Ok(output)
    }

    fn handle_decision_feedback(
        _request: Request,
        _response: Response,
        _params: HashMap<String, String>,
        _verdict: Verdict,
    ) -> Result<(), Error> {
        // Feedback loop implementations go here, or leave as a no-op.
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Your unit tests go here.
}
```
{% endcode %}

## Writing the Plugin

We're only going to use the `handle_request_decision` function in this plugin, so we can go ahead, open the file in your preferred editor, and delete the rest. We can also update the plugin name and the tag name we're going to use.

<pre class="language-rust" data-title="src/lib.rs"><code class="lang-rust">use bulwark_wasm_sdk::*;
use std::collections::HashMap;

<strong>pub struct LongContentType;
</strong>
#[bulwark_plugin]
<strong>impl HttpHandlers for LongContentType {
</strong>    fn handle_request_decision(
        _request: Request,
        _params: HashMap&#x3C;String, String>,
    ) -> Result&#x3C;HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
        // Main detection logic goes here.
        output.decision = Decision::restricted(0.0);
<strong>        output.tags = vec!["long-content-type".to_string()];
</strong>        Ok(output)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Your unit tests go here.
}
</code></pre>

Next we need to get the value of the `Content-Type` header, if any. If there's no `Content-Type` header, our plugin will keep the default verdict, which is [uncertainty](../introduction/core-concepts/#decisions).

<pre class="language-rust" data-title="src/lib.rs"><code class="lang-rust">use bulwark_wasm_sdk::*;
use std::collections::HashMap;

pub struct LongContentType;

#[bulwark_plugin]
impl HttpHandlers for LongContentType {
    fn handle_request_decision(
<strong>        request: Request,
</strong>        _params: HashMap&#x3C;String, String>,
    ) -> Result&#x3C;HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
<strong>        if let Some(content_type) = request.headers().get("Content-Type") {
</strong><strong>            // Main detection logic goes here.
</strong><strong>            output.tags = vec!["long-content-type".to_string()];        
</strong><strong>        }
</strong>        Ok(output)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Your unit tests go here.
}
</code></pre>

Now we're going to check the length of the header and render our decision based on it. There isn't a strict limit on length given in the specification for the `Content-Type` header. However, clients tend to be fairly consistent in what they send, favoring shorter values. We're going to capitalize on this for our simple detection.

We'll start by counting how many extra characters we've found in the header value above the arbitrary limit we've set. The longest legitimate `Content-Type` values in common usage are around 100 characters, typically some flavor of `multipart/form-data`. The shortest viable exploit payloads we have in mind are right around 150 characters, so we're going to pick a number in between.

Let's also introduce our first unit test.

<pre class="language-rust" data-title="src/lib.rs"><code class="lang-rust">use bulwark_wasm_sdk::*;
use std::collections::HashMap;

pub struct LongContentType;

<strong>const MAX_LEN: usize = 120;
</strong>
<strong>impl LongContentType {
</strong><strong>    fn chars_above_max(content_type: &#x26;HeaderValue) -> usize {
</strong><strong>        (content_type.len() - MAX_LEN).max(0)
</strong><strong>    }
</strong><strong>}
</strong>
#[bulwark_plugin]
impl HttpHandlers for LongContentType {
    fn handle_request_decision(
        request: Request,
        _params: HashMap&#x3C;String, String>,
    ) -> Result&#x3C;HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
        if let Some(content_type) = request.headers().get("Content-Type") {
<strong>            let chars_above_max = LongContentType::chars_above_max(content_type);
</strong><strong>            if chars_above_max > 0 {
</strong><strong>                output.tags = vec!["long-content-type".to_string()];
</strong><strong>            }
</strong>            // Next we'll need to convert to a decision here.
        }
        Ok(output)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

<strong>    #[test]
</strong><strong>    fn test_chars_above_max() {
</strong><strong>        let test_cases = vec![
</strong><strong>            (HeaderValue::from_static("application/json"), 0),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    "multipart/form-data; boundary=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
</strong><strong>                ),
</strong><strong>                0,
</strong><strong>            ),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    r"%{(#_='multipart/form-data').(#_memberAccess=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(@java.lang.Runtime@getRuntime().exec('curl www.example.com'))}",
</strong><strong>                ),
</strong><strong>                29,
</strong><strong>            ),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    r"%{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='curl www.example.com').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}",
</strong><strong>                ),
</strong><strong>                704,
</strong><strong>            ),
</strong><strong>        ];
</strong><strong>        for (content_type, expected) in test_cases {
</strong><strong>            let chars_above_max = LongContentType::chars_above_max(&#x26;content_type);
</strong><strong>            assert_eq!(chars_above_max, expected);
</strong><strong>        }
</strong><strong>    }
</strong>}
</code></pre>

The last step is to take the excess we calculated in the previous step and turn it into a [decision value](../introduction/core-concepts/#decisions). Bulwark uses decision values to render its verdict on whether a request should be blocked or not. Crucially, it encodes uncertainty into these values. In our example, there's nothing that forbids a client from sending a long `Content-Type`. It's not a guarantee of malicious behavior and to control false positives, we don't want to treat it as malicious until the evidence is very strong.

We're going to generate a score value in the range 0.0 to 0.75 to represent our confidence that the request is malicious. We stop at 0.75 because 1.0 would indicate certainty and we want to retain some of our uncertainty based on our detection method. We stay below 0.8 because that's the default threshold to block a request and we want multiple detections to contribute to a verdict before we block outright. We want to scale linearly and hit the maximum value when a request goes over by 150 characters. We know that exploits will go far past this secondary limit, as seen in the final test case from the previous change we made. That makes this detection harder to bypass. We can calculate our scaling factor by dividing 0.75 by 150 to get 0.005. Add in a simple clamp for the maximum score and that gives us our scoring function.

For testing, we're going to introduce the [`approx`](https://crates.io/crates/approx) crate as a [`dev-dependency`](https://doc.rust-lang.org/rust-by-example/testing/dev\_dependencies.html) to simplify working with floating point score values.

<pre class="language-rust" data-title="src/lib.rs"><code class="lang-rust">use bulwark_wasm_sdk::*;
use std::collections::HashMap;

pub struct LongContentType;

const MAX_LEN: i64 = 120;
<strong>const MAX_EXCESS: i64 = 150;
</strong><strong>const MAX_SCORE: f64 = 0.75;
</strong><strong>const SCALE_FACTOR: f64 = MAX_SCORE / MAX_EXCESS as f64;
</strong>
impl LongContentType {
<strong>    fn score_content_type(content_type: &#x26;HeaderValue) -> f64 {
</strong><strong>        let chars_above_max = Self::chars_above_max(content_type) as f64;
</strong><strong>        (chars_above_max * SCALE_FACTOR).min(MAX_SCORE)
</strong><strong>    }
</strong>
    fn chars_above_max(content_type: &#x26;HeaderValue) -> usize {
        (content_type.len() as i64 - MAX_LEN).max(0) as usize
    }
}

#[bulwark_plugin]
impl HttpHandlers for LongContentType {
    fn handle_request_decision(
        request: Request,
        _params: HashMap&#x3C;String, String>,
    ) -> Result&#x3C;HandlerOutput, Error> {
        let mut output = HandlerOutput::default();
        if let Some(content_type) = request.headers().get("Content-Type") {
<strong>            let score = LongContentType::score_content_type(content_type);
</strong><strong>            if score > 0.0 {
</strong>                output.tags = vec!["long-content-type".to_string()];
            }
<strong>            output.decision = Decision::restricted(score);
</strong>        }
        Ok(output)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
<strong>    use approx::assert_relative_eq;
</strong>
<strong>    #[test]
</strong><strong>    fn test_score_content_type() {
</strong><strong>        let test_cases = vec![
</strong><strong>            (HeaderValue::from_static("application/json"), 0.0),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    "multipart/form-data; boundary=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
</strong><strong>                ),
</strong><strong>                0.0,
</strong><strong>            ),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    r"%{(#_='multipart/form-data').(#_memberAccess=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(@java.lang.Runtime@getRuntime().exec('curl www.example.com'))}",
</strong><strong>                ),
</strong><strong>                0.145,
</strong><strong>            ),
</strong><strong>            (
</strong><strong>                HeaderValue::from_static(
</strong><strong>                    r"%{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='curl www.example.com').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}",
</strong><strong>                ),
</strong><strong>                0.75,
</strong><strong>            ),
</strong><strong>        ];
</strong><strong>        for (content_type, expected) in test_cases {
</strong><strong>            let score = LongContentType::score_content_type(&#x26;content_type);
</strong><strong>            assert_relative_eq!(score, expected);
</strong><strong>        }
</strong><strong>    }
</strong>
    #[test]
    fn test_chars_above_max() {
        let test_cases = vec![
            (HeaderValue::from_static("application/json"), 0),
            (
                HeaderValue::from_static(
                    "multipart/form-data; boundary=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
                ),
                0,
            ),
            (
                HeaderValue::from_static(
                    r"%{(#_='multipart/form-data').(#_memberAccess=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(@java.lang.Runtime@getRuntime().exec('curl www.example.com'))}",
                ),
                29,
            ),
            (
                HeaderValue::from_static(
                    r"%{(#_='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='curl www.example.com').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}",
                ),
                704,
            ),
        ];
        for (content_type, expected) in test_cases {
            let chars_above_max = LongContentType::chars_above_max(&#x26;content_type);
            assert_eq!(chars_above_max, expected);
        }
    }
}

</code></pre>

Our final detection is difficult to bypass effectively, addresses false positive risks, gives us additional traffic visibility by tagging offending requests in our logs, and has an embedded test suite that gives us confidence that we've implemented our logic correctly.

The [finished version of this plugin](https://github.com/bulwark-security/bulwark-community-ruleset/tree/main/rules/long-content-type) is available in the community ruleset.
{% endtab %}
{% endtabs %}
