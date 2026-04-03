## Code Walkthrough and Knowledge Transfer Materials

### 1. Purpose

This document serves as supporting material for code walkthrough sessions and knowledge transfer to the customer’s team. It should help new contributors and operators to understand the codebase and key integration points.

### 2. Repository Overview

Provide a high-level overview of the main repositories and components, for example:

- **dstack-kms** (KMS service)
  - Implements the core KMS logic, RA-TLS handling, and integration with `DstackKms` on-chain state.
  - Typical layout: service entrypoints, RA-TLS module, on-chain client, configuration and logging modules.
- **dstack-agent-nitro** (Nitro agent/library)
  - Runs inside Nitro enclaves, establishes RA-TLS to the KMS, and exposes a simple API to enclave applications.
  - Typical layout: RA-TLS client, attestation utilities, key usage helpers.
- **Smart contracts** (DstackApp, DstackKms, governance)
  - Solidity or other on-chain code defining KMS policies, application integration, and governance (Safe + timelock).
  - Typical layout: core contracts, upgrade proxies (if used), deployment scripts, tests.
- **CI/CD configuration repositories**
  - Pipeline definitions for:
    - KMS image builds
    - Nitro EIF builds and releases
    - Governance deployment scripts

For each repository, briefly describe:

- Directory structure
- How to build and test locally
- How it is tied into the overall deployment (for example, which images or contracts it produces)

### 3. Core Flows

Highlight and link to the code implementing key flows, such as:

- **Key request and release path**
  - Entrypoint in the KMS service that handles `getKey(name)`.
  - RA-TLS verification logic (where attestation evidence is parsed and validated).
  - Policy enforcement based on on-chain state from `DstackKms`.
- **Governance state reading and enforcement**
  - Modules that read from `DstackKms` and related contracts (for example, via an on-chain client or caching layer).
  - How the KMS reacts to changes in governance state (for example, updated measurements).
- **Telemetry and logging integration**
  - Where metrics are emitted (for example, request counts, latency, failures).
  - Where structured logs are written and how they are correlated across components.

For each flow, describe:

- The main entrypoints (functions, services)
- Key modules involved in the flow
- Typical extension points (for example, how to add a new key type or telemetry field)

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

1. **Architecture and threat model (30–45 min)**
   - Use `01-architecture-and-threat-model.md` as the primary reference.
   - Focus on components, trust boundaries, and where governance and TEEs fit in.
2. **Repositories and core flows (45–60 min)**
   - Walk through the main repositories listed in this document.
   - Trace the key request path from enclave to KMS to chain and back.
3. **Deployment and governance (45–60 min)**
   - Demonstrate KMS deployment using `dstack-cloud` (from `02` and `03`).
   - Show how Safe + timelock governance changes are applied and reflected in KMS behavior (`05` and `06`).
4. **Nitro and RA-TLS demo (30–45 min)**
   - Use the Nitro template and the RA-TLS demo steps from `04`.
   - Show `OS_IMAGE_HASH` registration via governance and successful `getKey(name)` calls.
5. **Monitoring, runbooks, and Q&A (30–45 min)**
   - Review dashboards and alerts defined in `08`.
   - Walk through one or two troubleshooting scenarios from `09`.
   - Open Q&A and discussion of future extensions (for example, new clouds or TEE types).

