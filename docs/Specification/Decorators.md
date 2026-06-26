---
title: Decorators
sidebar_position: 7.5
---

# Decorators

[Optional Capabilities](7_Optional_Capabilities.md) covered features that
**add new methods** to a driver (`healthcheck`, `get_native_tool_context`, …).
Those are pure *contracts*: a driver either implements the method or it does
not, and the rest of its behaviour is untouched.

Some cross-cutting concerns are different. **Authentication**,
**permission/approval**, and **lifecycle hooks** do not add an isolated
method — they need to *intervene in an existing operation*, namely the
tool-call execution. This section describes how MCS expresses that:
through **decorators**.

## Why this shape

Two goals point the same way, and the whole design serves them:

1. **The driver author stays simple.** A plain driver implements only the two
   mandatory methods (plus any optional contracts it wants). It never deals
   with capability aggregation or resolution -- the reference implementation
   (`DriverBase`) and the decorators handle that. A plain driver is matched
   directly and needs none of the machinery below. (`DriverBase`, for example,
   advertises its own capabilities via the idempotent `DriverMeta.with_capability`
   helper, so a subclass gets them automatically yet may also list them
   explicitly -- the author chooses.)
2. **The client sees one thing.** Every `MCSDriver` is treated identically,
   whether it is a plain driver, an orchestrator, or a decorator. Detection
   (`meta.has_capability`) and resolution (`DriverMeta.resolve_capability`)
   work uniformly, so the client never has to know what the stack contains --
   it just gets a driver, no matter what was injected via dependency injection.

In short: the capability-aggregation and resolution machinery exists **only**
to make wrappers (decorators, orchestrators) transparent. Without composition
there would be nothing to resolve -- a plain driver would not need any of it.

## What a decorator is

A decorator is **a driver that wraps another driver**. It satisfies the same
`MCSToolDriver` (and, where relevant, `MCSDriver`) interface as the driver it
wraps, delegates every interface call to the inner instance, and inserts its
own behaviour around the point it cares about — typically `execute_tool`.

```pseudo
class AuthDecorator implements MCSToolDriver:
    inner: MCSToolDriver

    list_tools()              -> inner.list_tools()          # delegate
    execute_tool(name, args)  -> try inner.execute_tool(name, args)
                                 catch AuthChallenge -> describe the auth step
```

Because a decorator *is* a driver, the core promise of MCS holds: **everything
is a driver.** A decorator can be injected via dependency injection exactly
like any other driver — from the outside it simply looks like a driver, and
the client need not know what the stack contains. This is the same principle
the [Orchestrator](5_Orchestrator.md) already uses to stack drivers.

## Why decorators (and not mix-ins)

Implementation inheritance (mix-ins that override `execute_tool`) can achieve
the same effect in a single language, but it has three problems that matter
for a cross-language **standard**:

1. **Portability.** Multiple inheritance of *implementation* does not exist in
   many OO languages (Java, C#, Go, …). Pure *contracts* (interfaces) compose
   everywhere; inherited behaviour does not. A standard must not depend on a
   language-specific mechanism.
2. **Order.** With inheritance, the order of layers is fixed by the class
   declaration and reads as arbitrary. With decorators the order is the
   **nesting order**, chosen by the client at composition time:
   `Permission(Auth(RealToolDriver))`.
3. **Combinatorial explosion.** Mix-ins force one driver class per capability
   combination (`Rest+Auth`, `Rest+Auth+Permission`, …). A decorator is a
   single composable object that applies to *any* inner driver.

Decorators trade inheritance for **composition**: portable, explicitly
ordered, and free of the N×M class explosion.

## How they fit the overall picture

- **Contract vs. implementation.** A capability still has a *contract*
  (`SupportsAuth`, `SupportsPermission`, `SupportsHooks`) that declares the
  feature and carries its `CAPABILITY` flag. The decorator is the *example
  implementation* of that contract — exactly as `DriverBase` is the example
  implementation of the core contracts. A driver author may also build the
  behaviour directly into their own driver instead of using the decorator.
- **Detection and invocation, never `isinstance`.** A decorator aggregates
  the inner driver's `capabilities` and appends its own, so
  `DriverMeta.capabilities` reflects the **whole** stack. `isinstance` only
  sees the outermost layer and would miss a capability provided deeper down.
  Two operations replace it, and both go through `DriverMeta`:
    - *Detection* ("is it there?"):
      `driver.meta.has_capability(Contract)` — a pure read over the
      aggregated flags.
    - *Invocation* ("give me the object that provides it"):
      `DriverMeta.resolve_capability(driver, Contract)` returns the layer that
      satisfies the contract — typed, no cast — by searching inward through
      the stack. Wrappers implement `SupportsCapabilityResolution` (a single
      `resolve_capability` method that delegates to the inner driver) so
      resolution reaches any depth; a plain driver is matched directly on
      itself. This is why the client can treat every `MCSDriver` the same,
      whether it is a plain driver, an orchestrator, or a decorator.
- **Where the decorator sits.** Auth/permission decorators wrap the
  **ToolDriver** layer (`execute_tool`), reachable through the existing
  composition seam a driver already uses to delegate execution. They do *not*
  wrap `process_llm_response`; the surrounding driver keeps its single
  response-processing loop, and the decorator intercepts execution beneath it.

## Summary

| Concern type | Mechanism |
| --- | --- |
| Adds a new method (`healthcheck`, `get_native_tool_context`) | **Contract** (interface), detected via its `CAPABILITY` flag |
| Intervenes in tool execution (auth, permission, hooks) | **Decorator** (a wrapping driver), composed via DI, detected via its `CAPABILITY` flag |

In both cases the **core contract stays minimal** and the **client detects
features through `meta.capabilities`** — keeping MCS portable and keeping
every layer, plain or decorated, just a driver.
