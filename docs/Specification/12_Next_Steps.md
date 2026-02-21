---
title: 12. Next Steps
sidebar_position: 12
---

# 12 · Next Steps

## Core Standard Enhancements (Define in Spec for Consistency Across SDKs)
- **Finalize JSON-Schema for `DriverMeta` and capability flags**: This ensures metadata is machine-validatable, promoting interoperability. Include schemas for Bindings, Tools, and ToolParameters to standardize serialization/deserialization.
- **Decide if sync/async drivers are needed**: Specify optional async semantics in the standard (e.g., for I/O-heavy bridges); define when to use (e.g., streaming responses). SDKs handle language idioms (e.g., async/await in Python).
- ~~**Clear signaling in `process_llm_response` for call occurrence**~~: **Resolved in v0.4** -- `process_llm_response` now returns a `DriverResponse` object that carries `result`, `call_executed`, `call_failed`, `call_detail`, and `retry_prompt`. The driver itself is fully stateless and thread-safe. See Section 3.
- **Exception handling and error reporting vs. return values in error cases**: Define what clients expect (e.g., structured error objects with codes/messages for failures, raw results on success); SDKs adapt to language-specific exceptions/logging.
- **Should the output from `process_llm_response()` really be ANY or a structured object?**: Mandate a minimal envelope (e.g., JSON with ```{ "result": any, "status": int, "error": string? }```) for metadata like status codes; keeps flexibility while ensuring parseability. SDKs can add type safety.
- **Driver versioning, registries, dynamic loading for true Plug & Play and max security**: Outline guidelines in the spec (e.g., semver rules, registry discovery protocols, checksum requirements); include autoloading via names/checksums. SDKs implement loading mechanics (e.g., Pip integration).
- **Autostart recommendation (container labels, health endpoints)**: Expand with virtualization mandates (e.g., Docker params, sandboxing rules) and startup guidelines; define how drivers signal needs (e.g., via metadata). SDKs provide reference frameworks for launching.
- **Explore a Prompt Provider for dynamic loading**: Add informative section on future extensions for external prompt overrides/loading (e.g., via URLs/registries); define override semantics in init. Prototype in SDKs before standardizing.
- **Hybrid driver-orchestrator patterns**: Clarify in spec how drivers can implement both MCSDriver and MCSToolDriver interfaces for versatility; provide guidelines on when to use hybrids. SDKs offer examples.
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

The spec only governs the first level (the logical prefix). The import convention and file layout are SDK-internal decisions -- but they are equally important because they determine how developers *find and use* drivers in their IDE. Predictable import paths enable autocompletion: typing `from mcs.drivers.` in Python lists all installed drivers; typing `import { } from "mcs-driver-rest-http"` in TypeScript does the same. Each SDK must document its full naming stack (package manager name → import path → class/file names) so that all three levels are consistent and discoverable.

**Open questions:**

- Should `-standalone` and `-toolonly` be reserved suffixes enforced by `mcs-pkg`, or just a convention?
- Should `mcs-pkg` enforce the `<protocol>-<transport>` structure, or allow freeform names with metadata-based classification?

## Defer to SDKs (Implementation Details, Not Core Spec)
- **Provide language-specific reference interfaces in `python-sdk` / `typescript-sdk`**: Fully shift to SDKs as the standard is agnostic; use them to bootstrap other languages (e.g., Go, Rust) with code samples.
- **Checksums and autoloading in clients via names alone**: Spec recommends security practices (e.g., mandatory verification); SDKs build autoload logic, handling platform-specific package management and resolution.
- **Exception handling nuances and type hints**: Spec defines semantics (e.g., what to return/raise); SDKs adapt with language features (e.g., typed errors in TypeScript).
- **Collect community feedback**: Ongoing for the standard (e.g., via GitHub issues); SDKs incorporate into best practices/examples, feeding back to spec revisions.