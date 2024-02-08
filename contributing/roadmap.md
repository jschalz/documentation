---
description: Here's what's on the horizon!
---

# Roadmap

* [x] Logging for plugin stdout
* [x] Request and response body handling
* [ ] Language support
  * [ ] Python SDK
  * [ ] Go (Tinygo) SDK
  * [ ] AssemblyScript SDK
* [ ] Run Bulwark in other configurations:
  * [ ] As its own stand-alone reverse proxy
  * [ ] As an Nginx module
  * [ ] As an Apache module
* [ ] Built-in ruleset integration testing capability
* [ ] Metrics & distributed tracing
  * [ ] OpenTelemetry support
  * [x] Prometheus metrics support
  * [x] StatsD support
  * [x] Built-in per-plugin metrics
  * [ ] Allow plugins to emit custom metrics with a permission grant
  * [ ] Calculate the degree-of-disagreement between plugins
* [ ] Custom restrict responses, default and per-endpoint
* [ ] Sign injected header values
* [ ] Plugin access to the file-system behind a permission grant
* [ ] Client geo-location support, possibly via delegation to a plugin
* [ ] Subscribe to an external configuration management API
* [ ] Plugin changes deployable in "audit mode"
  * [ ] Log the delta between deployed configuration and proposed configuration
