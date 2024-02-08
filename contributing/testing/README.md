# Testing

Built-in testing capabilities are currently a feature on the [Roadmap](../roadmap.md) and intended to help test the accuracy of an entire ruleset. However individual plugins already can and should have their own unit test suites. Unit tests for plugins are an excellent way to help control accuracy regressions.

Contributing test cases to plugins in the [Community Ruleset](../community-ruleset.md) is an excellent way to get involved in the community! [Writing a new plugin test case](plugin-tests.md) is straight-forward and it's an excellent place to start, particularly if you are new to Rust.

The engine itself also benefits from its own test suite. Bulwark is composed of several Rust crates, each with their own independent test suite. Refer to the documentation on [engine tests](engine-tests.md) for instructions on running the full set. Contributions to the engine test suite are also welcomed!
