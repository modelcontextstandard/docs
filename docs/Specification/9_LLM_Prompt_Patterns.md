---
title: 9. LLM Prompt Patterns / Dynamic Optimization
sidebar_position: 9
---

# 9 · LLM Prompt Patterns and Dynamic Optimization (Informative)

While outside the formal spec, MCS introduces a paradigm shift in how we think about LLM-system integration. The core insight: Since LLMs are programmed through natural language, and their reliability depends entirely on how well they follow prompts, the prompt itself becomes the primary optimization target.

## The Prompt as an Asset
MCS elevates prompts from hardcoded strings to reusable, optimizable assets. When prompt engineering efforts (e.g., using DSPy for iterative optimization) are encapsulated within drivers, they become instantly reusable across all applications. This transforms prompt engineering from a per-project burden into a one-time investment with compounding returns.

Prompt hubs have been around for a while, but they typically required manual integration. Developers had to find, adapt, and wire prompts into their own applications.
MCS likes to take this idea further by embedding prompt logic directly into the driver layer. Prompts can be automatically selected and optimized based on the active model, without any action required from the application developer.
This mechanism should be entirely optional. If no prompt provider is configured, the driver falls back to a standard built-in prompt. 

## Three Building Blocks for Flexible Prompt Layers
To support true model-agnostic flexibility, this idea separates the prompt logic into three dynamically loadable components:

- **Tool Description Templates**  
  Define how a tool is presented to the model—for example, as a JSON schema, a plain text block, or structured XML. Different models respond better to different representations.

- **Usage Instructions**  
  Explain when and how the model should use the tools. These include placeholders for the tool descriptions and specify the expected output format (e.g., JSON tool call).

- **Parsing Rules**  
  Define how to reliably extract tool calls from the model’s output, using regular expressions. Since LLM output is always plain text, even complex responses can be parsed using regular expressions alone. In essence, this makes the parser per Design it fully compatible with any LLM output.

This modular design allows the same driver function to be tailored dynamically to each model. These three components, description template, usage instruction, and parsing rule, form a complete prompt strategy for a specific model or model family.

They define how the model is instructed, how its output is interpreted, and how the driver bridges both ends.
This design enables flexible, model-specific prompt optimization without touching application or driver code.

Powerful models may receive a rich, inline specification with full details, while smaller or token-constrained models can be served a minimal schema version.  
The driver selects the optimal prompt configuration automatically—no changes in client code are needed.  
As a result, more models can be supported seamlessly with a single driver implementation.


## Dynamic Prompt Loading
Prompts do not have to be hardcoded in driver code. A driver can load model specific prompt parts from a registry during initialization or when the active model changes. This makes it possible to fix errors in existing prompts, support new models or optimize token usage without redeploy. Versioning with checksums is possible so clients can pin to tested prompt states and reproduce results.

A minimal flow.

```python
provider = PromptProvider.from_url("https://prompt-registry.example/v1")
driver = MCSDriver(spec_url=api_spec, prompt_provider=provider)
system_msg = driver.get_driver_system_message(model_name="gpt-4o-mini")
```

The driver loads template usage text and parser profile that match the model. The client receives only the final system prompt like before.

What a prompt-centric ecosystem enables
- Prompt marketplaces where optimised patterns for specific model and tool combinations are shared rated and improved by all
- Scalable A/B testing of prompt versions across thousands of real interactions
- Model specific tuning through automated tools such as DSPy
- Zero day support for brand new models by publishing a prompt package without touching driver or clientcode

## Token Management Strategies
As tool counts grow, token costs explode—a fundamental challenge when every tool adds to the system prompt. MCS proposes two mitigation strategies:

### 1. Inline-spec Style
Full tool descriptions embedded in the prompt. Simple but token-heavy.

**Example:**
You have access to the following tools:
```json
{ "name": "get_weather", "description": "Get current weather", "parameters": { "city": { "type": "string" } } }
```

### 2. **Schema-only Style** 
Minimal descriptions with on-demand fetching. Token-efficient but requires multi-step reasoning.

**Example:**
```json
Available tools: get_weather (fetches weather data)
For full specs, call: fetch_tool_spec(tool_name)
```

## Model-Specific Optimization

Different models excel with different prompt styles. GPT-4 might prefer JSON schemas, while Claude might perform better with XML-style descriptions. **The parsing rules can also vary**: some models might reliably use specific delimiters, while others need complex RegEx patterns to handle variations.

This isn't just optimization—it's acknowledging that **the success of any tool call depends entirely on the model's ability to follow the specific prompt format, not on the underlying transport mechanism**.

## Implementation Considerations

While fully dynamic prompt loading represents the ideal, initial implementations should:
1. Allow prompt overrides via constructor parameters or mixins
2. Support model-specific prompt variants within the driver
3. Document which models have been tested/optimized
4. Provide clear extension points for future prompt providers

## The Paradigm Shift

This approach represents a double revolution:
1. **First revolution (MCS core)**: Eliminate unnecessary protocols, use existing interfaces directly
2. **Second revolution (Prompt Patterns)**: Separate the LLM interaction layer entirely from the execution mechanism

When fully realized, this means:
- **Drivers become pure execution engines**, agnostic to LLM details
- **Prompts become tradeable assets**, optimized and shared like algorithms
- **Model differences become configuration**, not code changes
- **Reliability improvements happen at the prompt layer**, without touching infrastructure

The technology stack becomes completely autonomous from the LLMs themselves. A driver author can focus purely on bridging to external systems, while prompt engineers optimize the LLM interaction independently. When a new model appears, existing drivers immediately become compatible through new prompt configurations—no code changes required.

**This is the ultimate realization of MCS's core principle: The only thing that matters for successful LLM integration is how well the model follows prompts. Everything else is implementation detail.**


## Keep in Mind
Always allow external prompt overrides (via constructor parameters or a mixin) to enable:
- A/B testing of prompt variations
- Quick fixes for model-specific issues  
- Custom optimizations for specific use cases
- Gradual migration to dynamic prompt providers