# Bulwark Parameter Schema

An extensible schema definition for inter-plugin communication.

> **NOTE:**
>
> This schema is incomplete and suggestions for expanding it are welcomed!

## Overview

To support communication between plugins without requiring explicitly declared plugin dependencies, name and format parameters according to given schema definition.

If your plugin requires parameter values not included in this schema, the value names must be namespaced appropriately, e.g. `plugin_name.extension`. Out-of-band coordination may be necessary between plugins if custom parameter values are used.

The Bulwark parameter schema is inspired by the [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-reference.html) (ECS). Much of ECS is inapplicable to this use-case, but it should still be used as a reference for any gaps. Refer to the [ECS guidelines for naming](https://www.elastic.co/guide/en/ecs/current/ecs-guidelines.html#\_guidelines\_for\_field\_names).

Bulwark also uses ECS for [logging and observability](../ops/observability.md).

## Schema reference

### `param.*`

These parameters are automatically set by the host environment when extracted by the configured [resource route patterns](../ops/configuration.md#route).

Example: if an application registers users on the API path `/api/:user/register`, a user with the ID `123` accesses `/api/123/register`. The `param` schema creates a `user` as `param.user` with the value `123`.

### `client.*`

* [client.as.number](#clientasnumber)

#### `client.as.number`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `integer` | The unique number allocated to the autonomous system (AS). | 15169 |

The autonomous system number (ASN) uniquely identifies each network on the Internet.

### `client.geo.*`

* [client.geo.country_iso_code](#clientgeocountry_iso_code)
* [client.geo.location](#clientgeolocation)

#### `client.geo.country_iso_code`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The client's country 2-letter ISO code as defined by [ISO-3166 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2). | |

The `country_iso_code` can either derive from the client's IP address or from a client's geo-location API response.

#### `client.geo.location`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| object`{}` | The client's location in atitude and longitude coordinates, either derived from the client's IP address or from a client's geo-location API response. | `{ "lon": -73.614830, "lat": 45.505918 }` |

### `device.*`

* [device.fingerprint](#devicefingerprint)
* [device.id](#deviceid)

#### `device.fingerprint`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The device fingerprint. | `"3a54"` |

Use this field to relay a device fingerprint in a plugin.

Implementors must use caution when deploying device fingerprints. Fingerprints must only be stable over short periods of time.

**Recommended:** Truncate the device fingerprint for [k-anonymity](https://en.wikipedia.org/wiki/K-anonymity) whenever possible to avoid uniquely identifying a user with them.

#### `device.id`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The unique identifier for a device. | |

The `device.id` must be stable across multiple application sessions, and must not carry information that uniquely identifies a user.

For iOS devices, use the [vendor identifier](https://developer.apple.com/documentation/uikit/uidevice/1620059-identifierforvendor).

For Android, use either the [Firebase Installation ID](https://firebase.google.com/docs/projects/manage-installations) or a globally unique UUID persisted across sessions in your application.

### `user.*`

* [user.email](#useremail)
* [user.hash](#userhash)
* [user.id](#userid)
* [user.name](#username)

#### `user.email`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The user's email address. | |

#### `user.hash`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | A hash value identifier for the user. | |

In some cases, `user.id`, `user.email`, and `user.name` parameter values may be disallowed by security policies. The `user.hash` value should be a privacy-preserving hash value that anonymizes the user while retaining most or all of the ability to uniquely identify the user.

#### `user.id`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The user's unique identifier. | |

The `user.id` may be used in conjunction with rate-limiting plugins, anti-fraud plugins, or any other use-case that benefits from a stable identifier independent of network or device information.

If this value is an email address, the `user.email` parameter must be the same value.

#### `user.name`

| Type      | Description | Example |
| --------- | ----------- | ------- |
| `string` | The user's short name, or "username". | |
