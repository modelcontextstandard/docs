---
title: MCS Driver Contract
sidebar_position: 2
---

# MCS Driver Contract – Version 0.3

This document defines the minimal contract all MCS-compatible drivers must implement.

The syntax is language-agnostic and intended to guide implementations across all runtimes.

```pseudo
struct Binding {
    protocol: string  // "REST", "GraphQL", "SOAP", etc.
    transport: string  // "HTTP", "MQTT", "gRPC", etc.
    spec_format: string  // "OpenAPI", "WSDL", "Custom", etc.
}

struct DriverMeta {
    id: string  // UUID for unique identification
    name: string  // Human-readable name
    version: string  // Semantic version, e.g. "1.0.0"
    bindings: array[Binding]  // Supported bindings
    target_llms: array[string]  // "*" or specific models like "claude-3", None for ToolDrivers
    capabilities: array[string]  // e.g. "healthcheck", "autostart"
}

abstract class MCSDriver {
    meta: DriverMeta

    // --- per-call state (reset at the start of every process_llm_response call) ---
    call_executed: boolean = false   // true after a tool call was successfully executed
    call_failed: boolean = false      // true if a tool call signature was found but could not be parsed or executed
    last_call_detail: string? = null // optional: reason when call_failed is true (for debugging)

    abstract get_function_description(model_name?: string) -> string  // Machine-readable spec

    abstract get_driver_system_message(model_name?: string) -> string  // Full system prompt

    abstract process_llm_response(llm_response: string) -> any  // Execute if call detected, else return input

    get_retry_prompt() -> string?  // Returns a prompt hint when call_failed is true, null otherwise
}
```

**Per-call state:** `call_executed`, `call_failed` and `last_call_detail` are reset at the beginning of every `process_llm_response()` invocation. They reflect the outcome of the most recent call only. This makes the driver stateful per-call but not per-conversation -- conversation history remains the client's responsibility.

**Internal self-healing:** Before setting `call_failed = true`, the driver should attempt any configured self-healing patterns (e.g. fixing known model-specific formatting errors). Only if self-healing also fails does the driver signal the failure to the client.

**`get_retry_prompt()`:** Returns a driver-authored prompt hint that the client can append to the conversation when `call_failed` is true. This keeps prompt knowledge inside the driver. Returns null when there is nothing to retry.

### ToolDriver (for Orchestration)

```pseudo
struct ToolParameter {
    name: string
    description: string
    required: boolean = false
    schema?: dict  // e.g. {"type": "string", "enum": [...]}
}

struct Tool {
    name: string
    description: string
    parameters: array[ToolParameter]
}

abstract class MCSToolDriver {
    meta: DriverMeta  // target_llms = null, as it's orchestrator-facing

    abstract list_tools() -> array[Tool]  // List available tools

    abstract execute_tool(tool_name: string, arguments: dict) -> any  // Execute and return result
}
```

### Orchestrator Example

An Orchestrator is a MCS Driver itself, so it is transparent to the client.

```pseudo
class BasicOrchestrator extends MCSDriver {
    drivers: array[MCSToolDriver]

    constructor(drivers: array[MCSToolDriver], name?: string)

    private collect_tools() -> array[Tool]  // Aggregate from drivers

    get_function_description(model_name?: string) -> string  // Format tools

    get_driver_system_message(model_name?: string) -> string  // Build prompt

    process_llm_response(llm_response: string) -> any  // Parse, find tool, execute
}
```

For the client, it is irrelevant whether it interacts with a single / multiple MCS Driver(s) or Orchestrator(s), as both adhere to the same interface. This allows mixing and matching components arbitrarily without requiring adjustments to the client's logic.

### Mixins for Capabilities

```pseudo
abstract class SupportsHealthcheck {
    abstract healthcheck() -> dict  // e.g. {"status": "OK"}
}

abstract class SupportsAutostart {
    abstract autostart(kwargs: dict) -> void  // Launch container, etc.
}
```

Drivers extend with mixins as needed, e.g.:

```pseudo
class ConcreteDriver extends MCSDriver, SupportsHealthcheck {
    // Implement methods
}
```

Clients check `meta.capabilities` before invoking. Mixins should follow a convention like prefixing with "Supports" followed by the capability name and providing a method of the same name, but deviations are allowed for flexibility. The standard itself makes no prescriptions; SDKs should define these to standardize common capabilities.