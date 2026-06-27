---
title: 8. Optional Capabilities
sidebar_position: 8
---

# 8 · Optional Capabilities

MCS keeps the base contract tiny. Optional behavior is signaled via **capability flags** in `DriverMeta`. Consumers must *feature-detect* before invoking an optional method: check that the flag is present in `meta.capabilities`, then call the method on the layer that [`resolve_capability`](#how-drivermeta-carries-capabilities) returns — *not necessarily on the driver instance you hold*, since a [decorator](6_Decorators.md) delegates the interface but not arbitrary methods. The [client-side pattern](#using-a-capability-from-the-clients-side) below shows the full detect → resolve → call sequence.

Extend via capabilities, e.g.:

| Capability       | Flag            | Suggested Mix-in / Interface | Description |
| ---------------- | --------------- | ---------------------------- | ----------- |
| Health check     | `healthcheck`   | `abstract class SupportsHealthcheck { abstract healthcheck() -> dict }`  | Returns status info, e.g., `{"status": "OK"}`. |
| Resource preload | `cache`         | `abstract class SupportsCache { abstract warmup() -> void }`       | Preloads resources for faster execution. |
| Status & metrics | `status`        | `abstract class SupportsStatus { abstract get_status() -> dict }`   | Provides runtime metrics or detailed status. |
| Autostart        | `autostart`     | `abstract class SupportsAutostart { abstract autostart(kwargs: dict) -> void }` | Launches required infrastructure (e.g., containers). |
| Native tools     | `native_tools` | `abstract class SupportsNativeTools { abstract get_native_tool_context() -> NativeToolContext }` | Provides structured tool definitions for native tool-calling APIs (see below). |


**Rule of Thumb:** For easy use cases, name the mixin class `Supports<CapabilityName>` with the method named `<capabilityName>`. This convention simplifies dynamic invocation but is not mandatory. SDKs may define their own standards for common capabilities.


## How `DriverMeta` carries capabilities

A capability has two halves: the **contract** (the `Supports…` interface plus
its `CAPABILITY` flag) and the **metadata** that advertises it. `DriverMeta`
carries the metadata and offers three operations. The reason they exist —
rather than a simple `isinstance` check — is **composition**: capabilities that
*intervene in how MCS handles a call* (authentication, permission, lifecycle
hooks) are built as [decorators](6_Decorators.md) — a driver that wraps another
driver. A wrapper hides the inner layers, so an `isinstance` check (which only
sees the outermost object) is no longer enough. These three operations make a
stack of wrapped drivers look like *one* driver to the client:

- **Declaration — `meta.with_capability(Contract)`** returns a copy of the
  metadata with the contract's `CAPABILITY` flag added (idempotent). A plain
  driver may list its flags explicitly; a reference base (like `BaseDriver`)
  may add its own automatically; a decorator **aggregates** the inner driver's
  flags and appends its own — so `meta.capabilities` always reflects the
  *whole* stack.
- **Detection — `meta.has_capability(Contract)`** is a pure read over those
  aggregated flags: *"is this capability present anywhere in the stack?"* It
  replaces `isinstance`, which would miss a capability provided deeper down.
- **Resolution — `DriverMeta.resolve_capability(driver, Contract)`** returns
  the actual layer that satisfies the contract (typed, ready to call) by
  searching inward through the stack. A plain driver is matched directly; a
  wrapper delegates the search to the driver it wraps (via the optional
  `SupportsCapabilityResolution` contract).

### Using a capability from the client's side

The three operations combine into one **detect → resolve → call** sequence. The
key point: you call the method on what `resolve_capability` *returns*, not on the
driver you hold — for a decorated or orchestrated stack those are different
objects.

```python
# `driver` may be a plain driver, an orchestrator, or a decorator stack —
# the client does not know or care.
if driver.meta.has_capability(SupportsHealthcheck):
    layer = DriverMeta.resolve_capability(driver, SupportsHealthcheck)
    result = layer.healthcheck()        # called on the resolving layer
```

If `driver` is a plain driver, `resolve_capability` returns it unchanged and the
call lands directly. If `driver` is, say, an `AuthDecorator` wrapping a
healthcheck-capable ToolDriver, the flag is still visible — the decorator
*aggregated* it — but the method lives on the inner driver, and
`resolve_capability` walks inward to hand you that layer. Calling
`driver.healthcheck()` directly would fail: a decorator delegates the
*interface* (`list_tools`, `execute_tool`), **not** arbitrary methods.

Why on `DriverMeta` and not on the driver interface? So the **core `MCSDriver`
contract stays minimal** — it gains no methods for this. Detection and
declaration are pure metadata reads/writes (instance methods on the metadata);
resolution navigates the object stack, so it is a *static* helper that takes
the driver explicitly (the shared class-level metadata cannot reach a specific
driver instance). The payoff: the **driver author stays simple** — a plain
driver needs none of this — and the **client treats every `MCSDriver` the
same**, whether it is a plain driver, an orchestrator, or a decorator, no
matter what was injected.


## NativeToolContext: Native Tool-Calling Support

The core MCS contract is text-centric: `get_driver_system_message()` returns a string, `process_llm_response()` accepts a string or dict. This works universally -- every LLM can consume text prompts.

However, some LLMs (notably OpenAI's GPT family and Anthropic's Claude) offer **native tool-calling APIs** where tools are passed as structured objects alongside the system message, rather than embedded in the prompt text. The LLM then returns structured `tool_calls` objects instead of text that needs parsing.

The `NativeToolContext` capability bridges this gap. A driver that supports it can provide its tools in the structured format that native tool-calling APIs expect:

```pseudo
struct NativeToolContext {
    system_message: string           // the system prompt (may exclude tool descriptions)
    tools: array[dict]               // tools in the LLM provider's native format (e.g. OpenAI function schema)
}

abstract class SupportsNativeTools {
    abstract get_native_tool_context(model_name?: string) -> NativeToolContext
}
```

### How it works

When using native tool-calling:

1. The client calls `get_native_tool_context()` instead of `get_driver_system_message()`.
2. The returned `system_message` contains only the behavioral prompt (usage instructions, formatting guidance) -- tool descriptions are **not** inlined, since they are provided separately in `tools`.
3. The `tools` array contains tool definitions in the provider's native schema (e.g. OpenAI's `{"type": "function", "function": {"name": ..., "parameters": ...}}` format).
4. The client passes `system_message` as the system prompt and `tools` as the `tools` parameter to the LLM API.
5. The LLM responds with structured `tool_calls` objects. The client passes these to `process_llm_response()`, which accepts dicts via the `ExtractionStrategy` chain (see [Section 11](11_LLM_Prompt_Patterns/LLM_Prompt_Patterns.md)).

### Why this is optional

Native tool-calling is not universally supported. Many models (open-source, local, or older commercial models) only work with text prompts. The text-based path (`get_driver_system_message()` + text parsing) remains the universal default. `NativeToolContext` is an optimization for clients that target specific providers.

### Relationship to BaseDriver

In the Python SDK, `BaseDriver` implements `SupportsNativeTools` by default. It derives the `tools` array from the `MCSToolDriver.list_tools()` output, converting each `Tool` into the OpenAI function-calling schema. The `system_message` is generated from the `PromptStrategy` but without inlined tool descriptions. This means any driver that inherits from `BaseDriver` automatically supports native tool-calling without additional code.