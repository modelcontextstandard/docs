---
title: 10. Relation to MCP
sidebar_position: 10
---

# 10 · Relation to MCP

MCS does **not** compete with MCP directly. It generalizes the same idea of standardizing LLM external system connections without imposing a new wire protocol or stack. MCP updates in 2025 (e.g., streamable HTTP transport, OAuth Resource Servers with mandatory Resource Indicators per RFC 8707) have improved its robustness, but vulnerabilities persist. Mostly because of the new protocol stack. MCP uses it and MCS shows that this is not needed.

MCS approaches the goal from a different angle, focusing on minimalism and reuse of established standards, but the concepts are complementary. Existing MCP servers can be seamlessly integrated via MCS drivers (e.g., MCP-over-STDIO or MCP-over-SSE). The driver treats the MCP server as its **bridge** and exposes MCP's tool list as its **spec**. Conversely, MCP servers can leverage MCS drivers—especially ToolDrivers, with a single MCS-compatible MCP implementation bridging all MCS drivers into the MCP ecosystem.

This mutual compatibility allows reusing solutions without client changes, fostering an open foundation for diverse integrations.