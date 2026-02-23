---
title: 5. Orchestrator
sidebar_position: 5
---

# 5 · Orchestrator

An Orchestrator is not just a router -- it is an **interface author**. It constructs a new, curated tool interface by aggregating tools from multiple ToolDrivers, resolving name collisions, setting priorities, enriching descriptions, and deciding the order and density of tools presented to the LLM.

## When you need an Orchestrator

A hybrid driver handles one capability. When an application needs **multiple capabilities** or **multiple connections** of the same capability, an Orchestrator coordinates them.

Typical scenarios:

| Scenario | What aggregates | Example |
|---|---|---|
| Multiple connections, same capability | Several instances of the same ToolDriver with different endpoints | Three REST APIs (CRM, ERP, payment gateway) |
| Multiple capabilities | Different ToolDrivers for different capabilities | REST API + local filesystem + message bus |
| Domain orchestration | A curated set of tools for a specific domain | Finance: REST (market data) + filesystem (reports) + FIX (trading) |

The Orchestrator implements `MCSDriver`. It collects tools from all registered ToolDrivers, builds a unified prompt, and dispatches tool calls to the correct driver. For the AI client, it appears as a single driver. To be composable itself, the Orchestrator should also implement `MCSToolDriver` and expose the aggregated tool list via `list_tools()` -- this allows it to be embedded as a building block in a higher-level Orchestrator.

## Orchestrator responsibilities

1. **Tool aggregation**: Collects tools from all ToolDrivers via `list_tools()`.
2. **Unified prompt**: Builds a single `get_driver_system_message()` that presents all tools in a consistent format. Every ToolDriver contributes its tools, but the Orchestrator owns the final prompt -- it decides naming, ordering, and description style.
3. **Dispatch**: Parses the LLM response, identifies which tool was called, routes to the correct ToolDriver via `execute_tool()`.
4. **Conflict resolution**: When multiple ToolDrivers expose tools with the same name, the Orchestrator must resolve the collision. Strategies include:
   - **Namespacing** (e.g. `crm_list_contacts` vs `erp_list_contacts`): more tools, but deterministic. The Orchestrator adjusts descriptions so the LLM uses namespaced names, and maps them back before dispatching.
   - **Priority ordering**: one ToolDriver takes precedence; the other is tried only if the first rejects.
   - **Rejecting the conflict** at registration time.
   - **Active-target switching**: fewer tools, but the client or user switches the "active" target via a command. This makes the Orchestrator stateful. If statelessness is a hard requirement, switching must happen as a client-side reinstantiation or the architecture falls back to namespacing. Both approaches are valid; the important thing is a clear policy so the LLM does not have to guess.

## Orchestrators are composable

When an Orchestrator implements `MCSToolDriver`, it can itself be wrapped by another Orchestrator. This allows layered compositions: a domain-specific Orchestrator aggregating capability-level Orchestrators, each managing multiple connections.

```
AI Client
  └── Finance Orchestrator (MCSDriver)
        ├── REST Orchestrator (MCSDriver)
        │     ├── Market-Data ToolDriver (REST/HTTP)
        │     └── CRM ToolDriver (REST/HTTP)
        ├── Filesystem ToolDriver (FS/local)
        └── FIX ToolDriver (FIX/TCP)
```

## Orchestrators as ToolDrivers (recursive stacking)

An Orchestrator that also implements `MCSToolDriver` becomes a building block for higher-level Orchestrators. This enables recursive stacking: a domain Orchestrator aggregates capability-level Orchestrators, each of which aggregates individual ToolDrivers.

This works because the contract is uniform at every level:
- The **client** needs `MCSDriver` -- it gets it from the top-level Orchestrator.
- Each **Orchestrator** needs `MCSToolDriver` inputs -- it gets them from hybrid drivers *or* from other Orchestrators that also implement `MCSToolDriver`.
- Each **hybrid driver** implements both interfaces, so it can participate at any level.

This is why hybrid is the practical default: a driver that implements both interfaces can be used standalone, stacked via an Orchestrator, or even used as a sub-Orchestrator -- without any code changes.

If an Orchestrator only implements `MCSDriver` (not `MCSToolDriver`), it serves as a terminal entry point for the client and cannot be embedded further. Both patterns are valid; hybrid is simply more flexible.

## Summary

| Component | Implements | LLM knowledge | Role |
|---|---|---|---|
| **ToolDriver** | `MCSToolDriver` | None | Pure bridge; input to Drivers and Orchestrators |
| **Driver (hybrid)** | `MCSDriver` + `MCSToolDriver` | Yes | Standalone or stackable; see [Section 4](4_ToolDriver_Adapter.md) |
| **Orchestrator** | `MCSDriver` (optionally `MCSToolDriver`) | Yes | Interface author; aggregates, curates, and dispatches |
