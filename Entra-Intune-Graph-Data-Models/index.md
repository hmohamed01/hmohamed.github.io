---
layout: default
title: Entra & Intune Graph Data Models
---

# Entra & Intune Graph Data Models

Entra ID and Intune are two separate data stores that share a URL namespace and an access token, and nothing else. Every difficulty in cross-model automation follows from that.

Microsoft Graph stores nothing. It is a façade over unrelated backing services, in the same way the SMS Provider is a façade over the ConfigMgr site database.

- **[Entra Schema Blueprint](/Entra-Intune-Graph-Data-Models/entra-schema-blueprint.html)** — interactive. How Entra's types inherit from `entity`, built from Graph's live v1.0 schema: select a type to trace its inheritance path back to the root and list its properties.

## Sections

| Section | Covers |
|---|---|
| Foundations | What Graph actually is, and the OData/EDM type system both models share |
| Entra data model | The directory model: `entity` → `directoryObject` → concrete types |
| Intune data model | Device management: singletons, policy polymorphism, query idioms |
| Where the models touch | Correlation keys, the assignment seam, what the platform does not guarantee |
| Practice | Query support, permissions, and replacing query-based collections |
| Reference | ConfigMgr crosswalk, gotchas, glossary |
