---
layout: default
title: CSP & MDM Deep Dive
---

# CSP & MDM Deep Dive

How Intune configuration reaches a Windows endpoint: the management tree that addresses a setting, the Configuration Service Providers that own its branches, and the OMA-DM session that carries configuration down and status back.

- **[CSP Anatomy](/CSP-MDM-Deep-Dive/csp-anatomy.html)** — what a CSP physically is. How an OMA-URI resolves to code, where that registration lives, where the schema lives, and which stores hold state.
- **[Intune ↔ CSP flow](/CSP-MDM-Deep-Dive/intune-csp-flow.html)** — interactive. A sync session end to end, the SyncML envelope a single command travels in, and how a compliance Get becomes an access decision in Entra.
- **[OMA-URI tree](/CSP-MDM-Deep-Dive/oma-uri-tree.html)** — interactive. The common namespaces, and how a context root, the vendor namespace, and a CSP compose into a path.
