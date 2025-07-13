---
title: 5. Configuration & Instantiation
sidebar_position: 5
---

# 5 · Configuration & Instantiation

Drivers are configured via constructors, e.g., URLs for specs, auth tokens, proxies. Use libraries like Pydantic for validation.

This decouples configuration from functionality, enabling dynamic setups from env vars or JSON. The motivation is to allow drivers to be fully 
automatically loaded and configured in the future, without requiring the client to implement special functions for particular tools.

Ideally, driver-specific settings can be handled through a configuration file or dependency injection, keeping the client agnostic.

With tools like Pydantic, the client could even dynamically generate configuration interfaces for the user without needing to know the details 
(though this can be challenging, as auto-generated forms have their own complexities).

This mirrors traditional drivers that bring their own configuration interfaces, without involving the operating system.
