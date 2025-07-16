---
title: 1.1 Fundamental Principles
sidebar_position: 1
---

### **1.1 · Fundamental Principles**

**The Core Reality of LLM Integration**

Before diving into MCS specifics, it's crucial to understand the fundamental constraints that shape any LLM-to-system integration.

1. **Token-Based Interface**: The only interface an LLM has to its environment is token-based input/output (simplified as text throughout this document). LLMs cannot directly execute code, access APIs, or interact with systems. They can only consume and produce text.

2. **Natural Language Programming**: LLMs are systems that you program with natural language. Unlike traditional programming where you write deterministic code, with LLMs you craft prompts that guide their behavior probabilistically.

3. **Function Calling Reality**: Function calling is fundamentally a prompting technique. You include function definitions in the prompt to "program" the LLM to output text in a specific, parseable format. This is not a native capability but an emergent behavior from careful prompt engineering and LLM Training/Fine-Tuning.

4. **The Parser/Bridge**: Between the LLM's text output and actual function execution lies a critical component: the parser. This parser must:
   - Detect when the LLM intends to call a function
   - Extract the function name and parameters from free text
   - Handle ambiguity and formatting variations
   - Bridge to the actual execution environment

5. **Reliability Factors**: The success of this entire chain depends on:
   - **The Model**: Different models have varying capabilities in following instructions and producing structured output
   - **The Prompt**: How well the function descriptions and formatting instructions are crafted for these models.
   - **The Parser**: How robust it is in handling edge cases
   
   **Not on**: The protocol, framework, or tooling used. These are merely implementation details.

6. **The Driver Analogy**: If you think of this bridge in terms of protocol and transport, it's analogous to a driver in an operating system. It abstracts the complexity of hardware (external systems) from the kernel (LLM), providing a standardized interface.

**This is what MCS standardizes**: Not the protocol or transport (those already exist), but the contract for these "drivers" that bridge LLMs to external systems. MCS acknowledges that the quality of integration depends primarily on the model and prompt quality, not on inventing new protocols, tools or frameworks.