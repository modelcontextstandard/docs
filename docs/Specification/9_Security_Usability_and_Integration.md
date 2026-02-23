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

## User consent (open design point)

MCS currently does not standardize a built-in user-consent mechanism ("approve this tool execution before it runs"). This is a deliberate choice: the right consent UX depends on the client (TUI, GUI, batch), the environment, and the risk profile of the tools involved.

The architecture already provides a natural seam for consent: the Orchestrator parses the LLM output, extracts the tool name and arguments, and calls `execute_tool()` as a discrete step. A consent check fits naturally between parse and execute. Standalone drivers that use `process_llm_response()` have no such seam today, but the capability/mixin system can provide one if the ecosystem converges on common patterns (see [Section 14](14_Next_Steps.md) for ongoing discussion).

## Beyond LLM integration: two perspectives

Sometimes the most valuable consequence of an architecture is not what it adds but what it makes *possible* when you look at it from the other side.

**Expose a service using OpenAPI -- no SDK required.** If you publish a service over the network using an established standard like OpenAPI, then any MCS client can use it by configuration alone (pointing the driver to the OpenAPI URL). But so can every other tool that understands OpenAPI: Swagger UI, Postman, code generators, API gateways, test automation. You do not lock yourself into an MCS-only ecosystem, like MCP does. The service is useful with and without MCS.

**Reuse ToolDrivers outside of LLM clients.** A `MCSToolDriver` is deliberately LLM-agnostic. Its `list_tools()` and `execute_tool()` methods form a machine-readable capability surface and a generic dispatch mechanism -- useful for any system that needs to discover and invoke operations uniformly. The example `demo_hybrid_driver.py` in the Python SDK illustrates this duality: the same component works as an `MCSDriver` (LLM-facing) and as a `MCSToolDriver` (orchestrator- or system-facing). A Kafka consumer could dispatch events to the same ToolDriver an LLM uses; a workflow engine like n8n or Airflow could populate its node catalog from `list_tools()` and execute steps via `execute_tool()`. This is a story for another day, but the architecture would allow it.
