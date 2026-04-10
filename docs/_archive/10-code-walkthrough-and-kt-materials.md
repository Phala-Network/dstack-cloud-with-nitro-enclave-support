## Code Walkthrough and Knowledge Transfer Materials

### 1. Purpose

This document serves as supporting material for code walkthrough sessions and knowledge transfer to the customer’s team. It should help new contributors and operators to understand the codebase and key integration points.

### 2. Repository Overview

Provide a high-level overview of the main repositories and components, for example:

- dstack-kms (KMS service)
- dstack-agent-nitro (Nitro agent/library)
- Smart contracts (DstackApp, DstackKms, governance)
- CI/CD configuration repositories

For each repository, briefly describe its responsibility and structure.

### 3. Core Flows

Highlight and link to the code implementing key flows, such as:

- Key request and release path (including RA-TLS handling)
- Governance state reading and enforcement in KMS
- Telemetry and logging integration

For each flow, describe the entry points, main modules, and typical extension points.

### 4. Development Practices

Document:

- Coding conventions and patterns that are important for consistency
- Testing strategy (unit, integration, and end-to-end tests)
- How to run tests locally and in CI

### 5. Extensibility and Future Work

Provide guidance on:

- How to add support for new clouds or TEE technologies
- How to extend governance logic or add new contracts
- How to add new metrics, dashboards, or runbook entries

### 6. Suggested KT Session Outline

Include a proposed agenda for live KT sessions, for example:

- Architecture and threat model (summary)
- Walkthrough of key repositories and flows
- Live demo of deployment and RA-TLS key retrieval
- Q&A and discussion of future extensions

