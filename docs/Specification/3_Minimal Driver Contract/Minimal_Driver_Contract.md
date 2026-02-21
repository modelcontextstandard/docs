---
title: 1. Minimal Driver Contract
sidebar_position: 1
---

# 3 · Minimal Driver Contract

See [MCS Driver Contract](Driver_Contract.md) for the detailed pseudocode.

The core MCSDriver interface is minimal:

- `meta`: Driver metadata (ID, version, protocol, transport, etc.)
- `get_function_description(model_name?)`: Returns function spec (raw or processed, depends on the driver).
- `get_driver_system_message(model_name?)`: Returns full system prompt. Ideally using `get_function_description` internally.
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

Strictly, the MCS paradigm only requires two functions: a system prompt and a response processor. `get_function_description` is an intentional addition that forces driver developers to separate the function specification (data) from the system prompt (transformation logic). Without this separation, drivers would tightly couple spec and prompt, removing the AI application developer's ability to customize.

`get_driver_system_message` typically calls `get_function_description` internally and wraps the result in prompt guidance tailored for the target LLM. But the app developer can also call `get_function_description` directly and build an entirely different system prompt around it.

Together with the ToolDriver interface (see below), this creates a spectrum of control:

| Level | What you get | Control |
|---|---|---|
| `get_driver_system_message(model_name?)` | Ready-to-use system prompt | Convenience -- driver handles everything |
| `get_function_description(model_name?)` | Spec string | Middle ground -- own prompt, driver-prepared spec |
| `MCSToolDriver.list_tools()` | Structured `Tool` objects | Maximum -- choose tools, format, prompt freely |

### `model_name` parameter

Both `get_function_description` and `get_driver_system_message` accept an optional `model_name`:

- Different LLMs process different formats with varying effectiveness. For example, some models handle XML structures better than JSON, while others are optimized for JSON function-calling syntax.
- `get_function_description(model_name)` allows the driver to render the same underlying spec in the most effective format for a given model (XML, JSON, simplified text, etc.).
- `get_driver_system_message(model_name)` allows model-specific prompt wording (format hints, few-shot examples, JSON schema constraints).
- For small models, model-specific optimization can meaningfully improve tool-call accuracy.
- When `model_name` is `null`, the driver returns a generic format that works across models. This will be the common case.


### 3.1 `get_function_description(model_name?)`

Returns a machine-readable description of the available functions. How the driver builds this description depends on the implementation:

**a) Custom extraction logic** -- The driver contains its own logic to read a source specification (e.g. OpenAPI, WSDL, GraphQL SDL) and transform it into an LLM-readable format. The driver developer decides what to extract, simplify, or reformat.

**b) Via ToolDriver** -- The driver delegates to an internal `MCSToolDriver` that provides structured `Tool` objects via `list_tools()`. The driver then serializes these objects into the appropriate format (JSON, XML, plain text, etc.) based on the target model. This is the recommended pattern for hybrid drivers.

**c) Raw pass-through** -- If the source specification is already LLM-readable (e.g. a well-structured OpenAPI JSON), the driver may pass it through with minimal or no transformation. Whether the *original* raw source should always be accessible is an open question (see [Section 12 -- Next Steps](../12_Next_Steps.md)).


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

1) Validate and parse the input (typically JSON or structured text).
2) If parsing fails: attempt self-healing (see [Section 9 -- Prompt Patterns](../9_LLM_Prompt_Patterns/LLM_Prompt_Patterns.md#self-healing)). If self-healing succeeds, continue with the corrected call.
3) If a call is detected and valid: execute the operation and return `DriverResponse(result=<raw output>, call_executed=true)`.
4) If a call is detected but could not be executed (even after self-healing): return `DriverResponse(result=llm_response, call_failed=true, call_detail=<reason>, retry_prompt=<hint>)`.
5) If no call is detected: return `DriverResponse(result=llm_response)` with both flags false.

Because all state lives in the response object, the driver itself remains stateless and thread-safe. Each call produces its own self-contained result. The driver holds no mutable state between calls, which eliminates any ambiguity about which call a status flag refers to.

**`retry_prompt`:** When `call_failed` is true, the `DriverResponse` includes a driver-authored prompt hint. The client can append this to the conversation so the LLM can correct its output and retry. This keeps prompt knowledge inside the driver -- the client never needs to understand why the call failed or how to fix it.

This separation ensures that drivers focus on interfacing with the external system, while clients and orchestrators remain agnostic of internal logic and implementation.
