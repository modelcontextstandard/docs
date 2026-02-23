---
title: 6. Configuration & Instantiation
sidebar_position: 6
---

# 6 · Configuration & Instantiation

All driver configuration happens via the **constructor**. URLs, auth tokens, proxies, adapter selection, connection parameters -- everything a driver needs is passed at creation time. This decouples configuration from functionality and keeps the client agnostic: it does not need to know what a specific driver requires internally.

This constructor-based approach enables dynamic setups: configuration can come from environment variables, JSON files, dependency injection, or a UI form. The motivation is to allow drivers to be fully automatically loaded and configured in the future, without requiring the client to implement special logic for particular tools.

This is the foundation for **hot-loading and unloading drivers at runtime** -- much like an operating system loading a device driver on demand. A client could discover a new driver package, instantiate it with the right configuration, and make its tools available to the LLM immediately, without restarting. The driver model makes this possible because the interface is uniform: from the client's perspective, every driver is just another `MCSDriver`, regardless of when it was loaded and what it contains.

With validation libraries like Pydantic, a client could even **dynamically generate a configuration interface** from the driver's constructor signature -- presenting the user with the right fields without the client knowing the driver's internals. This is similar to how traditional device drivers bring their own configuration dialogs without involving the operating system.

Drivers are **immutable after construction** -- there are no setters the client needs to call. This keeps the interface simple and predictable: construct, use, done.

The one exception is the Orchestrator. Because an Orchestrator manages a set of ToolDriver instances, it is practical to allow adding or removing connections at runtime (e.g. registering a new API URL or detaching one). The Orchestrator handles instantiation and teardown of the underlying drivers internally. This does not break immutability of individual drivers -- each driver instance remains unchanged; the Orchestrator simply manages which instances are active. Note that this makes the Orchestrator the one component in MCS that can be **stateful** -- a property it shares with the active-target switching pattern described in [Section 5](./5_Orchestrator.md).

Constructor parameters should be defined as **plain data objects** (dataclasses, Pydantic models, or similar). This makes them serializable and inspectable, which enables dynamic configuration loading and auto-generated UI forms.
