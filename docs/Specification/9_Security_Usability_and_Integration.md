---
title: 9. Security, Usability and Integration
sidebar_position: 9
---

# 9 · Security, Usability and Integration

MCS is intentionally **not** a security framework. Its goal is to connect LLMs to external systems directly and minimally by reusing established technologies (HTTP, OpenAPI, OAuth, Docker, ...) instead of inventing a new protocol stack.

The experience with MCP has made one thing very clear: introducing a new protocol stack is not trivial. New protocols bring new attack surfaces that take years to discover, report, and close. By early 2026, the MCP ecosystem has accumulated over 50 documented vulnerabilities -- 13 of them critical -- spanning remote code execution, cross-client data leaks, and SSRF-based credential theft [VulnerableMCP](https://vulnerablemcp.info/). Tool poisoning, where malicious instructions are hidden in tool descriptions and invisible to the user but acted on by the model, has emerged as the primary attack vector; the OWASP MCP Top 10 lists it alongside token mismanagement, privilege escalation, and supply-chain attacks [OWASP MCP Top 10](https://mcpplaygroundonline.com/blog/mcp-security-tool-poisoning-owasp-top-10-mcp-scan). An Astrix Security analysis found that 88% of MCP servers require credentials, with 53% relying on static, long-lived secrets, leading to the conclusion that *"stdio-based MCP servers break nearly every enterprise security pattern"* [Posta 2025](https://blog.christianposta.com/mcp-should-be-remote/). Academic research confirmed the breadth of the problem: across thousands of servers, 7.2% exhibit general security flaws and 5.5% show MCP-specific tool-poisoning vulnerabilities [Arxiv 2504.03767](https://arxiv.org/abs/2504.03767).

MCS avoids this entire class of protocol-level vulnerabilities by building on battle-tested standards whose security properties are well understood and whose tooling (firewalls, API gateways, OAuth flows, TLS, ...) is already deployed in production. No new wire protocol means no new protocol-specific attack surface.

That said, **running untrusted drivers is still running untrusted code**. MCS can improve trust signals -- unique driver IDs, distribution through established package registries like PyPI and npm, optional signed checksums, and a discovery index (see Section 11) -- but it cannot eliminate the fundamental risk of executing third-party software. Future work includes verifiable origin, integrity checks, and community review mechanisms for drivers (see [Section 14 -- Driver trust and verification](14_Next_Steps.md)).

## What MCS does (and does not) guarantee

MCS standardizes the **driver contract** -- not authentication, authorization, secret storage, or policy enforcement. The client remains in control of the execution loop: when to call the LLM, when to execute tools, when to stop. Security work therefore lives primarily in the client and deployment environment: access control, sandboxing, auditing, and safe secret handling.

Because MCS reuses existing transports and standards, all existing infrastructure applies directly. API gateways, rate limiting, OAuth2, API keys, mTLS, CORS rules, logging and monitoring -- none of these need to be reinvented or adapted. The LLM integration fits into the existing security architecture like any other module.

## Credentials and configuration

MCS expects driver configuration to happen via the **constructor** (see [Section 6 -- Driver construction](6_Configuration_Instantiation.md)). Credentials are typically either:

- passed explicitly to the driver constructor, or
- read by the driver from environment variables.

Where credentials are stored (keychain, vault, env injection in CI/CD, ...) and which policies apply is a client and deployment concern. MCS does not prescribe a credential management strategy.

## Credential isolation: the driver as a trust boundary

In most current agent frameworks, an LLM agent invokes external tools through shell execution, for example `exec("himalaya list")` to read e-mail via IMAP. The agent process has access to the same filesystem, environment variables, and secrets as the tool it is calling. Even container-based sandboxing does not solve this. The tool needs the credentials to function, so they must be present inside the sandbox. The agent can read them at any time -- `cat ~/.config/himalaya/.passwd` is one shell command away.

MCS eliminates this class of exposure structurally. The driver holds credentials internally, received through its constructor and configured by the operator. The agent never sees them. It calls `execute_tool("mail.list", {...})` and receives results -- nothing more. There is no shell, no filesystem path, and no environment variable the agent could inspect to extract secrets.

This has a direct consequence for **prompt injection resilience**. Even if an attacker succeeds in hijacking the agent's instructions, making it call unintended tools or pass crafted arguments, the credentials remain inaccessible. The agent can be tricked into *using* a tool, but it cannot be tricked into *leaking* the secrets behind it. The driver acts as a structural firewall that holds even in the worst case of a successful injection.

Consider the contrast with shell-based tool execution:

```
Shell model:    Agent → exec("tool ...") → shell (has credentials) → tool
                Agent → exec("cat .passwd") → credentials leaked

MCS model:      Agent → execute_tool("mail.list", ...) → Driver (has credentials) → API
                Agent has no path to credentials
```

This is not a policy or a convention, it is an architectural property of the driver contract. As long as the client does not expose its own internals to the LLM, the credentials configured in the driver constructor are unreachable from the agent's perspective.

Because MCS builds on existing standards (HTTP, OAuth, mTLS, API keys, ...), the driver inherits the security model of the underlying transport without requiring new infrastructure. An HTTP-based driver benefits from the same TLS encryption, API gateway policies, and OAuth2 token flows that protect any other API client in the organization. The credential isolation described above then adds a layer on top: even if the transport is secure, the agent still cannot see *which* credentials are being used or extract them for other purposes. The result is defense in depth -- the transport secures the wire, and the driver contract secures the boundary between agent and secrets.

## User consent (open design point)

MCS currently does not standardize a built-in user-consent mechanism ("approve this tool execution before it runs"). This is a deliberate choice: the right consent UX depends on the client (TUI, GUI, batch), the environment, and the risk profile of the tools involved.

The architecture already provides a natural seam for consent: the Orchestrator parses the LLM output, extracts the tool name and arguments, and calls `execute_tool()` as a discrete step. A consent check fits naturally between parse and execute.

The current design direction is a **consent mixin** (e.g. `SupportsConsent`). When a driver or orchestrator enables this capability, it signals pending tool calls to the client before execution -- including the tool name, arguments, and an optional timeout. The client presents the request to the user and either confirms or denies. If consent is not granted within the timeout, the tool call is aborted and the driver returns an error via `DriverResponse` (with `call_failed = true` and an appropriate `call_detail`). This keeps consent opt-in, driver-controlled, and compatible with all client types (TUI, GUI, batch). The mixin system can provide one solution if the ecosystem converges on common patterns (see [Section 14](14_Next_Steps.md) for ongoing discussion).

## Beyond LLM integration: two perspectives

Sometimes the most valuable consequence of an architecture is not what it adds but what it makes *possible* when you look at it from the other side.

**Expose a service using OpenAPI -- no SDK required.** If you publish a service over the network using an established standard like OpenAPI, then any MCS client can use it by configuration alone (pointing the driver to the OpenAPI URL). But so can every other tool that understands OpenAPI: Swagger UI, Postman, code generators, API gateways, test automation. You do not lock yourself into an MCS-only ecosystem, like MCP does. The service is useful with and without MCS.

**Reuse ToolDrivers outside of LLM clients.** A `MCSToolDriver` is deliberately LLM-agnostic. Its `list_tools()` and `execute_tool()` methods form a machine-readable capability surface and a generic dispatch mechanism -- useful for any system that needs to discover and invoke operations uniformly. The example `demo_hybrid_driver.py` in the Python SDK illustrates this duality: the same component works as an `MCSDriver` (LLM-facing) and as a `MCSToolDriver` (orchestrator- or system-facing). A Kafka consumer could dispatch events to the same ToolDriver an LLM uses; a workflow engine like n8n or Airflow could populate its node catalog from `list_tools()` and execute steps via `execute_tool()`. This is a story for another day, but the architecture would allow it.
