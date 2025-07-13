---
title: 7. Autostart Convention
sidebar_position: 7
---

# 7 · Autostart Convention

Autostart is mostly obsolete in MCS, as existing interfaces (e.g., HTTP endpoints) are used directly. For local systems, employ direct code (e.g., mcs-driver-filesystem-localfs) without spinning up extra processes.

In MCS, the autostart functionality, key to MCP's popularity, is largely unnecessary. You'd never think to start a FastAPI server for local filesystem access, yet that's common in MCP. MCS focuses on established interfaces: HTTP endpoints already run on servers, and local operations use direct code, eliminating additional processes.

This makes integration more intuitive and secure, no extra server logic to wrap existing APIs. Anything accessible via HTTP, local methods, or standards can be used directly.

**Example:** The Context7 MCP server [7](https://github.com/upstash/context7) provides access to LLM developer docs [8](https://context7.com). It's already an API, so instead of starting a local MCP server, bind it via an MCS driver. Context7 lacks an OpenAPI spec, but create one from its MCP tool description and host it on a CDN [9](https://gist.githubusercontent.com/bizrockman/7fbca8d1c3d30ef9c54db6f7190c6166/raw/4236a47e555552bea0c00e1384964a1ea0d568ae/context7_openapi_llm_friendly.json). Context7 becomes MCS-compatible without extra infrastructure.

This highlights OpenAPI's advantage: A simple alternative description suffices—extend, simplify, or LLM-optimize it. Clients swap a URL; no development, hosting, or maintenance for additional servers, especially thin wrappers.

Thus, local autostart is fully eliminated. However, if a driver needs to spin up local infrastructure, it's optional via the SupportsAutostart mixin:
```python
from abc import ABC, abstractmethod

class SupportsAutostart(ABC):
    @abstractmethod
    def autostart(self, **kwargs) -> None:
        pass
```

No concrete reference implementation exists yet, but the concept is clear: The driver defines start parameters (e.g., for a Docker container), and a framework or Orchestrator launches it automatically on demand. Users might just provide the Docker image name, but execution and binding happen in the background. Resulting in a better Plug & Play experience.

If autostart is used, it must be virtualized for safety. Driver authors should specify how systems start containers, including guidelines for container developers to ensure uniform startup. For a REST-HTTP driver with autostart, include management to launch containers, isolate ports, and pass URLs back to the driver for populating get_function_description() or list_tools().

This is safer than MCP's implicit STDIO autostart, avoiding privilege risks. Controlled, reproducible, and secure through process virtualization. Local environments need no manual config, they're orchestrated.

But really most of the time you will not need this anymore.