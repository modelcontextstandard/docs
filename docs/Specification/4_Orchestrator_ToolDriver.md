---
title: 4. Orchestrator & ToolDriver
sidebar_position: 4
---

# 4 · Orchestrator & ToolDriver

When using multiple MCS drivers in a chain, each driver must autonomously determine if a function call in the LLM response is intended for it, assuming drivers maintain an internal representation of their tools (implemented in varying ways without a strict convention). The LLM's output is passed sequentially through all `process_llm_response` methods, and the system prompts from all drivers are combined and fed to the LLM. This can lead to side effects, especially with numerous drivers offering different formats. The LLM may struggle to cleanly separate and handle them, resulting in parsing errors or unreliable calls.

To achieve higher integration and mitigate these issues, MCS recommends a middleware like an Orchestrator. It serves as a central coordinator for multiple drivers, offering several advantages when dealing with diverse or numerous tools. For instance, it prevents function collisions (e.g., when multiple drivers provide tools with the same name) by unifying and renaming them if needed. Instead of blindly routing the LLM response through every driver, the Orchestrator intelligently dispatches to the specific tool the model intended. A key benefit is tool unification. It converts varied formats and descriptions from individual drivers into a consistent target format, possibly optimized for particular LLMs, reducing context bloat and improving reliability.

The Orchestrator's usage depends on the application scenario. In simple setups with one or two drivers, direct chaining suffices. In complex environments with many drivers, an Orchestrator can manage all tasks or target specific areas. Hybrid forms are also possible, allowing flexible scaling without client changes.

To enable this, MCS introduces the ToolDriver—a specialized driver type focused solely on technical bridging, without direct LLM interaction. This separation motivates the ToolDriver: It allows authors to concentrate purely on execution and bridging logic, without needing knowledge of LLMs, prompts, or model-specific quirks—shifting that responsibility to the Orchestrator for better modularity and expertise division.

The core MCSToolDriver interface is minimal:

- `meta`: Driver metadata (ID, version, protocol, transport, etc.)
- `list_tools()`: Returns list of Tool objects (name, description, parameters).
- `execute_tool(tool_name, arguments)`: Executes and returns result.

*The exact signatures are up to each SDK. The semantics **must** match.*

### 4.1 `list_tools()`

Returns a list of Tool objects representing the functions or tools provided by the driver. Each Tool includes a name, description, and a list of parameters (with name, description, required flag, and optional schema).

This method allows the Orchestrator to discover and aggregate tools from multiple ToolDrivers without needing to know their internal details. The returned tools should be in a standardized format to facilitate unification.

ToolDrivers may dynamically generate this list based on a standard spec (e.g., fetching from an OpenAPI endpoint) or return a static set for unstandardized tools, like filesystems over local file system, where no real spec is available.

### 4.2 `execute_tool(tool_name, arguments)`

Executes the specified tool with the given arguments and returns the raw result. The method should:

1) Validate the tool_name and arguments against the driver's internal capabilities.
2) Map the call to the underlying bridge operation (e.g., HTTP request, file operation).
3) Handle errors gracefully, potentially raising exceptions for invalid calls.

This keeps execution isolated and focused on the technical bridge, allowing Orchestrators to handle LLM-specific parsing and routing.

BasicOrchestrator implements MCSDriver by collecting from ToolDrivers, formatting prompts, and dispatching calls.

This separation keeps ToolDrivers lean and execution-focused (e.g., handling HTTP calls or file access), while Orchestrators (which implement the MCSDriver interface) aggregate these tools, build unified prompts, and route executions efficiently. For the client, an Orchestrator appears as a single MCSDriver, enabling seamless mixing without logic adjustments.

#### Hybrid Tools may also be possible to be used with or without an Orchestrator, implementing both interfaces.
It is now an easier task because the driver already has an internal representation of the tools with this approach.