## Nitro Enclave Integration and RA-TLS Guide

### 1. Purpose

This document explains how Nitro Enclave workloads integrate with the KMS using RA-TLS and the `dstack-agent-nitro` library. It is aimed at application developers and platform engineers responsible for enclave workloads.

### 2. Overview of RA-TLS in This Project

Provide a concise overview of:

- How RA-TLS is used to establish mutual TLS (mTLS) between the enclave and KMS
- How attestation evidence and measurements are embedded into the TLS handshake
- How the KMS validates attestation and decides whether to release keys

### 3. dstack-agent-nitro

Describe:

- The role of `dstack-agent-nitro` in enclave applications
- How it is initialized and configured
- How it manages keys, certificates, and attestation evidence

Include a minimal usage example or pseudocode for an enclave application integrating with the agent.

### 4. getKey(name) API

Document:

- The semantics of the `getKey(name)` API
- Input parameters, return values, and error conditions
- How measurement-based authorization is enforced (e.g., which measurements are allowed and where they are configured)

This section should also outline typical error scenarios (e.g., measurement mismatch, lack of governance approval, attestation failures) and their expected responses.

### 5. End-to-End Flow

Explain, step by step, the end-to-end flow for:

- Bootstrapping an enclave application
- Establishing RA-TLS with the KMS
- Requesting a key with `getKey(name)`
- Handling errors and retries

Sequence diagrams are recommended to illustrate the interaction between enclave, agent, KMS, and any on-chain components involved in authorization.

### 6. Demo Scenario

Define a simple demo scenario that:

- Uses a reference Nitro Enclave application
- Establishes RA-TLS with the KMS
- Successfully calls `getKey(name)` and uses the key (for example, to sign or decrypt data)

The demo should be reproducible on a developer or test environment and serve as the basis for the "Verified RA-TLS demo" deliverable.

