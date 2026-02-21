---
title: 1. Minimal Driver Contract
sidebar_position: 1
---

# 3 · Minimal Driver Contract

See [MCS Driver Contract](Driver_Contract.md) for the detailed pseudocode.

The core MCSDriver interface is minimal:

- `meta`: Driver metadata (ID, version, protocol, transport, etc.)
- `get_function_description(model_name?)`: Returns LLM-readable function spec.
- `get_driver_system_message(model_name?)`: Returns full system prompt.
- `process_llm_response(llm_response) -> DriverResponse`: Parses and executes calls, returns a self-contained response object.

The driver itself is **stateless** -- all outcome information lives in the returned `DriverResponse`:

- `result`: The raw tool output (on success) or the unchanged input (on failure / no match).
- `call_executed`: Boolean -- true when a tool call was successfully executed.
- `call_failed`: Boolean -- true when a tool call signature was found but could not be parsed or executed.
- `call_detail`: Optional string -- human-readable reason when `call_failed` is true (for debugging).
- `retry_prompt`: Optional string -- driver-authored prompt hint the client can append for a retry.

If no call is detected, `result` contains the unchanged input for chaining.

*The exact signatures are up to each SDK. The semantics **must** match.*

### Why three methods instead of two?

Strictly, the MCS paradigm only requires two functions: a system prompt and a response processor. `get_function_description` is an intentional addition that forces driver developers to separate the function specification (data) from the system prompt (prompt logic). Without this separation, drivers would tightly couple spec and prompt, removing the AI application developer's ability to customize.

`get_driver_system_message` typically calls `get_function_description` internally and wraps the result in prompt guidance tailored for the target LLM. But the app developer can also call `get_function_description` directly and build an entirely different system prompt around it.

Together with the ToolDriver interface (see below), this creates a spectrum of control:

| Level | What you get | Control |
|---|---|---|
| `get_driver_system_message(model_name?)` | Ready-to-use system prompt | Convenience -- driver handles everything |
| `get_function_description(model_name?)` | Machine-readable spec string | Middle ground -- own prompt, driver-prepared spec |
| `MCSToolDriver.list_tools()` | Structured `Tool` objects | Maximum -- choose tools, format, prompt freely |

### `model_name` parameter

Both `get_function_description` and `get_driver_system_message` accept an optional `model_name`:

- Different LLMs process different formats with varying effectiveness. For example, some models handle XML structures better than JSON, while others are optimized for JSON function-calling syntax.
- `get_function_description(model_name)` allows the driver to render the same underlying spec in the most effective format for a given model (XML, JSON, simplified text, etc.).
- `get_driver_system_message(model_name)` allows model-specific prompt wording (format hints, few-shot examples, JSON schema constraints).
- For small models, model-specific optimization can meaningfully improve tool-call accuracy.
- When `model_name` is `null`, the driver returns a generic format that works across models. This is the common case.


### 3.1 `get_function_description(model_name?)`

Returns a static artifact that describes the available functions in a LLM-readable format.
The approach follows a standard-first principle. If an established specification format exists, it should be used.

Standard formats like:
* OpenAPI (JSON/YAML) – for RESTful APIs
* JSON Schema – for structured input/output validation, CLI tools, or message formats
* GraphQL SDL – for GraphQL-based APIs
* WSDL – for SOAP and legacy enterprise services
* gRPC / Protocol Buffers (proto) – for high-performance binary APIs
* OpenRPC – for JSON-RPC APIs
* EDIFACT/X12 schemas – for EDI-based B2B interfaces

If no standard is available, a custom function description has to be written.

Drivers may implement/use a dynamic descriptions to tailor the spec based on the LLM’s capabilities. 
For example, instead of exposing a raw OpenAPI schema, the driver may generate a simplified and LLM-friendly 
representation that retains full fidelity but improves comprehension.

Important is that the driver can accept a standard spec, how that is treated is up to the driver.


### 3.2 `get_driver_system_message(model_name?)`

Returns a complete system message containing or referencing the function description, 
crafted specifically for an LLM family.

This message guides the model to make valid and parseable tool calls. While the 
default behavior may simply inline get_function_description(), advanced drivers can 
define custom prompts tailored to different LLMs (e.g. OpenAI, Claude, Mistral), including:

* Format hints
* JSON schema constraints
* Few-shot examples
* Token budget control


### 3.3 `process_llm_response(llm_response) -> DriverResponse`

Consumes the output message generated by the LLM and returns a `DriverResponse` that carries both the result and the execution status. The method should:

1) Validate and parse the input (typically JSON or structured text)
2) If a call is detected but parsing fails: attempt self-healing patterns (e.g. fixing known model-specific formatting errors). If self-healing succeeds, continue with the corrected call.
3) If a call is detected and valid: map the request to a bridge-compatible operation (e.g. HTTP call, MQTT message), return `DriverResponse(result=<raw output>, call_executed=true)`.
4) If a call is detected but could not be executed (parsing failed even after self-healing): return `DriverResponse(result=llm_response, call_failed=true, call_detail=<reason>, retry_prompt=<hint>)`.
5) If no call is detected: return `DriverResponse(result=llm_response)` with both flags false.

Because all state lives in the response object, the driver itself remains stateless and thread-safe. Each call produces its own self-contained result.

This separation ensures that drivers focus on interfacing with the external system, while clients and orchestrators remain agnostic of internal logic and implementation.

