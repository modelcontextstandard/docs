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

## Why hybrid is the default

The hybrid pattern (implementing both `MCSDriver` and `MCSToolDriver`) is the recommended default because it maximizes composability:

- **Standalone**: A client calls the driver directly via the `MCSDriver` interface (`get_driver_system_message()` + `process_llm_response()`).
- **Stacked**: An Orchestrator calls `list_tools()` + `execute_tool()` on the same driver and handles the LLM response itself.
- **Chained by client**: A client can hold multiple drivers and pass each LLM response through them sequentially. Because every driver implements `MCSDriver`, the client logic is identical regardless of how many drivers participate.

This uniformity is the key design goal: the client never needs to know whether it talks to a single driver, a hybrid, or an orchestrator wrapping dozens of ToolDrivers. The interface is always `MCSDriver`.

The same principle applies to Orchestrators. An Orchestrator that also implements `MCSToolDriver` exposes its aggregated tools via `list_tools()` and `execute_tool()`, making it a valid input for a higher-level Orchestrator. This allows encapsulating domain-specific concerns at one level (e.g. a "Finance Orchestrator" that bundles market data + reports + trading) and composing them at a higher level (e.g. an "Enterprise Orchestrator" that combines Finance, HR, and IT). How practical deep nesting turns out to be remains to be seen -- but the architecture does not prevent it, and even two levels of composition cover the majority of real-world scenarios.

### Unknown tool calls: pass-through, not failure

When a standalone driver parses a valid JSON tool call but does **not** recognize the tool name, it must behave as if no tool call was detected at all -- return an empty `DriverResponse()` (no flags set), just like it would for a regular text response. From the outside, the LLM output passes through unchanged. This is critical for sequential client setups. The client passes the same LLM response to each driver in turn. If driver A does not recognize the tool, it acts as if nothing happened. The client then tries driver B, and so on. Only if *no* driver claims the call should the client treat it as unresolved or not a tool call emitted by the LLM.

An Orchestrator follows the same rule. From the client's perspective it is just another `MCSDriver` -- and it may itself be embedded in a higher-level Orchestrator. If the tool name is not found in any registered ToolDriver, the Orchestrator returns an empty `DriverResponse()`, just like any standalone driver would. The decision of what to do when *no* driver claims a call is always the client's responsibility.

## When you need an Orchestrator

A hybrid driver handles one binding (one protocol/transport pair). When an application needs **multiple bindings** or **multiple connections** of the same binding, an Orchestrator coordinates them.

Typical scenarios:

| Scenario | What aggregates | Example |
|---|---|---|
| Multiple connections, same binding | Several instances of the same ToolDriver with different endpoints | Three REST APIs (CRM, ERP, payment gateway) |
| Multiple bindings | Different ToolDrivers for different protocol/transport pairs | REST API + local filesystem + message bus |
| Domain orchestration | A curated set of tools for a specific domain | Finance: REST (market data) + filesystem (reports) + FIX (trading) |

The Orchestrator implements `MCSDriver`. It collects tools from all registered ToolDrivers, builds a unified prompt, and dispatches tool calls to the correct driver. For the AI client, it appears as a single driver. To be composable itself, the Orchestrator should also implement `MCSToolDriver` and expose the aggregated tool list via `list_tools()`.

### Orchestrator responsibilities

1. **Tool aggregation**: Collects tools from all ToolDrivers via `list_tools()`.
2. **Unified prompt**: Builds a single `get_driver_system_message()` that presents all tools in a consistent format. Every ToolDriver contributes its tools, but the Orchestrator owns the final prompt -- it decides naming, ordering, and description style.
3. **Dispatch**: Parses the LLM response, identifies which tool was called, routes to the correct ToolDriver via `execute_tool()`.
4. **Conflict resolution**: When multiple ToolDrivers expose tools with the same name, the Orchestrator must resolve the collision. Strategies include namespacing (e.g. `crm_list_contacts` vs `erp_list_contacts`), priority ordering, or rejecting the conflict at registration time. The Orchestrator adjusts the tool descriptions in the prompt accordingly so the LLM uses the namespaced names, and maps them back to the original names before dispatching to `execute_tool()`.

### Orchestrators are composable

Like we said, when an Orchestrator implements `MCSToolDriver`, it can itself be wrapped by another Orchestrator. This allows layered compositions: a domain-specific Orchestrator aggregating protocol-specific Orchestrators, each managing multiple connections.

```
AI Client
  └── Finance Orchestrator (MCSDriver)
        ├── REST Orchestrator (MCSDriver)
        │     ├── Market-Data ToolDriver (REST/HTTP)
        │     └── CRM ToolDriver (REST/HTTP)
        ├── Filesystem ToolDriver (FS/local)
        └── FIX ToolDriver (FIX/TCP)
```

### Orchestrators as ToolDrivers (recursive stacking)

An Orchestrator that also implements `MCSToolDriver` becomes a building block for higher-level Orchestrators. This enables recursive stacking: a domain Orchestrator aggregates protocol-level Orchestrators, each of which aggregates individual ToolDrivers.

This works because the contract is uniform at every level:
- The **client** needs `MCSDriver` -- it gets it from the top-level Orchestrator.
- Each **Orchestrator** needs `MCSToolDriver` inputs -- it gets them from hybrid drivers *or* from other Orchestrators that also implement `MCSToolDriver`.
- Each **hybrid driver** implements both interfaces, so it can participate at any level.

This is why hybrid is the practical default: a driver that implements both interfaces can be used standalone, stacked via an Orchestrator, or even used as a sub-Orchestrator -- without any code changes.

If an Orchestrator only implements `MCSDriver` (not `MCSToolDriver`), it serves as a terminal entry point for the client and cannot be embedded further. Both patterns are valid; hybrid is simply more flexible.

## Summary

| Component | Implements | LLM knowledge | Typical use |
|---|---|---|---|
| **ToolDriver** | `MCSToolDriver` | None | Pure bridge; used via Driver or Orchestrator |
| **Driver (hybrid)** | `MCSDriver` + `MCSToolDriver` | Yes | Standalone use or via Orchestrator; wraps one ToolDriver 1:1 |
| **Orchestrator** | `MCSDriver` (optionally `MCSToolDriver`) | Yes | Aggregates multiple ToolDrivers across bindings or connections |
