---
description: An extensible schema definition for inter-plugin communication
---

# Bulwark Label Schema

## Overview

To support communication between plugins without the need to explicitly declare plugin dependencies, it is recommended that plugins name and format labels according to this schema definition. Plugins may use label names that do not appear in this schema, with the expectation that out-of-band coordination between plugins may be required in such cases. Ideally, such extensions should be namespaced appropriately, e.g. `plugin_name.extension`. This schema is inspired by the [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-reference.html) (ECS) and while large swaths of ECS are inapplicable to this use-case, it may still be used as a reference for any gaps. Bulwark also uses ECS for [logging and observability](../ops/observability.md).

This schema is incomplete and suggestions for expanding it are welcomed. Refer to the [ECS guidelines for naming](https://www.elastic.co/guide/en/ecs/current/ecs-guidelines.html#\_guidelines\_for\_field\_names).

## Schema Reference

* `route.*`
  * These labels will automatically be set by the host environment when extracted by the configured [resource route patterns](../ops/configuration.md#route).
* `user.id`
  * A unique identifier for the user record. May be used in conjunction with rate-limiting plugins, anti-fraud plugins, or any other use-case that benefits from a stable identifier which is independent of network or device information. If this value is an email address, the `user.email` label value should also be populated with the same value.
* `user.email`
  * The email address for the user record.
* `user.name`
  * The short name, or username, for the user record.
* `user.hash`
  * In some cases, usage of `user.id`, `user.email`, and/or `user.name` label values may be disallowed by an organization policy. The `user.hash` value should be a privacy-preserving hash value that anonymizes the user while retaining most or all of the ability to differentiate users.
* `client.as.number`
  * Unique number allocated to the autonomous system. The autonomous system number (ASN) uniquely identifies each network on the Internet.\
    \
    **Example:**\
    15169
* `client.geo.country_iso_code`
  * The country ISO code where the client is believed to be located, either derived from the client IP address or from a client geo-location API.
* `client.geo.location`
  * The latitude and longitude where the client is believed to be located, either derived from the client IP address or from a client geo-location API.\
    \
    **Example:**\
    `{ "lon": -73.614830, "lat": 45.505918 }`
* `device.id`
  * The unique identifier for a device. This identifier should be stable across multiple application sessions. For iOS devices, this should be the [vendor identifier](https://developer.apple.com/documentation/uikit/uidevice/1620059-identifierforvendor). For Android, it should either be the [Firebase Installation ID](https://firebase.google.com/docs/projects/manage-installations) or a globally unique UUID which is persisted across sessions in your application. This identifier must not carry information which would uniquely identify a user.
* `device.fingerprint`
  * Plugins may relay a device fingerprint via this field. It is recommended that fingerprints be truncated for [k-anonymity](https://en.wikipedia.org/wiki/K-anonymity) whenever possible, to avoid uniquely identifying a user with one. There are many trade-offs inherent with fingerprinting methods and implementors should use appropriate caution when deploying one. Fingerprints should be expected to be stable over short periods of time and unstable over longer periods.\
    \
    **Example:**\
    `"3a54"`
