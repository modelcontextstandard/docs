---
title: 13. Next Steps
sidebar_position: 13
---

# 13 · Next Steps

## Core Standard Enhancements (Define in Spec for Consistency Across SDKs)
- **Finalize JSON-Schema for `DriverMeta` and capability flags**: This ensures metadata is machine-validatable, promoting interoperability. Include schemas for Bindings, Tools, and ToolParameters to standardize serialization/deserialization.
- **Decide if sync/async drivers are needed**: Specify optional async semantics in the standard (e.g., for I/O-heavy bridges); define when to use (e.g., streaming responses). SDKs handle language idioms (e.g., async/await in Python).
- ~~**Clear signaling in `process_llm_response` for call occurrence**~~: **Resolved in v0.4** -- `process_llm_response` now returns a `DriverResponse` object that carries `tool_call_result`, `call_executed`, `call_failed`, `call_detail`, and `retry_prompt`. The driver itself is fully stateless and thread-safe. See Section 3.
- ~~**Streaming and native tool-call support**~~: **Resolved in v0.5** -- `process_llm_response` now accepts `str | dict` (supporting both raw text and structured native tool-call objects) and an optional `streaming` flag. The `DriverResponse` gained a `messages` field that provides pre-formatted conversation entries the client can append directly to its message history. This shifts message formatting responsibility from the client to the driver. See Section 3.
- **Exception handling and error reporting vs. return values in error cases**: Define what clients expect (e.g., structured error objects with codes/messages for failures, raw results on success); SDKs adapt to language-specific exceptions/logging.
- ~~**Should the output from `process_llm_response()` really be ANY or a structured object?**~~: **Resolved in v0.5** -- `DriverResponse` now separates `tool_call_result` (raw tool output, only meaningful on success) from `messages` (pre-formatted conversation entries). The client no longer receives the unchanged LLM input in `result`; instead, `messages` provides everything needed for the conversation history.
- **Driver versioning, registries, dynamic loading for true Plug & Play and max security**: Outline guidelines in the spec (e.g., semver rules, registry discovery protocols, checksum requirements); include autoloading via names/checksums. SDKs implement loading mechanics (e.g., Pip integration).
- **Autostart recommendation (container labels, health endpoints)**: Expand with virtualization mandates (e.g., Docker params, sandboxing rules) and startup guidelines; define how drivers signal needs (e.g., via metadata). SDKs provide reference frameworks for launching.
- **Explore a Prompt Provider for dynamic loading**: Add informative section on future extensions for external prompt overrides/loading (e.g., via URLs/registries); define override semantics in init. Prototype in SDKs before standardizing.
- **Hybrid driver-orchestrator patterns**: Clarify in spec how drivers can implement both MCSDriver and MCSToolDriver interfaces for versatility; provide guidelines on when to use hybrids. SDKs offer examples.
- **Multi-connection support within a single driver**: Currently, a driver handles one binding with one connection. Multiple connections of the same binding (e.g., three REST APIs) require an Orchestrator. An alternative model: allow a single driver to hold multiple connections natively (e.g., accept a list of URLs). This would reduce the Orchestrator's role to pure multi-binding coordination (REST + Filesystem + ...), simplifying the architecture for the common case of "same protocol, multiple endpoints." Trade-offs: a multi-connection driver is more complex internally but removes an Orchestrator layer for the most frequent use case. This needs prototyping and community feedback before standardizing.
- **User Consent before tool execution**: Currently `process_llm_response` is atomic -- it parses the LLM output, detects a tool call, and executes it in a single step. The client has no opportunity to intervene between detection and execution. This becomes critical when:
  - The user should approve tool calls before they happen (e.g., "The AI wants to send an email to X -- allow?")
  - Cost or rate-limit awareness requires gating (e.g., paid API calls)
  - Audit trails demand explicit human-in-the-loop confirmation
  - Safety-sensitive operations need review (e.g., file deletion, financial transactions)

  The **ToolDriver** interface already provides a natural separation: the orchestrator parses the LLM output, extracts the tool name and arguments, and calls `execute_tool` as a discrete step. A consent check fits naturally between parse and execute. But **standalone MCSDriver** has no such seam -- `process_llm_response` does everything at once.

  Possible approaches under discussion:
  1. **Two-phase `process_llm_response`**: Split into `detect_tool_call(llm_response) -> ToolCallIntent | None` (parse only, no execution) and `execute_tool_call(intent) -> DriverResponse` (execute). The client decides what happens in between.
  2. **Consent callback**: Pass an optional `on_before_execute(tool_name, arguments) -> bool` callback to the driver. The driver calls it before execution and aborts if it returns `False`.
  3. **Accept that consent is an orchestrator concern**: Standalone drivers execute immediately by design. If consent is required, the architecture should use a ToolDriver + Orchestrator, where the orchestrator provides the consent seam. This is consistent with the principle that standalone drivers optimize for simplicity.

  The third approach aligns well with the existing contract: it keeps the standalone driver simple while the orchestrator pattern naturally supports consent. However, option 1 would make consent possible without requiring a full orchestrator. This needs further discussion and community feedback.

- **Raw source spec access**: Should drivers expose the *original* unprocessed source specification (e.g. the raw OpenAPI file before any reduction or transformation)? Currently `get_function_description` returns whatever the driver has prepared, which may be reduced, reformatted, or generated from `Tool` objects. A capability like `"raw_spec"` could signal that the original source is available, but no concrete use case has been identified yet. If needed, this could be modeled as a mixin or a method on `DriverMeta`.

### Package Naming Convention & Registry (for review)

A consistent naming scheme is essential for package discovery via PyPI, npm, and the planned `mcs-pkg` registry. The following convention is under consideration:

**Two prefixes only:**

```
mcs-driver-<protocol>-<transport>[-<variant>]     # Drivers (all types)
mcs-orchestrator-<strategy>[-<variant>]            # Orchestrators
```

**Driver type defaults to hybrid** (implements both MCSDriver and MCSToolDriver). When a driver only supports one mode, the variant suffix makes this explicit:

| Package name | Type | Usage |
|---|---|---|
| `mcs-driver-rest-http` | Hybrid (default) | Standalone or via Orchestrator |
| `mcs-driver-rest-http-petstore` | Hybrid, variant | Specific API, both modes |
| `mcs-driver-rest-http-standalone` | Standalone only | Direct client use, no `list_tools()`/`execute_tool()` |
| `mcs-driver-filesystem-localfs-toolonly` | Tool only | Orchestrator required, no LLM-facing methods |
| `mcs-orchestrator-basic` | Orchestrator | Aggregates ToolDrivers |
| `mcs-orchestrator-weighted` | Orchestrator | Priority-based routing |

**Design rationale:**

- `<protocol>-<transport>` maps directly to the Binding concept in `DriverMeta`. Each driver handles exactly one pair.
- `[-<variant>]` is freely chosen by the author to distinguish implementations, API-specific drivers, or capability restrictions (`-standalone`, `-toolonly`).
- Discovery stays simple: `pip search mcs-driver-rest-http` finds all REST-HTTP drivers regardless of type. Web-based package browsers (PyPI, npm) can filter by prefix without needing capability metadata.
- `DriverMeta.capabilities` carries the machine-readable type information (`"standalone"`, `"orchestratable"`) for programmatic discovery via `mcs-pkg`.

**Language-specific conventions are SDK responsibility:**

The specification defines the *logical* naming structure (`mcs-driver-<protocol>-<transport>[-<variant>]`), but each language ecosystem must solve discovery through its own SDK and existing package infrastructure. The goal: drivers must be discoverable and installable *without* a central MCS registry from day one. For Python, this means PyPI prefix search; for TypeScript, npm scoped packages or prefix conventions; for Go, module paths; and so on. Each SDK documents its own convention. A central registry (`mcs-pkg`) may complement these later, but must never be a prerequisite.

**SDKs define three naming levels**, not just one:

| Level | Spec defines | SDK defines |
|---|---|---|
| **Package manager** | Logical prefix `mcs-driver-<protocol>-<transport>` | Mapping to ecosystem (PyPI, npm, crates.io, ...) |
| **Import / module path** | — | Language-idiomatic import convention for IDE autocompletion |
| **Class / file naming** | — | Internal structure and naming rules for code navigation |

The spec only governs the first level (the logical prefix). The import convention and file layout are SDK-internal decisions -- but they are equally important because they determine how developers *find and use* drivers in their IDE. Predictable import paths enable autocompletion: typing `from mcs.driver.` in Python lists all installed drivers; typing `import { } from "mcs-driver-rest-http"` in TypeScript does the same. Each SDK must document its full naming stack (package manager name → import path → class/file names) so that all three levels are consistent and discoverable.

**Open questions:**

- Should `-standalone` and `-toolonly` be reserved suffixes enforced by `mcs-pkg`, or just a convention?
- Should `mcs-pkg` enforce the `<protocol>-<transport>` structure, or allow freeform names with metadata-based classification?

### ToolCallSignalingMixin -- Inline tool-call detection during streaming (prototyped in Python SDK)

When an LLM streams text token-by-token and does not provide native tool-call events (e.g. older or local models), the client cannot distinguish tool-call JSON from regular text until the stream ends.  By then, the raw JSON has already been displayed to the user.

**Proposed approach:** An opt-in mixin `ToolCallSignalingMixin` that gives the driver a lightweight signaling interface the client can query during streaming:

- `might_be_tool_call(partial: str) -> bool` -- fast heuristic on a few accumulated tokens.  Returns `True` if the text *could* be the start of a tool call.  The client uses this to pause display and start buffering.
- `is_complete_tool_call(text: str) -> bool` -- checks whether the buffered text is a fully parseable tool call ready for `process_llm_response`.

**Design constraints:**

- The mixin is **opt-in** and does not change the core `MCSDriver` interface.  Clients detect support via `isinstance(driver, ToolCallSignalingMixin)`.
- Both methods are **pure functions** on the provided text -- the driver stays stateless.  All buffering, timeout, and display logic is the client's responsibility.
- The heuristic operates like a "magic byte" check: the client buffers a small token window and asks the driver whether it could be a tool call opener (e.g. `{`, `` ``` ``).  This avoids the latency of routing every token through the driver.
- **Timeout handling** is a client concern: if `might_be_tool_call` returned `True` but `is_complete_tool_call` does not confirm within N milliseconds, the client flushes the buffer and displays it.  Different clients may choose different strategies (spinner, delayed display, etc.).
- When the LLM writes explanatory text *before* the JSON ("Let me look that up: `{...}`"), `might_be_tool_call` triggers mid-stream at the `{`.  The user sees the preceding text and then a brief pause while the tool executes -- which is natural UX.

**Status:** A reference implementation exists in the Python SDK (`ToolCallSignalingMixin` in `mcs-driver-core`, demonstrated in `mcs-examples/` with `csv_localfs_driver_tcs.py` and `mcs_driver_minimal_client_stream_tcs.py`).  The design should be validated across more drivers and models before promoting to a standard recommendation.

### Provider Tool-Call Format Helpers (for evaluation)

LLM providers return tool calls in vastly different wire formats:

| Provider | Format |
|---|---|
| OpenAI / Azure | `tool_calls` array with `function.name` + `function.arguments` (JSON string) |
| Anthropic Claude | `tool_use` content block with `name` + `input` (object) |
| Google Gemini | `functionCall` with `name` + `args` (object) |
| Ollama / local models | Plain-text JSON in assistant content (no structured event) |

Currently each driver must handle all formats it wants to support.  This leads to duplicated parsing logic across drivers and makes it harder for driver authors who are not LLM specialists.

**Proposed approach:**  Provide a set of **format helpers** (utility functions or lightweight mixins) in `mcs-driver-core` that normalize provider-specific tool-call representations into a common `{"tool": ..., "arguments": ...}` dict.  A driver author can then call a single helper before executing the tool, instead of implementing format-specific parsing.

Key design constraints:

- The helpers are **opt-in utilities**, not part of the core `MCSDriver` interface.  The contract stays format-agnostic.
- `process_llm_response` already accepts `str | dict` (v0.5).  The helpers operate at the layer *before* the driver is called (client-side) or *inside* the driver (driver-side), depending on the integration style.
- The format decision is driven by the **wire format**, not the model name.  A helper like `normalize_openai_tool_call(delta)` or `normalize_anthropic_tool_use(block)` makes the format explicit.  There is no need for a `model_name` parameter on `process_llm_response` itself -- the format is what matters, not who produced it.
- Each SDK evaluates which provider formats to support and ships the appropriate helpers.  The spec does not mandate specific providers.

**Open question:**  Should these helpers live in `mcs-driver-core` (zero extra dependencies, string/dict only) or in a separate `mcs-driver-llm-utils` package that can depend on provider SDKs for type safety?

### Runner Concept (for discussion)

The OpenAI Agents SDK introduces a "Runner" that encapsulates the entire LLM-tool execution loop: LLM call, tool-call detection, execution, history management, and iteration -- all in a single `run(input) -> output` call. The developer never writes the loop; the Runner does everything.

With MCS v0.5 and the `messages` field on `DriverResponse`, a similar convenience layer becomes trivially implementable:

```pseudo
class MCSRunner {
    driver: MCSDriver
    llm_client: LLMClient

    run(user_input: string) -> string {
        messages = [system_prompt, user_message]
        loop {
            llm_output = llm_client.chat(messages)
            response = driver.process_llm_response(llm_output)
            if response.messages:
                messages.extend(response.messages)
            if not response.call_executed and not response.call_failed:
                return llm_output  // final answer
        }
    }
}
```

**How it relates to existing MCS concepts:**

| Concept | Direction | What it aggregates |
|---|---|---|
| **Orchestrator** | Horizontal | Multiple ToolDrivers into one MCSDriver |
| **Runner** | Vertical | MCSDriver + LLM + Loop into one `run()` call |

A Runner is *not* an Orchestrator in the strict MCS sense. An Orchestrator aggregates tools across drivers and is transparent to the client (it *is* an MCSDriver). A Runner automates the client-side conversation loop and sits *above* the driver layer.

However, a Runner *could* be implemented as an MCSDriver -- its `process_llm_response` would internally run the full loop and return only the final answer. This makes it chainable but breaks the principle that the client controls the loop. For simple use cases (chatbots, scripts, batch processing), this trade-off is acceptable.

**Recommendation:** SDKs may offer a Runner as an optional convenience class (similar to `BasicOrchestrator`). Complex clients that need streaming, human-in-the-loop consent, or custom UI continue to use the manual loop. The Runner is opt-in, not a replacement for the core contract.

## Defer to SDKs (Implementation Details, Not Core Spec)
- **Provide language-specific reference interfaces in `python-sdk` / `typescript-sdk`**: Fully shift to SDKs as the standard is agnostic; use them to bootstrap other languages (e.g., Go, Rust) with code samples.
- **Checksums and autoloading in clients via names alone**: Spec recommends security practices (e.g., mandatory verification); SDKs build autoload logic, handling platform-specific package management and resolution.
- **Exception handling nuances and type hints**: Spec defines semantics (e.g., what to return/raise); SDKs adapt with language features (e.g., typed errors in TypeScript).
- **Collect community feedback**: Ongoing for the standard (e.g., via GitHub issues); SDKs incorporate into best practices/examples, feeding back to spec revisions.
