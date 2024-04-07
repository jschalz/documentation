# Getting Started

## Overview

### Installation

Before anything else, make sure Bulwark has been [installed](installation.md).

### Envoy

Bulwark is an Envoy external processor. It needs to be used with Envoy to function. Future [roadmap](../contributing/roadmap.md) plans include support for other deployment patterns for request processing. The [deployment guide](../ops/deployment.md) covers setting up Envoy to use Bulwark.

### Plugins

Bulwark relies on WebAssembly plugins to perform all of its detection logic. You will need a collection of plugins to power Bulwark's detection capabilities. You can use the [community ruleset](https://github.com/bulwark-security/bulwark-community-ruleset) to get started. Or, see the tutorial on [writing your first plugin](../guides/writing-your-first-plugin.md) to supply your own.

### Configuration

Bulwark's [configuration file](../ops/configuration.md) determines which plugins are loaded, from where, and on which paths those plugins will be applied.
