---
title: 1. MCS Motivation Overview
sidebar_position: 1
---

# 1 · MCS Motivation Overview

LLMs are token predictors. They consume and produce sequences of tokens that are rendered as text via a tokenizer. This means text is the only interface an LLM has to interact with its environment. It cannot execute anything directly.

To enable execution, you need a parser. The parser takes the LLM’s output, identifies structured instructions, and translates them into real-world actions.

> **Function Calling Recap**
> 
> Function calling enables LLMs to interact with external systems by generating structured outputs (typically JSON) that describe function invocations.
>
> The process involves:
> 1. Providing the LLM with function schemas/descriptions 
> 2. The LLM generates a structured call in response to user queries
> 3. A parser extracts and validates the function call
> 4. The system executes the call against the target API/service
> 5. Results are returned to continue the conversation
>
> This pattern allows LLMs to act as intelligent orchestrators using text alone. The exact format of the function description doesn't matter. What matters is that the LLM understands when and how to call a tool and how to format the output for the parser.

This concept was formalized in [TALM: Tool Augmented Language Models](https://arxiv.org/abs/2205.12255), which showed how LLMs can be extended with non-differentiable tools to solve real-world tasks.

Modern AI frameworks often provide parsers, but not standardized descriptions or callable functions. This leads to a fragmented landscape of custom tooling where developers repeatedly reinvent the same logic for each application.

The Model Context Protocol (MCP) addressed this by introducing the first open standard to connect LLMs with external systems. OpenAI followed a similar idea with "Actions" for Custom GPTs long before that, but never published it as a general concept.

However, MCP added a full protocol stack, along with new complexity and security implications. As of 2025, recent updates to MCP include OAuth Resource Servers, mandatory Resource Indicators (RFC 8707), and streamable HTTP as a new transport mechanism (released March 2025). Despite these advancements, critiques highlight ongoing issues like prompt injection weaknesses (reported May 2025) and vulnerabilities such as CVE-2025-49596 (RCE in MCP Inspector, June 2025). Much of the effort that followed focused on building wrappers around APIs that could already be used directly by LLMs, as demonstrated in the MCS proof of concept.

Critically, MCP often reimplements features the web has solved decades ago. Take authentication: instead of relying on proven standards like Basic Auth, OAuth2, or API Keys over HTTPS, MCP introduces its own way, all while using JSON-RPC under the hood. This adds layers of complexity with little gain.

Despite this, MCP succeeded, not because of elegance, but because it is the first real standard in this space. And a standard was needed.

Useful features like autostarting MCP servers were not design decisions, but emerged from practical needs when using the STDIO transport layer. Making a core benefit for some developers a side effect not a core of MCP itself.

MCS now distills this all down to the essentials. What is actually required to connect an LLM to external systems in a standardized and reusable way.

At the core, it's a driver problem.

A MCS driver must expose a function specification that LLMs can consume. Most modern LLMs can use these out of the box. But to ensure precision, the driver should also provide usage instructions and formatting hints so the output can be correctly parsed.

The parser is the other half of the equation. It bridges the LLM’s output to real-world execution by scanning for and dispatching structured calls.

Previously, function implementations were written from scratch for every use case. But with MCS generalization is key. If a REST call works for one service, it can be reused for all REST-over-HTTP services.

An MCS Driver does exactly that. It generalizes function calling for a given protocol over a specific transport layer.