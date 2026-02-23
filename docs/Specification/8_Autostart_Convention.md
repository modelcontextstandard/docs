---
title: 8. Autostart Convention
sidebar_position: 8
---

# 8 · Autostart Convention

MCS uses established interfaces directly -- HTTP endpoints that already run on servers, local libraries called in-process. There is no need to start an extra server just to talk to a tool. You would never spin up a FastAPI server to access the local filesystem, yet that is exactly what MCP does with its STDIO-based servers. MCS eliminates that layer entirely.

**Example -- Context7:** The Context7 MCP server ([GitHub](https://github.com/upstash/context7)) provides access to LLM developer documentation via [context7.com](https://context7.com). It is already an HTTP API. Instead of starting a local MCP proxy, an MCS driver can bind to it directly. Context7 does not publish an OpenAPI spec, but one can be [derived from its MCP tool descriptions](https://gist.githubusercontent.com/bizrockman/7fbca8d1c3d30ef9c54db6f7190c6166/raw/4236a47e555552bea0c00e1384964a1ea0d568ae/context7_openapi_llm_friendly.json) and hosted on a CDN. The client swaps a URL -- no additional server to develop, host, or maintain.

This illustrates a broader point: when the underlying service already speaks an established standard (HTTP, OpenAPI, gRPC, ...), MCS simply connects to it. The autostart problem disappears.

### When autostart is still needed

In rare cases a driver may need to spin up local infrastructure -- for example, launching a Docker container that provides a database or a sandboxed runtime. For this, MCS offers an optional mixin:

```python
from abc import ABC, abstractmethod

class SupportsAutostart(ABC):
    @abstractmethod
    def autostart(self, **kwargs) -> None:
        pass
```

No reference implementation exists yet, but the intent is straightforward: the driver declares what it needs (e.g. a Docker image name), and the client or Orchestrator launches it on demand, isolates ports, and passes the resulting URL back to the driver for use in `list_tools()` or `get_function_description()`.

Two constraints apply:

1. **Virtualization is mandatory.** Autostarted processes must run inside containers or sandboxes -- never as bare host processes. This avoids the privilege escalation risks inherent in MCP's implicit STDIO autostart.
2. **Uniform startup conventions.** Driver authors should document how their containers are started so that any client can handle them the same way.

The result is controlled, reproducible, and secure. But for the vast majority of drivers this section is irrelevant -- they connect to something that already exists.