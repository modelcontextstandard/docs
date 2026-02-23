---
title: 4. ToolDriver & Adapter
sidebar_position: 4
---

# 4 · ToolDriver & Adapter

## The ToolDriver as the canonical interface unit

The smallest canonical interface unit in MCS is the `MCSToolDriver`. It defines the **source of truth** about which tools exist and how they are executed:

- `meta`: Driver metadata (ID, version, capability, etc.)
- `list_tools()`: Returns a list of `Tool` objects (name, description, parameters).
- `execute_tool(tool_name, arguments)`: Executes and returns the raw result.

*The exact signatures are up to each SDK. The semantics **must** match.*

A ToolDriver is focused solely on technical bridging. It has no knowledge of LLMs, prompts, or model-specific quirks. ToolDriver authors can focus entirely on the technical bridge -- an HTTP call, a filesystem operation, a CAN-Bus message -- without needing any AI knowledge. The LLM-facing logic lives in the `MCSDriver` layer (see below).

Both `MCSDriver` and Orchestrator derive from the ToolDriver's tool definitions. The ToolDriver defines the interface; the MCSDriver makes it LLM-ready; the Orchestrator aggregates and curates it.

### 4.1 `list_tools()`

Returns a list of `Tool` objects representing the available functions. Each tool includes a name, description, and a list of parameters (with name, description, required flag, and optional schema).

ToolDrivers may generate this list dynamically from a standard spec (e.g., fetching from an OpenAPI endpoint) or return a static set for capabilities where no formal spec exists (e.g., filesystem access, CSV processing).

### 4.2 `execute_tool(tool_name, arguments)`

Executes the specified tool with the given arguments and returns the raw result. The method should:

1. Validate the `tool_name` and `arguments` against the driver's capabilities.
2. Map the call to the underlying bridge operation (HTTP request, file operation, etc.).
3. Handle errors gracefully, raising exceptions for invalid calls.

## The Adapter pattern

### Why adapters exist

When building drivers for different capability+transport combinations (e.g. csv-localfs, csv-http, pdf-localfs, pdf-http), code duplication emerges quickly:

- Identical tool signatures and function descriptions across transports
- Identical parsing logic for specifications
- Identical connection logic (timeouts, proxies, retries, headers, ...) across capabilities

To eliminate this duplication, the ToolDriver delegates to two swappable building blocks:

### SpecAdapter and TransportConnector

1. **SpecAdapter** (format/semantics) -- converts a specification or domain description into standardized `Tool` objects. It knows nothing about runtime transport details.
2. **TransportConnector** (I/O) -- encapsulates connection and execution logic (HTTP, local filesystem, S3, ...). It knows nothing about tool semantics.

The ToolDriver composes both:

```
ToolDriver.execute_tool("write", {path, content})
  └── self.transport.write(path, content)    // adapter interface (internal)
        ├── LocalFsConnector.write()
        ├── HttpConnector.write()
        └── S3Connector.write()
```

**Spec source and execution transport are independent dimensions.** A spec sitting on the local filesystem does not imply that calls execute locally. Equally, a spec loaded via HTTP may trigger execution through an entirely different connector.

### Two interface levels

MCS has two distinct interface layers that must not be confused:

- **MCS interfaces** (`MCSDriver`, `MCSToolDriver`) are the external contract for clients and orchestrators. The LLM and the client see only this level.
- **Adapter interfaces** are internal technical contracts within a ToolDriver that make implementations swappable. They consist of methods like `read()`, `write()`, `list()`, `delete()` -- purely technical, no LLM awareness. Adapter interfaces are **not** part of the MCS standard; they are SDK and implementation concerns.

### Adapter consistency rule

All adapters under a given ToolDriver must fulfill the same interface. The ToolDriver defines the tool API; different adapters provide different implementations of the same operations.

When the semantic gap between two backends is small, they can share a ToolDriver and adapter interface:

- A `filesystem` ToolDriver exposes hierarchical paths, directories, and shell-like operations. It fits localfs, SMB, FTP, and SFTP because the fundamental semantics are similar.
- An `objectstore` ToolDriver exposes keys and prefixes. It fits S3, GCS, and Azure Blob.

When the semantic gap is large, a separate ToolDriver with its own tool API is the cleaner choice. S3 *can* be exposed as a filesystem adapter, but this is emulation with explicit caveats: `rename` becomes copy+delete, `mkdir` may be a no-op, listing may be eventually consistent. Such emulation is valid but must be clearly documented -- through tool descriptions, capability flags, or well-defined errors.

## From ToolDriver to Hybrid Driver

The recommended development workflow is bottom-up:

**Step 1 -- Write the ToolDriver.** Implement `list_tools()` and `execute_tool()` for your capability. This is pure technical work: read a spec, map calls, handle errors. No LLM knowledge required.

**Step 2 -- Write the Driver.** Create a class that implements both `MCSDriver` and `MCSToolDriver`. Internally, it delegates the ToolDriver methods (`list_tools`, `execute_tool`) to the ToolDriver from Step 1 and adds the three LLM-facing methods (`get_function_description`, `get_driver_system_message`, `process_llm_response`). This is the **hybrid driver** -- the recommended default.

The MCSDriver is typically a **thin wrapper** around a ToolDriver: same tool semantics, just made LLM-ready through prompt formatting, output parsing, and self-healing. A driver *may* curate, rename, or merge tools, but the default case is 1:1 pass-through. Conceptually, it is a single-driver orchestrator: one driver wrapping one ToolDriver.

This two-step approach produces a driver that works in both modes:

- **Standalone**: An AI client calls `get_driver_system_message()` + `process_llm_response()` directly.
- **Via Orchestrator**: An orchestrator calls `list_tools()` + `execute_tool()` and handles the LLM interaction itself.

## Why hybrid is the default

The hybrid pattern (implementing both `MCSDriver` and `MCSToolDriver`) is the recommended default because it maximizes composability:

- **Standalone**: A client calls the driver directly via the `MCSDriver` interface (`get_driver_system_message()` + `process_llm_response()`).
- **Stacked**: An Orchestrator calls `list_tools()` + `execute_tool()` on the same driver and handles the LLM response itself.
- **Chained by client**: A client can hold multiple drivers and pass each LLM response through them sequentially. Because every driver implements `MCSDriver`, the client logic is identical regardless of how many drivers participate.

This uniformity is the key design goal: the client never needs to know whether it talks to a single driver, a hybrid, or an orchestrator wrapping dozens of ToolDrivers. The interface is always `MCSDriver`.

### Unknown tool calls: pass-through, not failure

When a standalone driver parses a valid JSON tool call but does **not** recognize the tool name, it must behave as if no tool call was detected at all -- return an empty `DriverResponse()` (no flags set), just like it would for a regular text response. From the outside, the LLM output passes through unchanged. This is critical for sequential client setups. The client passes the same LLM response to each driver in turn. If driver A does not recognize the tool, it acts as if nothing happened. The client then tries driver B, and so on. Only if *no* driver claims the call should the client treat it as unresolved or not a tool call emitted by the LLM.

An Orchestrator follows the same rule. From the client's perspective it is just another `MCSDriver` -- and it may itself be embedded in a higher-level Orchestrator. If the tool name is not found in any registered ToolDriver, the Orchestrator returns an empty `DriverResponse()`, just like any standalone driver would. The decision of what to do when *no* driver claims a call is always the client's responsibility.

## Summary

| Component | Implements | LLM knowledge | Typical use |
|---|---|---|---|
| **ToolDriver** | `MCSToolDriver` | None | Pure bridge; used via Driver or Orchestrator |
| **Driver (hybrid)** | `MCSDriver` + `MCSToolDriver` | Yes | Standalone use or via Orchestrator; wraps one ToolDriver 1:1 |
| **Orchestrator** | `MCSDriver` (optionally `MCSToolDriver`) | Yes | Aggregates multiple ToolDrivers; see [Section 5](5_Orchestrator.md) |
| **Adapter** | Internal (not MCS standard) | None | Swappable transport/spec implementation inside a ToolDriver |
