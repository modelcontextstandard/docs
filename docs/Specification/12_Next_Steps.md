---
title: 12. Next Steps
sidebar_position: 12
---

# 12 · Next Steps

## Core Standard Enhancements (Define in Spec for Consistency Across SDKs)
- **Finalize JSON-Schema for `DriverMeta` and capability flags**: This ensures metadata is machine-validatable, promoting interoperability. Include schemas for Bindings, Tools, and ToolParameters to standardize serialization/deserialization.
- **Decide if sync/async drivers are needed**: Specify optional async semantics in the standard (e.g., for I/O-heavy bridges); define when to use (e.g., streaming responses). SDKs handle language idioms (e.g., async/await in Python).
- **Clear signaling in `process_llm_response` for call occurrence**: Standardize return types (e.g., a tuple like (result, was_called: bool)) or flags to indicate execution; include error conventions (e.g., raise vs. return error objects). This avoids ambiguity in chaining.
- **Exception handling and error reporting vs. return values in error cases**: Define what clients expect (e.g., structured error objects with codes/messages for failures, raw results on success); SDKs adapt to language-specific exceptions/logging.
- **Should the output from `process_llm_response()` really be ANY or a structured object?**: Mandate a minimal envelope (e.g., JSON with ```{ "result": any, "status": int, "error": string? }```) for metadata like status codes; keeps flexibility while ensuring parseability. SDKs can add type safety.
- **Driver versioning, registries, dynamic loading for true Plug & Play and max security**: Outline guidelines in the spec (e.g., semver rules, registry discovery protocols, checksum requirements); include autoloading via names/checksums. SDKs implement loading mechanics (e.g., Pip integration).
- **Autostart recommendation (container labels, health endpoints)**: Expand with virtualization mandates (e.g., Docker params, sandboxing rules) and startup guidelines; define how drivers signal needs (e.g., via metadata). SDKs provide reference frameworks for launching.
- **Explore a Prompt Provider for dynamic loading**: Add informative section on future extensions for external prompt overrides/loading (e.g., via URLs/registries); define override semantics in init. Prototype in SDKs before standardizing.
- **Hybrid driver-orchestrator patterns**: Clarify in spec how drivers can implement both MCSDriver and MCSToolDriver interfaces for versatility; provide guidelines on when to use hybrids. SDKs offer examples.

## Defer to SDKs (Implementation Details, Not Core Spec)
- **Provide language-specific reference interfaces in `python-sdk` / `typescript-sdk`**: Fully shift to SDKs as the standard is agnostic; use them to bootstrap other languages (e.g., Go, Rust) with code samples.
- **Checksums and autoloading in clients via names alone**: Spec recommends security practices (e.g., mandatory verification); SDKs build autoload logic, handling platform-specific package management and resolution.
- **Exception handling nuances and type hints**: Spec defines semantics (e.g., what to return/raise); SDKs adapt with language features (e.g., typed errors in TypeScript).
- **Collect community feedback**: Ongoing for the standard (e.g., via GitHub issues); SDKs incorporate into best practices/examples, feeding back to spec revisions.