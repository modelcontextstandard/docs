---
title: 6. Optional Capabilities
sidebar_position: 6
---

# 6 · Optional Capabilities

MCS keeps the base contract tiny. Optional behavior is signaled via **capability flags** in `DriverMeta`. Consumers must *feature-detect* before invoking an optional method (i.e., check if the flag exists in `meta.capabilities` and then dynamically call the corresponding method).

Extend via capabilities, e.g.:

| Capability       | Flag            | Suggested Mix-in / Interface | Description |
| ---------------- | --------------- | ---------------------------- | ----------- |
|Health check      | `healthcheck`   | ```ts abstract class SupportsHealthcheck { abstract healthcheck() -> dict }``` |Returns status info, e.g., ```ts {"status": "OK"}```.|
| Resource preload | `cache`         | ```ts abstract class SupportsCache { abstract warmup() -> void }``` | Preloads resources for faster execution. |
| Status & metrics | `status`        | ```ts abstract class SupportsStatus { abstract get_status() -> dict }``` | Provides runtime metrics or detailed status. |
| Autostart        | `autostart`     | ```ts abstract class SupportsAutostart { abstract autostart(kwargs: dict) -> void }``` | Launches required infrastructure (e.g., containers). | 


**Rule of Thumb:** For easy use cases, name the mixin class `Supports<CapabilityName>` with the method named `<capabilityName>`. This convention simplifies dynamic invocation but is not mandatory. SDKs may define their own standards for common capabilities.