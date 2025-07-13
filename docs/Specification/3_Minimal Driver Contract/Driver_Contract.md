---
title: MCS Driver Contract
sidebar_position: 2
---

# MCS Driver Contract – Version 0.2

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

    abstract get_function_description(model_name?: string) -> string  // Machine-readable spec

    abstract get_driver_system_message(model_name?: string) -> string  // Full system prompt

    abstract process_llm_response(llm_response: string) -> any  // Execute if call detected, else return input
}
```

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