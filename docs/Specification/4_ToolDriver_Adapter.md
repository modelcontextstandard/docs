---
title: 4. ToolDriver & Adapter
sidebar_position: 4
---

# 4 · ToolDriver & Adapter

## The ToolDriver as the canonical interface unit

The smallest canonical interface unit in MCS is the `MCSToolDriver`. It defines the **source of truth** about which tools exist and how they are executed:

- `meta`: Driver metadata (ID, version, capability, etc.)
- `list_tools()`: Returns a list of `Tool` objects (name, title, description, parameters).
- `execute_tool(tool_name, arguments)`: Executes and returns the raw result.

Think of `list_tools()` as publishing an **API contract**: each `Tool` declares a method signature -- a name, a description of what it does, and the parameters it accepts. This is the interface that the LLM (indirectly, via the MCSDriver) and the Orchestrator work against. Everything downstream -- prompt generation, tool-call parsing, execution dispatch -- derives from this contract.

A ToolDriver is focused solely on technical bridging. It has no knowledge of LLMs, prompts, or model-specific quirks. ToolDriver authors can focus entirely on the technical bridge -- an HTTP call, a filesystem operation, a CAN-Bus message -- without needing any AI knowledge. The LLM-facing logic lives in the `MCSDriver` layer (see below).

Both `MCSDriver` and Orchestrator derive from the ToolDriver's tool definitions. The ToolDriver defines the interface; the MCSDriver makes it LLM-ready; the Orchestrator aggregates and curates it.

### 4.1 `list_tools()`

Returns a list of `Tool` objects representing the available functions. Each tool includes a name, an optional short title, a description, and a list of parameters (with name, description, required flag, and optional schema). At least one of `title` or `description` must be provided. The `title` serves as a token-efficient label for tool listings; the full `description` may contain multi-line prompt-engineering instructions and can be loaded on demand.

ToolDrivers may generate this list dynamically from a standard spec (e.g., fetching from an OpenAPI endpoint) or return a static set for capabilities where no formal spec exists (e.g., filesystem access, CSV processing).

### 4.2 `execute_tool(tool_name, arguments)`

Executes the specified tool with the given arguments and returns the raw result. The method should:

1. Validate the `tool_name` and `arguments` against the driver's capabilities.
2. Map the call to the underlying bridge operation (HTTP request, file operation, etc.).
3. Handle errors gracefully, raising exceptions for invalid calls.

## The Adapter pattern

### Why adapters exist

When building drivers for different capability+backend combinations (e.g. csv-localfs, csv-http, openapi-http, pdf-localfs, pdf-http), code duplication emerges quickly: the tool definitions, descriptions, and parsing logic are identical -- only the execution backend differs.

To eliminate this duplication, the ToolDriver delegates execution to a swappable **adapter**. The ToolDriver owns the tool definitions and knows which operations exist. The adapter implements those operations against a specific backend.

```
ToolDriver.execute_tool("write", {path, content})
  └── self.adapter.write(path, content)    // adapter interface (internal)
        ├── LocalFsAdapter.write()
        ├── HttpAdapter.write()
        └── S3Adapter.write()
```

The ToolDriver itself is responsible for defining tools -- whether statically (e.g. a fixed set of CSV operations) or dynamically (e.g. parsing an OpenAPI specification). This is not the adapter's concern. The adapter only provides the execution backend.

### Two interface levels

MCS has two distinct interface layers that must not be confused:

- **MCS interfaces** (`MCSDriver`, `MCSToolDriver`) are the external contract for clients and orchestrators. The LLM and the client see only this level.
- **Adapter interfaces** are internal technical contracts within a ToolDriver that make implementations swappable. Adapter interfaces are **not** part of the MCS standard; they are SDK and implementation concerns.

### Adapter consistency rule

If a ToolDriver exposes tools like `read`, `write`, `list`, and `delete`, then **every adapter** plugged into that ToolDriver must implement `read()`, `write()`, `list()`, and `delete()` -- the same internal interface, just against a different backend. The ToolDriver calls `self.adapter.write(...)` without knowing (or caring) whether the adapter talks to the local filesystem, an SMB share, or an HTTP endpoint.

This works well when the backends share similar semantics:

- A `filesystem` ToolDriver exposes hierarchical paths, directories, and shell-like operations. LocalFS, SMB, FTP, and SFTP all fit because they share this fundamental model.
- An `objectstore` ToolDriver exposes flat keys and prefixes. S3, GCS, and Azure Blob all fit.

When the semantic gap between two backends is large, forcing them into the same adapter interface creates a leaky abstraction. S3 *can* be exposed as a filesystem adapter, but it is emulation: `rename` becomes copy+delete, `mkdir` may be a no-op, listing may be eventually consistent. Such emulation is valid when the trade-off is acceptable, but the caveats must be clearly documented -- through tool descriptions, capability flags, or well-defined errors. When the gap is too large, a separate ToolDriver with its own tool API is the cleaner choice.

## From ToolDriver to Hybrid Driver

The recommended development workflow is bottom-up:

**Step 1 -- Define the Adapter interface.** Decide which operations the capability requires (`read()`, `write()`, `list()`, `delete()`, ...) and define them as an abstract interface. This is the contract that all backend implementations must fulfill, for the given ToolDriver.

**Step 2 -- Implement the Adapter.** Write a concrete implementation of that interface against your target backend (e.g. local filesystem, HTTP, S3). Each backend gets its own adapter class behind the same interface.

**Step 3 -- Write the ToolDriver.** For each method in the adapter interface, create a corresponding `Tool` in `list_tools()` -- with a name, description, and parameters. `execute_tool()` maps the incoming tool name to the matching adapter method and delegates. No LLM knowledge is required up to this point.

**Step 4 -- Write the Driver.** Create a class that implements both `MCSDriver` and `MCSToolDriver`. Internally, it delegates the ToolDriver methods (`list_tools`, `execute_tool`) to the ToolDriver from Step 2 and then adds the three LLM-facing methods (`get_function_description`, `get_driver_system_message`, `process_llm_response`). This is the **hybrid driver** -- the recommended default.

The MCSDriver is typically a **thin wrapper** around a ToolDriver: same tool semantics, just made LLM-ready through prompt formatting, output parsing, and optional self-healing. A driver *may* curate, rename, or merge tools, but the default case is 1:1 pass-through. 

Each step produces a reusable building block. If an adapter for your backend already exists, start at Step 3. If the adapter interface and ToolDriver already exist and you just need a new backend, write only the adapter (Step 2). A new capability on an existing transport (e.g. PDF over HTTP when CSV over HTTP already exists) means reusing the adapter and writing a new ToolDriver + Driver.

This layered approach produces a hybrid driver that a client can use directly via `get_driver_system_message()` + `process_llm_response()`, and that can also be composed into larger setups (see [Section 5](5_Orchestrator.md)).

## Why hybrid is the default

The hybrid pattern (implementing both `MCSDriver` and `MCSToolDriver`) is the recommended default because it maximizes composability:

- **Standalone**: A client calls the driver directly via the `MCSDriver` interface (`get_driver_system_message()` + `process_llm_response()`).
- **Stacked**: An Orchestrator calls `list_tools()` + `execute_tool()` on the same driver and handles the LLM response itself (see [Section 5](5_Orchestrator.md)).
- **Chained by client**: A client can hold multiple drivers and pass each LLM response through them sequentially. Because every driver implements `MCSDriver`, the client logic is identical regardless of how many drivers participate (but toolnameing conflicts may need to be resolved). 

This uniformity is the key design goal: the client never needs to know whether it talks to a single driver, a hybrid, or an orchestrator wrapping dozens of ToolDrivers. The interface is always `MCSDriver`.

### Unknown tool calls: pass-through, not failure

When a driver parses a valid JSON tool call but does **not** recognize the tool name, it must behave as if no tool call was detected at all -- return an empty `DriverResponse()` (no flags set), just like it would for a regular text response. From the outside, the LLM output passes through unchanged. This is critical for sequential client setups. The client passes the same LLM response to each driver in turn. If driver A does not recognize the tool, it acts as if nothing happened. The client then tries driver B, and so on. Only if *no* driver claims the call should the client treat it as unresolved or not a tool call emitted by the LLM.

An Orchestrator follows the same rule. From the client's perspective it is just another `MCSDriver` -- and it may itself be embedded in a higher-level Orchestrator. If the tool name is not found in any registered ToolDriver, the Orchestrator returns an empty `DriverResponse()`, just like any standalone driver would. The decision of what to do when *no* driver claims a call is always the client's responsibility.
