---
title: 2. Core Idea
sidebar_position: 2
---


# 2 · Core Idea

**MCS** defines a *thin glue layer* between a Language Model and any existing interface such as REST, GraphQL, CAN-Bus, EDI, or even filesystems.

Every MCS driver must implement **two mandatory capabilities**:

1. **Expose**: Provide a machine-readable *function description* via `get_function_description()` and usage instructions (including function description) via `get_driver_system_message()`. This allows the LLM to discover available tools and learn how to call them effectively. While `get_function_description()` is optional for LLMs that natively handle specs like OpenAPI (e.g., ChatGPT), `get_driver_system_message()` is preferred. It delivers a complete, pre-optimized system prompt tailored by the driver author, freeing clients from prompt engineering and ensuring high-quality, reusable prompts that evolve over time.
2. **Execute**: Handle structured LLM-emitted calls via `process_llm_response()`. The driver parses the request, routes it through a bridge (HTTP, serial, message bus, etc.), and returns the raw result.

The complexity of a **MCS Driver** is mostly concentrated in the execution phase. Everything related to authentication, rate-limiting, retries, logging, or protocol-specific quirks is handled internally by the driver using existing transports and if available machine readable standard specs like OpenAPI.

Drivers are initialized with configuration parameters through the constructor. This makes it easy to inject dependencies or load configuration dynamically at runtime.

Optional or advanced functionality can be added modularly via *capabilities*, allowing drivers to remain lightweight by default.

The **client** acts as a coordinator. It retrieves the function specification from the driver, injects it into the LLM system message, and later passes the LLM’s output back to the driver for inspection, if an execution of a function is wanted by the LLM. 

Importantly, the client does not need to know how the driver works internally, which technology stack it uses or what prompts should be used.

To handle multiple drivers efficiently and avoid format and tool name conflicts, MCS introduces the concept of an **Orchestrator**. The Orchestrator aggregates tools from multiple **ToolDrivers** (a specialized driver type that lists tools and executes them without direct LLM interaction), unifies their descriptions into a consistent format, and presents them as a single MCSDriver to the client.

The Orchestrator is a MCS Driver itself, so it is transparent to the client.

For the client, it is irrelevant whether it interacts with a single / multiple MCS Driver(s) or Orchestrator(s), as both adhere to the same interface. This allows mixing and matching components arbitrarily without requiring adjustments to the client's logic.

This separation allows ToolDrivers to focus on technical bridging, while Orchestrators handle LLM-specific optimizations like prompt formatting also for a variety of LLMs.


---

### Phase A – Spec exposure

```
 Client ─── request spec ───▶  Driver                
  ▲                              │
  └─── Spec (OpenAPI …) ◄────────┘
```
The client first calls `get_function_description()` to retrieve a machine-readable function specification. How this spec is generated or retrieved—whether from a local file, HTTP endpoint, or generated dynamically, is left to the driver implementation.

The client may embed the spec into the LLM's system prompt or use it in other prompt injection strategies.

To simplify this process, drivers need to implement `get_driver_system_message()` which returns a complete, ready-to-use system prompt. This includes both the tool description and formatting guidance tailored for a LLM or for a specific LLM as well.

This is crucial because different LLMs may respond better to differently phrased prompts.


### Phase B – Call execution

```
LLM ──► JSON call ──► Driver/Parser ──► External API
 ▲                           │
 └─────────── Result ◄───────┘
```

Once the LLM emits a structured function call (typically as a JSON object in the text output), the client passes this to the driver’s `process_llm_response()` method.

The driver parses the call, dispatches it over its bridge (e.g. HTTP, CAN-Bus, AS2), and returns the raw result. The result can then be forwarded back into the conversation, either directly or via formatting logic handled elsewhere.



### Phase C – Conversation loop

In practice, a single user request may require multiple tool calls before the LLM can produce a final answer. The client therefore runs an iterative loop:

```
                    ┌───────────────────────────────────────────┐
                    │                                             ▼
User ──► Client ──► LLM ──► process_llm_response ──► call_executed?
                     ▲              │                       │
                     │          result               call_failed?
                     │              │                  │         │
                     └──────────────┘           retry     no match
                                                       │         │
                                              get_retry_prompt  Final answer
                                                       │
                     ┌─────────────────────────────┘
                     │
                     ▼
```

1. The client sends the conversation (system prompt + message history) to the LLM.
2. The LLM responds. The client passes the response to `process_llm_response()`.
3. If `call_executed` is true: the client appends the LLM message and the tool result to the conversation history and returns to step 1.
4. If `call_failed` is true: the driver found a tool-call signature but could not parse or execute it. The client appends `get_retry_prompt()` to the conversation so the LLM can correct its output, then returns to step 1.
5. If neither flag is set: no tool call was detected. The LLM’s response is the final answer for the user.

The per-call state (`call_executed`, `call_failed`, `last_call_detail`) is reset at the start of every `process_llm_response()` invocation. The driver does not track conversation history – that remains the client’s responsibility. This keeps the driver contract simple while allowing arbitrarily complex multi-step interactions.

A Proof of Concept with existing ChatModels can be found [here](https://github.com/modelcontextstandard#getting-started-experience-the-wow-moment-in-2-minutes).