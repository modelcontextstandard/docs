---
title: 4. Orchestrator & ToolDriver
sidebar_position: 4
---

# 4 · Orchestrator & ToolDriver

## The ToolDriver interface

A ToolDriver is a specialized driver type focused solely on technical bridging. It has no knowledge of LLMs, prompts, or model-specific quirks. Its job is to expose tools and execute them:

- `meta`: Driver metadata (ID, version, protocol, transport, etc.)
- `list_tools()`: Returns a list of `Tool` objects (name, description, parameters).
- `execute_tool(tool_name, arguments)`: Executes and returns the raw result.

*The exact signatures are up to each SDK. The semantics **must** match.*

This separation exists so that ToolDriver authors can focus entirely on the technical bridge -- an HTTP call, a filesystem operation, a CAN-Bus message -- without needing any AI knowledge. The LLM-facing logic lives elsewhere.

### 4.1 `list_tools()`

Returns a list of `Tool` objects representing the available functions. Each tool includes a name, description, and a list of parameters (with name, description, required flag, and optional schema).

ToolDrivers may generate this list dynamically from a standard spec (e.g., fetching from an OpenAPI endpoint) or return a static set for protocols where no formal spec exists (e.g., filesystem access).

### 4.2 `execute_tool(tool_name, arguments)`

Executes the specified tool with the given arguments and returns the raw result. The method should:

1. Validate the `tool_name` and `arguments` against the driver's capabilities.
2. Map the call to the underlying bridge operation (HTTP request, file operation, etc.).
3. Handle errors gracefully, raising exceptions for invalid calls.

## From ToolDriver to Hybrid Driver

The recommended development workflow is bottom-up:

**Step 1 -- Write the ToolDriver.** Implement `list_tools()` and `execute_tool()` for your protocol/transport pair. This is pure technical work: read a spec, map calls, handle errors. No LLM knowledge required.

**Step 2 -- Write the Driver.** Create a class that implements both `MCSDriver` and `MCSToolDriver`. Internally, it delegates the ToolDriver methods (`list_tools`, `execute_tool`) to the ToolDriver from Step 1 and adds the three LLM-facing methods (`get_function_description`, `get_driver_system_message`, `process_llm_response`). This is the **hybrid driver** -- the recommended default. Conceptually, it is a 1:1 orchestrator: one driver wrapping one ToolDriver.

This two-step approach produces a driver that works in both modes (Hybrid):

- **Standalone**: An AI client calls `get_driver_system_message()` + `process_llm_response()` directly.
- **Via Orchestrator**: An orchestrator calls `list_tools()` + `execute_tool()` and handles the LLM interaction itself.

## When you need an Orchestrator

A hybrid driver handles one binding (one protocol/transport pair). When an application needs **multiple bindings** or **multiple connections** of the same binding, an Orchestrator coordinates them.

Typical scenarios:

| Scenario | What aggregates | Example |
|---|---|---|
| Multiple connections, same binding | Several instances of the same ToolDriver with different endpoints | Three REST APIs (CRM, ERP, payment gateway) |
| Multiple bindings | Different ToolDrivers for different protocol/transport pairs | REST API + local filesystem + message bus |
| Domain orchestration | A curated set of tools for a specific domain | Finance: REST (market data) + filesystem (reports) + FIX (trading) |

The Orchestrator implements `MCSDriver`. It collects tools from all registered ToolDrivers, builds a unified prompt, and dispatches tool calls to the correct driver. For the AI client, it appears as a single driver.

### Orchestrator responsibilities

1. **Tool aggregation**: Collects tools from all ToolDrivers via `list_tools()`.
2. **Unified prompt**: Builds a single `get_driver_system_message()` that presents all tools in a consistent format.
3. **Dispatch**: Parses the LLM response, identifies which tool was called, routes to the correct ToolDriver via `execute_tool()`.
4. **Conflict resolution**: Handles name collisions when multiple ToolDrivers expose tools with the same name.

### Orchestrators are composable

Because an Orchestrator implements `MCSDriver`, it can itself be wrapped by another Orchestrator. This allows layered compositions: a domain-specific Orchestrator aggregating protocol-specific Orchestrators, each managing multiple connections.

```
AI Client
  └── Finance Orchestrator (MCSDriver)
        ├── REST Orchestrator (MCSDriver)
        │     ├── Market-Data ToolDriver (REST/HTTP)
        │     └── CRM ToolDriver (REST/HTTP)
        ├── Filesystem ToolDriver (FS/local)
        └── FIX ToolDriver (FIX/TCP)
```

### Should an Orchestrator also implement MCSToolDriver?

This is a design decision left to the implementation. If the Orchestrator implements `MCSToolDriver`, it can be embedded into a higher-level Orchestrator. If it only implements `MCSDriver`, it serves as a top-level entry point for the client. Both patterns are valid.

## Summary

| Component | Implements | LLM knowledge | Typical use |
|---|---|---|---|
| **ToolDriver** | `MCSToolDriver` | None | Pure bridge; used via Driver or Orchestrator |
| **Driver (hybrid)** | `MCSDriver` + `MCSToolDriver` | Yes | Standalone use or via Orchestrator; wraps one ToolDriver 1:1 |
| **Orchestrator** | `MCSDriver` (optionally `MCSToolDriver`) | Yes | Aggregates multiple ToolDrivers across bindings or connections |
