---
title: 7. Optional Capabilities
sidebar_position: 7
---

# 7 · Optional Capabilities

MCS keeps the base contract tiny. Optional behavior is signaled via **capability flags** in `DriverMeta`. Consumers must *feature-detect* before invoking an optional method (i.e., check if the flag exists in `meta.capabilities` and then dynamically call the corresponding method).

Extend via capabilities, e.g.:

| Capability       | Flag            | Suggested Mix-in / Interface | Description |
| ---------------- | --------------- | ---------------------------- | ----------- |
| Health check     | `healthcheck`   | `abstract class SupportsHealthcheck { abstract healthcheck() -> dict }`  | Returns status info, e.g., `{"status": "OK"}`. |
| Resource preload | `cache`         | `abstract class SupportsCache { abstract warmup() -> void }`       | Preloads resources for faster execution. |
| Status & metrics | `status`        | `abstract class SupportsStatus { abstract get_status() -> dict }`   | Provides runtime metrics or detailed status. |
| Autostart        | `autostart`     | `abstract class SupportsAutostart { abstract autostart(kwargs: dict) -> void }` | Launches required infrastructure (e.g., containers). |
| Driver context   | `driver_context` | `abstract class SupportsDriverContext { abstract get_driver_context() -> DriverContext }` | Provides structured tool definitions for native tool-calling APIs (see below). |


**Rule of Thumb:** For easy use cases, name the mixin class `Supports<CapabilityName>` with the method named `<capabilityName>`. This convention simplifies dynamic invocation but is not mandatory. SDKs may define their own standards for common capabilities.


## DriverContext: Native Tool-Calling Support

The core MCS contract is text-centric: `get_driver_system_message()` returns a string, `process_llm_response()` accepts a string or dict. This works universally -- every LLM can consume text prompts.

However, some LLMs (notably OpenAI's GPT family and Anthropic's Claude) offer **native tool-calling APIs** where tools are passed as structured objects alongside the system message, rather than embedded in the prompt text. The LLM then returns structured `tool_calls` objects instead of text that needs parsing.

The `DriverContext` capability bridges this gap. A driver that supports it can provide its tools in the structured format that native tool-calling APIs expect:

```pseudo
struct DriverContext {
    system_message: string           // the system prompt (may exclude tool descriptions)
    tools: array[dict]               // tools in the LLM provider's native format (e.g. OpenAI function schema)
}

abstract class SupportsDriverContext {
    abstract get_driver_context(model_name?: string) -> DriverContext
}
```

### How it works

When using native tool-calling:

1. The client calls `get_driver_context()` instead of `get_driver_system_message()`.
2. The returned `system_message` contains only the behavioral prompt (usage instructions, formatting guidance) -- tool descriptions are **not** inlined, since they are provided separately in `tools`.
3. The `tools` array contains tool definitions in the provider's native schema (e.g. OpenAI's `{"type": "function", "function": {"name": ..., "parameters": ...}}` format).
4. The client passes `system_message` as the system prompt and `tools` as the `tools` parameter to the LLM API.
5. The LLM responds with structured `tool_calls` objects. The client passes these to `process_llm_response()`, which accepts dicts via the `ExtractionStrategy` chain (see [Section 10](10_LLM_Prompt_Patterns/LLM_Prompt_Patterns.md)).

### Why this is optional

Native tool-calling is not universally supported. Many models (open-source, local, or older commercial models) only work with text prompts. The text-based path (`get_driver_system_message()` + text parsing) remains the universal default. `DriverContext` is an optimization for clients that target specific providers.

### Relationship to DriverBase

In the Python SDK, `DriverBase` implements `SupportsDriverContext` by default. It derives the `tools` array from the `MCSToolDriver.list_tools()` output, converting each `Tool` into the OpenAI function-calling schema. The `system_message` is generated from the `PromptStrategy` but without inlined tool descriptions. This means any driver that inherits from `DriverBase` automatically supports native tool-calling without additional code.