---
title: 6. Decorators
sidebar_position: 6
---

# 6 · Decorators

The [Orchestrator](5_Orchestrator.md) showed the first kind of composition:
many drivers aggregated behind a single `MCSDriver` interface. **Decorators**
are the second kind — and they are the reason for the capability machinery that
[Optional Capabilities](8_Optional_Capabilities.md) describes.

A driver exposes optional behaviour in two ways. Most features simply **add a
new method** — a health check, a native-tool context, … — and leave everything
else untouched. But some cross-cutting concerns — **authentication**,
**permission/approval**, **lifecycle hooks** — do not add an isolated method;
they need to *intervene in an existing operation*, namely the tool-call
execution. MCS expresses that through **decorators**.

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
is a driver.** It can be injected via dependency injection exactly like any
other driver — from the outside it simply looks like a driver, and the client
need not know what the stack contains. This is the same principle the
[Orchestrator](5_Orchestrator.md) already uses to stack drivers.

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

## Where a decorator sits

Auth/permission decorators wrap the **ToolDriver** layer (`execute_tool`),
reachable through the existing composition seam a driver already uses to
delegate execution. They do *not* wrap `process_llm_response`; the surrounding
driver keeps its single response-processing loop, and the decorator intercepts
execution beneath it.

A capability built as a decorator still has a *contract* (`SupportsAuth`,
`SupportsPermission`, …) that declares it and carries its `CAPABILITY` flag —
the decorator is just the *example implementation*, exactly as `BaseDriver` is
the example implementation of the core contracts. A driver author may also
build the behaviour directly into their own driver instead of using a decorator.

## Why this drives the capability machinery

Because a wrapper hides the inner layers, a plain `isinstance` check — which
only sees the outermost object — can no longer find a capability provided
deeper down. That is the entire reason `DriverMeta` carries its
detection / declaration / resolution operations, described in
[Optional Capabilities → How `DriverMeta` carries capabilities](8_Optional_Capabilities.md#how-drivermeta-carries-capabilities).
The payoff is twofold, and it is why this pattern was chosen:

- **The driver author stays simple.** A plain driver implements only its
  mandatory methods (plus any optional contracts) and needs none of that
  machinery — it is matched directly.
- **The client sees one thing.** Every `MCSDriver` is treated identically —
  plain driver, orchestrator, or decorator — so the client never has to know
  what the stack contains, no matter what was injected.

A practical consequence: a decorator delegates the *interface* (`list_tools`,
`execute_tool`), not arbitrary methods. So an inner capability such as
`healthcheck` is **advertised** through the stack (`meta.capabilities` aggregates
it) but **reached** via `resolve_capability` — never called directly on the
wrapper. See [Optional Capabilities → Using a capability from the client's
side](8_Optional_Capabilities.md#using-a-capability-from-the-clients-side) for
the detect → resolve → call pattern.

## Reference implementation

The Python SDK ships a reusable `BaseDecorator` in **`mcs-driver-core`**. It
delegates every interface call to a single inner driver and resolves
capabilities by searching inward — the one-inner counterpart to what the
[Orchestrator](5_Orchestrator.md) does for many. A concrete decorator
(`AuthDecorator`, `PermissionDecorator`, …) subclasses it and overrides only
the method it intercepts, usually `execute_tool`.

It lives in the kernel for the same reason `BaseDriver` does: it is **pure
composition mechanism** — delegation plus stack-navigation — zero-dependency,
with no concept of its own. The orchestrator is *also* a wrapping driver, but
it carries an abstraction of its own (a pluggable resolution strategy), so it
ships as a separate package (`mcs-orchestrator-base`). The rule of thumb: pure
mechanism stays in the kernel; composition that carries its own strategy
becomes its own package.

## Summary

| Concern type | Mechanism |
| --- | --- |
| Adds a new method (`healthcheck`, `get_native_tool_context`) | **Contract** (interface), advertised via its `CAPABILITY` flag |
| Intervenes in tool execution (auth, permission, hooks) | **Decorator** (a wrapping driver), composed via DI, advertised via its `CAPABILITY` flag |

In both cases the **core contract stays minimal** and features are detected
through `meta.capabilities` — keeping MCS portable and keeping every layer,
plain or decorated, just a driver.
