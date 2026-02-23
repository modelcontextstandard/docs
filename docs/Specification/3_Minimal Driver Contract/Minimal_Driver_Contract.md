---
title: 3. Minimal Driver Contract
sidebar_position: 1
---

# 3 · Minimal Driver Contract

See [MCS Driver Contract](Driver_Contract.md) for the detailed pseudocode.

The core MCSDriver interface is minimal:

- `meta`: Driver metadata (ID, version, protocol, transport, etc.)
- `get_function_description(model_name?)`: Returns function spec (raw or processed, depends on the driver).
- `get_driver_system_message(model_name?)`: Returns full system prompt. Ideally using `get_function_description` internally.
- `process_llm_response(llm_response, streaming?) -> DriverResponse`: Parses and executes calls, returns a self-contained response object with pre-formatted messages.

The driver itself is **stateless** -- all outcome information lives in the returned `DriverResponse`:

- `tool_call_result`: The raw tool output. Only meaningful when `call_executed` is true; `null` otherwise.
- `call_executed`: Boolean -- true when a tool call was successfully executed.
- `call_failed`: Boolean -- true when a tool call signature was found but could not be parsed or executed.
- `call_detail`: Optional string -- reason when `call_failed` is true (for debugging).
- `retry_prompt`: Optional string -- driver-authored prompt hint the client can append for a retry.
- `messages`: Optional list of pre-formatted conversation entries (e.g. `{"role": "assistant", "content": ...}`, `{"role": "tool", "content": ...}`) that the client can append directly to its message history. `null` when no tool call was detected.

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

**b) Via ToolDriver** -- The driver delegates to an internal `MCSToolDriver` that provides structured `Tool` objects via `list_tools()`. The driver then serializes these objects into the appropriate format (JSON, XML, plain text, etc.) based on the target model. This is the recommended pattern.

**c) Raw pass-through** -- If the source specification is already LLM-readable (e.g. a well-structured OpenAPI JSON), the driver may pass it through with minimal or no transformation. Whether the *original* raw source should always be accessible is an open question (see [Section 14 -- Next Steps](../14_Next_Steps.md)).


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


### 3.3 `process_llm_response(llm_response, streaming?) -> DriverResponse`

Consumes the output generated by the LLM and returns a `DriverResponse` that carries the result, execution status, and pre-formatted messages for the client's conversation history.

**Input:** `llm_response` accepts either a raw text string or a structured dict (for LLMs that emit native tool-call objects rather than text). The optional `streaming` flag (default `false`) tells the driver it is processing incremental chunks; the driver may skip expensive operations on intermediate chunks and only apply them on the final call.

The method should:

1) Validate and parse the input (typically JSON or structured text, or a native tool-call dict).
2) If parsing fails: attempt optional self-healing (see [Section 10 -- Prompt Patterns](../10_LLM_Prompt_Patterns/LLM_Prompt_Patterns.md#self-healing)). If self-healing succeeds, continue with the corrected call.
3) If a call is detected and the tool belongs to this driver: execute the operation and return a `DriverResponse` with `tool_call_result=<raw output>`, `call_executed=true`, and `messages` containing the assistant message and tool result formatted for the conversation history.
4) If a call is detected but could not be executed (even after optional self-healing): return a `DriverResponse` with `call_failed=true`, `call_detail=<reason>`, `retry_prompt=<hint>`, and `messages` containing the assistant message and retry hint.
5) If a call is detected but the tool name is not recognized by this driver: return an empty `DriverResponse()` (no flags set, `messages=null`) -- as if no call was detected. This pass-through behavior is critical for sequential client setups and orchestrator stacking (see [Section 4](../4_ToolDriver_Adapter.md)).
6) If no call is detected: return an empty `DriverResponse()` with both flags false and `messages=null`. The LLM output is a final answer for the user.

Because all state lives in the response object, the driver itself remains stateless and thread-safe. Each call produces its own self-contained result. The driver holds no mutable state between calls, which eliminates any ambiguity about which call a status flag refers to.

**`messages`:** The driver is responsible for building correctly formatted conversation entries. On success, this typically includes the original LLM output as an assistant message and the tool result as a system/tool message. On failure, it includes the LLM output and the retry hint. The client simply extends its message history with `response.messages` -- no knowledge of the driver's internal format or prompt engineering required. This shifts message formatting responsibility from the client to the driver, where the domain knowledge resides.

**`retry_prompt`:** When `call_failed` is true, the `DriverResponse` includes a driver-authored prompt hint. The client can append this to the conversation so the LLM can correct its output and retry. This keeps prompt knowledge inside the driver -- the client never needs to understand why the call failed or how to fix it. When `messages` is provided, the retry prompt is already included in the messages list.

This separation ensures that drivers focus on interfacing with the external system, while clients and orchestrators remain agnostic of internal logic and implementation.
