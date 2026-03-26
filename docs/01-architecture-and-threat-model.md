## Architecture and Threat Model

### 1. Purpose and Scope

This document describes the overall architecture and threat model for the dstack-based KMS deployment. It is intended for architects, security reviewers, and senior engineers who need to understand:

- The main components and how they interact
- The trust boundaries and attack surfaces
- The assumptions and guarantees provided by the system

It does not aim to be a formal security proof, but it should provide enough structure to support audits and internal security reviews.

### 2. High-level System Overview

At a high level, the system consists of:

- A KMS service built on top of dstack and deployed on GCP Confidential VM (TDX/SGX)
- Nitro Enclave workloads that request keys using RA-TLS
- On-chain smart contracts (DstackApp, DstackKms, and governance contracts)
- Supporting infrastructure such as RPC endpoints, monitoring, and CI/CD pipelines

The KMS exposes a limited API surface (for example, `getKey(name)`) and enforces access control using attestation, measurement-based authorization, and on-chain governance policies.

### 3. Components and Data Flows

This section should describe, in more detail:

- GCP environment: projects, networks, confidential VMs running dstack-kms
- dstack-kms: the KMS process, key storage, attestation verification logic
- Nitro Enclave: enclave application, `dstack-agent-nitro`, and RA-TLS client logic
- On-chain contracts: DstackApp, DstackKms, governance contracts, and Safe (or other multisig)
- External dependencies: L1/L2 RPC providers, time sources, credential stores

For each major interaction (e.g., key request, governance change, image upgrade), an end-to-end sequence diagram should be defined here or linked from separate diagrams.

### 4. Trust Boundaries and Assets

This section should identify:

- Critical assets: root keys, tenant keys, configuration state, governance control
- Trust anchors: TEEs, on-chain contracts, multisig signers, and governance rules
- Trust boundaries: between cloud provider and workload, between host and enclave, between off-chain systems and on-chain governance

It should clearly state what is assumed to be trusted, partially trusted, or untrusted, and why.

### 5. Threat Model

This section outlines the main attacker types and threat scenarios, for example:

- Malicious or curious cloud provider operators
- Compromised host OS or hypervisor
- Malicious or compromised Nitro Enclave workload
- Compromised RPC providers or network attackers
- Misconfigured or malicious governance participants

For each threat, the document should describe:

- The impact if the threat materializes
- The mitigations (e.g., attestation, isolation, governance checks, monitoring)
- Any residual risks or assumptions

### 6. Security Invariants and Guarantees

This section summarizes the key security properties that the system aims to guarantee, such as:

- Keys are only released to workloads that match an authorized measurement and governance policy
- Governance changes follow multisig + timelock rules and are auditable
- Host compromise alone does not allow key exfiltration

Where possible, these invariants should be linked back to specific mechanisms in the design.

### 7. Open Questions and Future Work

Finally, this section should track open security questions, areas for future hardening, and dependencies on external audits or formal verification work.

