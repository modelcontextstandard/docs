---
title: 11. Versioning & Compatibility
sidebar_position: 11
---

# 11 · Versioning & Compatibility

MCS follows Semantic Versioning (`MAJOR.MINOR.PATCH`) for the specification document itself, ensuring clear evolution while maintaining backward compatibility where possible.

- **MAJOR**: Breaking changes (e.g., interface redesigns) that require updates to drivers or clients.
- **MINOR**: Additive features (e.g., new optional capabilities) or deprecations. Deprecations always trigger a MINOR bump and include detailed migration notes to guide transitions.
- **PATCH**: Bug fixes or clarifications with no functional impact.

Versions in the 0.x range (as with the current Draft) are intended as proof-of-work and represent alpha status, not yet production-ready. They focus on exploration, feedback, and refinement, with potential for significant changes before 1.0.

Drivers and SDKs also adhere to Semantic Versioning. However, drivers do not need to explicitly declare the supported spec version in code. SDKs handle version translation and compatibility. Drivers are managed via SDK packages (e.g., installed/updated through Pip for Python), so their version aligns with the SDK they depend on. This simplifies maintenance: Update the SDK, and compatible drivers follow automatically, without manual declarations or conflicts.