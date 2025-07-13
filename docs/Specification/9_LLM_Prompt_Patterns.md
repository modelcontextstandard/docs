---
title: 9. LLM Prompt Patterns
sidebar_position: 9
---

# 9 · LLM Prompt Patterns (Informative)

While outside the formal spec, MCS emphasizes prompt patterns as a key driver benefit. They encapsulate optimized instructions within the driver, making prompt engineering a one-time, highly rewarding investment. Once refined (e.g., using DSPy for iterative optimization via input-output comparisons), a prompt becomes reusable across all compatible apps, models, and scenarios. Eliminating the need for scattered hubs or collections. Prompts evolve into specialized, model-agnostic assets, integrated exactly where needed and supporting multiple LLMs out of the box without client-side tweaks.

This clear separation of tool (execution) and prompt (communication) opens a strategic perspective on model optimization. A perfected prompt in the driver is instantly available to every compatible application and model, turning intensive prompt work into a sustainable, reusable building block for long-term use across projects. No more per-app reinvention. Specialized prompts handle diverse models seamlessly.

Inline-spec and schema-only are two common approaches (or "styles") for how drivers in MCS structure the prompts they provide to LLMs via `get_driver_system_message()`. These styles determine how the function description (from `get_function_description()`) is incorporated into the prompt to guide the LLM on tool usage. A ToolDriver will always follow a schema-only style, as it reduces the spec to a standardized Tools description (via `list_tools()`), focusing on lightweight aggregation for Orchestrators. In contrast, an MCSDriver can choose dynamically, e.g. inline-spec for powerful models (to provide full context upfront) or schema-only for weaker ones (to conserve tokens). The pros and cons balance comprehensiveness vs. efficiency, allowing adaptation per use case. 

1. Inline-spec Style

    What it means: The entire function description (e.g., a full OpenAPI spec or JSON schema) is directly embedded into the prompt as plain text, often formatted as a Markdown code block for readability. For example:
    ```
    You have access to the following tools:

    { "name": "get_weather", "description": "Get current weather", "parameters": { "city": { "type": "string" } } }
    Use this format to call: ```ts { "tool": "get_weather", "arguments": { "city": "Berlin" } }```
    ```
    Pros: Simple to implement. No extra logic needed. The LLM gets everything in one go, making it reliable for models that can handle the spec out of the box.
    Cons: Token-heavy, especially for large or complex specs (e.g., a big OpenAPI file could eat up thousands of tokens, limiting context for long conversations).
    When to use: For small, static toolsets where token cost isn't an issue.

2. Schema-only Style (optionalwith Fetch on Demand)

    What it means: The prompt includes only a condensed summary or schema of the arguments (e.g., just the required params and types), without the full description. The LLM could be instructed to "fetch" more details if needed. Meaning it generates a call to retrieve the full spec dynamically (e.g., via another tool or URL). Example prompt:
    ```
    You have access to tools. For details, fetch the schema first if unsure.

    Tool: get_weather
    Schema: { "city": { "type": "string", "required": true } }

    To fetch full description: Use { "tool": "fetch_spec", "arguments": { "tool_name": "get_weather" } }
    ```
    Pros: Saves tokens by keeping the initial prompt lightweight, ideal for large or dynamic toolsets.
    Cons: Requires a retrieval mechanism (e.g., the LLM must be able to call a "fetch" tool), which adds complexity and potential for errors if the model doesn't follow instructions.
    When to use: For expansive APIs or when token limits are tight, assuming the LLM supports multi-step reasoning.

Driver authors need not explicitly document their approach, as they are responsible for optimization. What matters to users is seamless performance, not the internals. In the driver's `__init__` method (or equivalent), allow external prompt overrides (or via a mixin for broader availability). This enables flexible updates without redeploying code. In the future, a dedicated Prompt Provider could allow dynamic loading of improved prompts, shared across clients and drivers, updating the behavior in drivers on the fly, but for now, focus on extensibility to see what patterns emerge.