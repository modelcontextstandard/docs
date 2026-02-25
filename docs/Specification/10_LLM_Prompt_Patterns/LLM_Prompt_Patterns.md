---
title: 10. LLM Prompt Patterns / Dynamic Optimization
sidebar_position: 10
---

# 10 · LLM Prompt Patterns and Dynamic Optimization (Perspective)

**Note**: This section describes experimental concepts and future possibilities that extend beyond the current MCS implementation.

While outside the formal spec, MCS introduces a paradigm shift in how we think about LLM-system integration. The core insight: Since LLMs are programmed through natural language, and their reliability depends entirely on how well they follow prompts, the prompt itself becomes the primary optimization target like a firmware.

## The Prompt as an Asset (Firmeware)
MCS elevates prompts from hardcoded strings to reusable, optimizable assets. When prompt engineering efforts (e.g., using DSPy for iterative optimization) are encapsulated within drivers, they become instantly reusable across all applications. This transforms prompt engineering from a per-project burden into a one-time investment with compounding returns.

The traditional approach to prompt engineering has been fragmented and inefficient. Prompt hubs have existed for a while, offering collections of proven prompts, but they typically required manual integration. Developers had to find suitable prompts, adapt them to their specific needs, and wire them into their applications. A process repeated for every project and every model update.

MCS takes this concept further by embedding prompt logic directly into the driver layer. Prompts can be automatically selected and optimized based on the active model, without any action required from the application developer. This mechanism is entirely optional, if no prompt provider is configured, the driver falls back to a standard built-in prompt, ensuring backward compatibility and ease of adoption.


## The Reality of Tool-Calling Today
The current state of tool calling across LLMs reveals a fragmented landscape where reliability varies dramatically between models. Recent industry analysis shows sobering realities:

- **Grok 4** frequently hallucinates tool syntax from its training data, outputting tool calls as plain text instead of using the provided format
- **Gemini 2.5 Pro** has a peculiar habit of announcing which tool it's about to use, then failing to actually execute the call
- **GPT models** historically lagged behind in reliability, only recently catching up through focused improvements
- **Claude models** maintain a near-monopoly on reliable tool calling, not through superior intelligence but through better prompt adherence

The mathematics of compound failure rates makes this particularly brutal. If a model achieves 98% accuracy on individual tool calls, but your application makes 5 calls per request, your overall reliability drops to 90%. A seemingly small decrease to 96% accuracy results in 80% reliability—effectively doubling your failure rate. This is why even marginal improvements in prompt engineering can have massive real-world impact.


## Breaking the Monopoly Through Optimization
Currently, if you need reliable tool calling for production applications, you're essentially forced to use Anthropic's models and pay their premium prices. But this monopoly exists not because of fundamental model superiority, but because of superior prompt handling and execution.

MCS's prompt-provider approach, combined with optimization frameworks like DSPy, could democratize tool-calling excellence. Imagine a future where:

- **Automated optimization tools** systematically test and refine prompts for each model through thousands of iterations
- **Small open-source models** (7B-13B parameters) achieve 95%+ tool-calling reliability with properly optimized prompts
- **Cost structures** shift dramatically, why pay Anthropic's premium when a locally-run Llama or Mistral model performs just as reliably?
- **Synthetic data generation** from reliable models like Kimi K2 enables training and distillation of tool-calling capabilities into smaller, faster models

This isn't theoretical. The release of Kimi K2 demonstrates that matching Anthropic's tool-calling prowess is possible. With the right prompts and training approach, any model may excel at tool calling. At least be improved.


## Three Building Blocks for Flexible Prompt Layers
To achieve true model-agnostic flexibility, MCS separates prompt logic into three dynamically loadable components that work together to form a complete interaction strategy.

### 1. Tool Description Templates
These define how tools are presented to the model. Different models respond better to different representations, some prefer JSON schemas, others work better with XML structures, and some excel with plain text descriptions. The template system allows the same tool to be presented optimally to each model.

```python
# For GPT
template_gpt4 = """
Available function: {name}
Description: {description}
Parameters (JSON Schema): {json_schema}
"""

# For Claude
template_claude = """
<tool name="{name}">
  <description>{description}</description>
  <parameters>{formatted_params}</parameters>
</tool>
"""
```

### 2. Usage Instructions
These explain when and how the model should use tools, including the expected output format. This component handles the crucial "programming" of the model's behavior.

```python
instructions_reliable = """
When you need to use a tool, output EXACTLY this format:
<tool_call>
{"tool": "tool_name", "arguments": {...}}
</tool_call>

Never describe what you're going to do. Just do it.
Never output tool syntax outside the markers.
"""
```

### 3. Parsing Rules
Since LLM output is always plain text, even complex responses can be parsed using regular expressions. This component defines how to reliably extract tool calls from each model's unique output style.

```python
# Grok 4 needs aggressive patterns due to hallucination tendency
parsing_grok4 = {
    "primary": r'<tool_call>(.*?)</tool_call>',
    "fallback": r'\{"tool":\s*"([^"]+)",\s*"arguments":\s*(\{[^}]+\})\}',
    "strict_mode": True  # Reject ambiguous matches
}
```

Note the `fallback` pattern: the model was instructed to wrap tool calls in `<tool_call>` markers, but if it outputs raw JSON instead, the fallback catches it. This is already a form of **self-healing** -- see below.

These three components form a complete prompt strategy for each model or model family. They define how the model is instructed, how its output is interpreted, and how the driver bridges both ends. This design enables flexible, model-specific prompt optimization without touching application or driver code.


## PromptStrategy: Unifying the Building Blocks

The three building blocks above -- tool description templates, usage instructions, and parsing rules -- share a fundamental constraint: **they must be consistent with each other.** If the prompt tells the LLM to output JSON, the parser must expect JSON. If the prompt uses XML markers, the parser must match those markers. Mismatches between instruction and parsing cause silent failures that are extremely hard to debug.

The `PromptStrategy` abstraction formalizes this insight by treating the three components as a single **codec**: it **encodes** tools and instructions into the prompt, and **decodes** the LLM response in the same format. Whoever defines the prompt format also defines the parser -- they are inseparable.

```
┌─────────────────────────────────────────────────────┐
│              PromptStrategy (Codec)                  │
│                                                     │
│  ┌─────────────────┐    ┌───────────────────────┐   │
│  │  Encode          │    │  Decode               │   │
│  │                 │    │                       │   │
│  │  format_tools() │    │  parse_tool_call()    │   │
│  │  call_example() │    │  healing rules        │   │
│  │  system_template│    │  field aliases        │   │
│  └─────────────────┘    └───────────────────────┘   │
│                                                     │
│  Configuration: TOML file (no hardcoded strings)    │
└─────────────────────────────────────────────────────┘
```

### Key Properties

1. **Prompt and parser are always consistent.** A `JsonPromptStrategy` instructs the LLM to output JSON and parses JSON. An `XmlPromptStrategy` would instruct XML and parse XML. There is no way to accidentally mismatch instruction and parser because both live in the same object.

2. **Zero hardcoded prompt text.** All strings that reach the LLM -- the system template, the call example, retry prompts, even the healing regex rules -- are loaded from external TOML configuration files. Python code contains only logic (serialization, parsing, regex application), never prompt text.

3. **Hot-swappable without code changes.** Switching from JSON to XML format means loading a different TOML file, not changing driver code. Prompt improvements, healing rules for new model quirks, or entirely new formats can be deployed by updating a text file.

4. **Cascading configuration.** The strategy loads a package-bundled default TOML, which can be overridden by a project-local TOML. New entries are added; existing entries are replaced. This allows fine-tuning prompts for specific models or use cases without forking the driver.

### Configuration via TOML

A strategy's TOML file defines every text element:

```toml
[system_message]
template = """
You are a helpful assistant with access to these tools:
{tools}
When you need to use a tool, respond with ONLY this format:
{call_example}
"""

[call_example]
template = '{"tool": "tool-name", "arguments": {"key": "value"}}'

[parsing]
tool_field_aliases = ["tool", "name"]

[retry_prompts]
no_tool_field = "Return a JSON object with a 'tool' field."
unknown_tool = "No tool '{tool_name}' found. Available: {available}."

[[healing]]
pattern = '```\w*\s*\n?'
replacement = ''
comment = "Strip opening markdown fences"
```

This separation of text from code has a practical consequence: prompt engineers can iterate on prompt wording, healing patterns, and retry messages without touching Python/TypeScript code -- and without requiring a new release of the driver package.

### Relationship to DriverBase

The `DriverBase` class in the Python SDK consumes a `PromptStrategy` and wires it into the `MCSDriver` contract:

- `get_function_description()` calls `strategy.format_tools()`
- `get_driver_system_message()` fills `strategy.system_template` with tools and call example
- `process_llm_response()` calls `strategy.parse_tool_call()` and handles the result

Concrete drivers like `RestDriver`, `FilesystemDriver`, or orchestrators inherit from `DriverBase` and only implement `list_tools()` and `execute_tool()`. All LLM-facing logic -- prompt generation, response parsing, healing, retry handling -- is shared through the strategy, eliminating code duplication across drivers.


### Self-Healing

Because LLM output is always plain text, parsing can fail -- even when the model clearly intended a tool call. A JSON bracket might be missing, a field name misspelled, or the output wrapped in unexpected markdown fences. These are not logic errors but formatting errors, and they are predictable per model.

Before signaling `call_failed` to the client, a driver should attempt to fix known formatting issues automatically. This is self-healing: the driver applies model-specific correction patterns and retries parsing on the corrected output. Typical corrections include:

- **Fallback patterns** (as shown above): the model ignores the instructed format and outputs raw JSON -- a secondary regex catches it.
- **Stripping markdown fences**: the model wraps its JSON in ` ```json ... ``` `.
- **Fixing common JSON syntax errors**: trailing commas, missing closing brackets, single quotes instead of double quotes.
- **Normalizing field names**: the model writes `"function"` instead of `"tool"`.

If self-healing succeeds, the call proceeds as if it was correctly formatted from the start -- the client never sees the intermediate failure. Only when all correction attempts fail does the driver return `call_failed = true` with a `retry_prompt` that instructs the LLM to try again.

Self-healing is optional but recommended. It directly reduces the compound failure rate described above and is particularly valuable for models with known formatting quirks.


## Dynamic Prompt Loading
The true power of this approach emerges when prompts become dynamically loadable. Rather than hardcoding prompts in driver code, drivers can load model-specific configurations from a registry during initialization or when the active model changes. This enables several capabilities:

```python
# Load optimized prompts for the current model
provider = PromptProvider.from_url("https://prompt-registry.example/v1")
driver = MCSDriver(spec_url=api_spec, prompt_provider=provider)

# The driver automatically selects the best prompt configuration
system_msg = driver.get_driver_system_message(model_name="gpt-x")
```

With versioning and checksums, clients can pin to tested prompt versions for reproducibility while still benefiting from improvements when ready to upgrade. Critical fixes for model quirks can be deployed instantly without touching application code.


## What a Prompt-Centric Ecosystem Enables
This approach unlocks possibilities that were previously impractical:

- **Prompt marketplaces** where optimized patterns for specific model/tool combinations are shared, rated, and continuously improved by the community
- **Large-scale A/B testing** of prompt variations across millions of real interactions, with results feeding back into optimization
- **Automated optimization** using frameworks like DSPy to systematically improve prompts through iterative testing and refinement
- **Zero-day support** for new models through prompt packages that can be published within hours of a model release
- **Community-driven improvement** where one person's optimization work benefits thousands of applications immediately


## Token Management Strategies
As tool counts grow, token costs explode. A fundamental challenge when every tool adds to the system prompt. Consider an application with 50 tools. Even with concise descriptions, you might consume 5,000+ tokens before the user even asks a question. MCS should addresses this through two primary strategies:

### 1. Inline-spec Style
This approach embeds full tool descriptions directly in the prompt. While simple to implement and reliable for models that can handle large contexts, it becomes expensive quickly:

```
You have access to the following tools:

1. get_weather - Retrieves current weather for a city
   Parameters: {"city": "string", "units": "celsius|fahrenheit"}
   
2. search_web - Searches the internet for information
   Parameters: {"query": "string", "max_results": "integer"}
   
[... 48 more tools ...]
```

### 2. Schema-only Style
This approach provides minimal descriptions initially, with the ability to fetch details on demand. While more token-efficient, it requires models capable of multi-step reasoning:

```
Available tools: get_weather, search_web, calculate_mortgage, send_email, 
generate_image, analyze_sentiment, translate_text, create_calendar_event...

For detailed specifications, use: fetch_tool_spec(tool_name)
```

Advanced implementations might use hybrid approaches, clustering related tools or progressively disclosing capabilities based on conversation context.

### 3. Curated Toolset Style

Instead of exposing a full OpenAPI spec with dozens of endpoints, a driver can load a **curated toolset definition** that contains only the endpoints relevant to the use case. Assuming that this already will be the common case wiht ToolDrivers. This has two benefits:

- **Token reduction**: A typical REST API might expose 40+ endpoints, but a specific use case often only needs 3-5. Sending the full spec wastes tokens and increases the chance of the LLM choosing an irrelevant tool.
- **Custom descriptions**: The original API documentation is written for human developers, not for LLMs. A curated toolset can replace endpoint descriptions with LLM-optimized text that improves accuracy. This is what MCP does implicitly when someone writes a wrapper server with hand-crafted tool descriptions -- MCS makes it explicit and reusable without a wrapper.

A curated toolset is a plain JSON file that the driver loads instead of (or alongside) the raw spec:

```python
driver = RestHttpDriver(
    urls=["https://raw.githubusercontent.com/org/mcs-toolsets/main/crm-subset.json"]
)
```

Toolset definitions can be versioned via git, shared across teams, and distributed without custom infrastructure. This makes "pick the right endpoints with the right descriptions" a shareable, reviewable artifact rather than knowledge locked inside a single wrapper server.


## Model-Specific Optimization
Different models excel with different prompt styles, and these differences go beyond mere preference. They can mean the difference between reliable execution and complete failure. These patterns have emerged:

- **GPT-x** performs best with JSON schemas and explicit structural markers
- **Claude** excels with XML-style formatting and clear behavioral instructions
- **Llama-based models** often need more explicit examples and stronger guardrails
- **Gemini** requires specific instructions to prevent premature tool announcements
- **Grok** needs careful formatting to prevent training data hallucinations

The parsing rules must also adapt to each model's tendencies. Some models reliably use specific delimiters, while others require complex patterns to handle variations. This isn't just optimization. It's acknowledging that the success of any tool call depends entirely on the model's ability to follow the specific prompt format, not on the underlying transport mechanism.


## Implementation Considerations
While fully dynamic prompt loading represents the ideal end state, practical implementations should start simpler and evolve:

1. **Begin with override capability**: Allow prompt overrides via constructor parameters or mixins
2. **Build in model variants**: Support model-specific prompt variants within the driver
3. **Document thoroughly**: Clearly indicate which models have been tested and optimized
4. **Design for extension**: Provide clear extension points for future prompt providers
5. **Default gracefully**: Always include sensible defaults that work adequately across models


## The Paradigm Shift
This approach represents a double revolution in how we think about LLM integration:

### First Revolution (MCS Core)
Eliminate unnecessary protocols and use existing interfaces directly. Instead of creating new communication layers, leverage what already works—REST, GraphQL, WebSockets—and focus on the real challenge: making LLMs use them reliably.


### Second Revolution (Prompt Patterns)
Separate the LLM interaction layer entirely from the execution mechanism. This separation means:

- **Drivers become pure execution engines**, handling the mechanics of external system interaction without concerning themselves with LLM quirks
- **Prompts become tradeable assets**, optimized and shared like algorithms, with value that compounds over time
- **Model differences become configuration**, not code changes, enabling instant adaptation to new models
- **Reliability improvements happen at the prompt layer**, without touching infrastructure or business logic

The technology stack becomes completely autonomous from the LLMs themselves. Driver authors focus purely on bridging to external systems, while prompt engineers optimize the LLM interaction independently. When a new model appears, existing drivers immediately become compatible through new prompt configurations—no code changes required.


## Keep in Mind
Always design with flexibility in mind. External prompt overrides should be possible through constructor parameters or mixins to enable:

- **A/B testing** of prompt variations in production environments
- **Quick fixes** for model-specific issues without code deployment
- **Custom optimizations** for specific use cases or domains
- **Gradual migration** to dynamic prompt providers as they mature

Think of prompts as instructions that program the model for specific tasks. Just as you might load different configurations for different environments, you load different prompts for different models. With this approach, even quirky models like Grok can be programmed to behave reliably, turning their weaknesses into solved problems rather than ongoing frustrations.

---

Prompt patterns and dynamic optimization are one part of the larger picture. Section 11 explores how this -- together with the driver contract, ToolDrivers, and orchestrators -- creates a division of labor and an ecosystem where different roles can specialize and contribute independently.
