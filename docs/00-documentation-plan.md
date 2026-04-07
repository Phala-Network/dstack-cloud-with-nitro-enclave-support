# Documentation Plan

This document defines the documentation structure for the `dstack-cloud` project. It targets developers of all levels and covers deployment on both **GCP Confidential VMs** and **AWS Nitro Enclaves**. All content is based on the dstack open-source framework and does not depend on Phala Cloud.

---

## Directory Structure

```
docs/
├── 00-documentation-plan.md              ← This file
│
├── concepts/                              ← Concept Guides
│   ├── overview.md                        ← dstack-cloud overview and core concepts
│   ├── kms-and-key-delivery.md            ← KMS and key delivery (GCP/Nitro adaptation)
│   ├── nitro-enclave.md                   ← AWS Nitro Enclave integration
│   ├── attestation-integration.md         ← Attestation integration (GCP TDX / Nitro NSM)
│   ├── governance.md                      ← On-chain governance model
│   └── security-model.md                  ← Threat model and security guarantees
│
├── how-to/                                ← How-to Guides
│   ├── run-dstack-on-gcp.md               ← Run a dstack CVM on GCP
│   ├── run-dstack-on-nitro.md             ← Run a dstack CVM on AWS Nitro
│   ├── run-dstack-kms-on-gcp.md           ← Run a dstack-kms CVM on GCP
│   ├── run-dstack-kms-on-nitro.md         ← Run a dstack-kms CVM on AWS Nitro
│   ├── deploy-onchain-kms.md              ← Deploy on-chain KMS smart contracts
│   ├── register-enclave-measurement.md    ← Register Enclave measurements on-chain
│   └── manage-governance.md               ← Governance operations (propose, approve, execute)
│
├── tutorials/                             ← Tutorials
│   └── get-started.md                     ← Quick Start: Deploy your first dstack CVM on AWS Nitro
│
├── operations/                            ← Operations Manual
│   ├── monitoring-alerting.md             ← Monitoring and alerting
│   ├── upgrade.md                         ← Upgrade procedures (KMS image, Enclave app, contracts)
│   └── runbook.md                         ← Troubleshooting runbook
│
├── reference/                             ← Reference
│   ├── api-reference.md                   ← API reference (getKey, etc.)
│   ├── configuration.md                   ← Configuration reference (app.json, docker-compose.yaml, etc.)
│   └── glossary.md                        ← Glossary
│
└── appendix/                              ← Appendix (internal deliverables)
    ├── e2e-test-report.md                 ← E2E test report
    ├── release-notes.md                   ← Release notes
    └── code-walkthrough.md                ← Code walkthrough and KT materials
```

---

## Section Details

### I. Concepts

**Audience:** All developers, especially those new to dstack.
**Goal:** Explain "what" and "why". No step-by-step instructions.
**Principle:** Focus on concepts users need to understand; avoid implementation details.

#### `concepts/overview.md` — dstack-cloud Overview

- **What is dstack SDK:** An open-source confidential computing framework, originally designed to run on TDX BareMetal only
- **What is dstack-cloud:** An extension of dstack SDK that enables deployment on GCP Confidential VMs and AWS Nitro Enclaves
- **Core value proposition:** Native Docker Compose deployment + hardware-level privacy protection, no application code changes required
- **How it works:**
  - `dstack-cloud deploy` reads your `docker-compose.yaml` and packages everything into a **dstack CVM** (Confidential Virtual Machine)
  - A dstack CVM runs a minimal OS (dstack-os) with your Docker containers inside
  - On GCP, the dstack CVM runs as a Confidential VM (Intel TDX)
  - On AWS, the dstack CVM runs as a Nitro Enclave
  - Inside each CVM, a **Guest Agent** handles attestation, key retrieval, and storage encryption
  - Your application containers communicate with the Guest Agent via `/var/run/dstack.sock` using the dstack SDK (Python / TypeScript / Rust / Go)
- **Typical use cases:**
  - AI inference with model weight protection
  - Sensitive data processing (healthcare, finance)
  - DeFi and Web3 applications (keys never leave TEE, on-chain verifiable execution environment)
- **System architecture diagram** (high-level, covering both GCP and Nitro topologies)
- **Supported environments:** GCP Confidential VMs (TDX) / AWS Nitro Enclaves (NSM)
- Link to Getting Started tutorial

#### `concepts/kms-and-key-delivery.md` — KMS and Key Delivery

> For KMS core design principles (Decentralized Root-of-Trust, MPC, key derivation protocol, etc.), refer to the official Phala documentation:
> - [Key Management Protocol](https://docs.phala.com/dstack/design-documents/key-management-protocol)
> - [Decentralized Root-of-Trust](https://docs.phala.com/dstack/design-documents/decentralized-root-of-trust)
>
> This document focuses on how KMS delivers keys in GCP and Nitro environments, and the differences between KMS operating modes.

- **Two KMS operating modes:**
  - **Local Key Provider mode:** Keys are generated and managed locally within the TEE. No separate KMS service needed. Suitable for development, testing, and single-node deployments.
  - **KMS mode:** A standalone KMS service runs in its own TEE (a dedicated dstack CVM). It verifies workload identity before dispatching keys. Suitable for production and multi-node deployments.
- **On-chain smart contracts as a security enhancement:**
  - KMS mode can optionally integrate with on-chain smart contracts (DstackApp, DstackKms)
  - On-chain contracts control which workloads are authorized to receive keys (via registered measurements)
  - All authorization changes are observable on-chain, preventing covert key policy modifications
  - This is a security enhancement to KMS mode, not a separate operating mode
- **Key delivery flow** (KMS mode, high-level):
  1. A dstack CVM starts and obtains a hardware attestation
  2. The CVM requests keys from KMS over a secure channel (RA-TLS)
  3. KMS verifies the attestation authenticity
  4. If on-chain contracts are integrated, KMS checks whether the workload is authorized on-chain
  5. Verification passed — KMS dispatches keys
- **GCP vs. Nitro differences at a glance:**

  | Aspect | GCP (TDX) | AWS Nitro Enclave |
  |--------|-----------|-------------------|
  | Attestation method | TDX Quote + TPM | Nitro Attestation Document |
  | Communication with KMS | Direct network | Via VSOCK proxy |
  | Measurement identifier | RTMR0-3 | PCR / OS_IMAGE_HASH |

- **Key lifecycle overview** (see official docs for details): generation → derivation → rotation → revocation

#### `concepts/nitro-enclave.md` — AWS Nitro Enclave Integration

- **What is Nitro Enclave:**
  - An isolated compute environment provided by AWS, fully isolated from the host (EC2 instance)
  - Memory-encrypted — the host OS cannot read Enclave memory
  - No persistent storage — every launch starts from a clean state
- **How dstack uses Nitro Enclaves:**
  - `dstack-cloud deploy` packages your Docker Compose application into a dstack CVM
  - On AWS Nitro, this dstack CVM runs as a Nitro Enclave
  - Your Docker containers run inside the dstack CVM, not directly in the Enclave
  - The Guest Agent inside the CVM handles attestation and key management
- **Networking and communication:**
  - The Enclave (dstack CVM) cannot directly access the external network
  - Communication happens via VSOCK to the host machine; a proxy on the host forwards network requests
  - This means accessing KMS, RPC, and other external services requires a VSOCK proxy
- **Enclave Image (EIF):**
  - Built from the dstack OS Docker image
  - Measurements (PCR) are generated at build time — any code or configuration change produces a different PCR
  - `OS_IMAGE_HASH`: unique identifier used for on-chain authorization
- **Resource constraints:**
  - CPU and memory are statically allocated at launch and cannot be changed during runtime
  - Maximum available resources depend on the host EC2 instance type
- **Key differences from GCP Confidential VMs:**

  | Aspect | GCP Confidential VM | AWS Nitro Enclave |
  |--------|---------------------|-------------------|
  | Isolation | VM-level isolation | Process-level isolation |
  | Network access | Direct virtual network access | Via VSOCK proxy |
  | Persistence | Supports disk storage | No persistent storage |
  | Resource allocation | Elastic | Static at launch |

#### `concepts/attestation-integration.md` — Attestation Integration

> For the fundamentals of dstack Remote Attestation, RA-TLS protocol design, ZT-TLS and ACME certificate management, refer to the official Phala documentation:
> - [TEE Attestation Guide](https://docs.phala.com/dstack/trust-center-technical)
> - [TEE-Controlled Domain Certificates](https://docs.phala.com/dstack/design-documents/tee-controlled-domain-certificates)

This document focuses on **how dstack-cloud integrates attestation mechanisms on different hardware platforms**.

- **What is Remote Attestation:**
  - The TEE generates a cryptographic proof that it is genuinely running in a hardware-isolated environment
  - The proof contains workload measurements, reflecting code and configuration integrity
  - The verifier checks the proof and measurements to confirm the workload has not been tampered with
- **Attestation on GCP:**
  - When a dstack CVM starts, the hardware automatically records measurements across the entire boot chain (firmware → kernel → user space)
  - The Guest Agent obtains a TDX Quote containing these measurements and a hardware signature
  - KMS verifies the Quote signature to confirm the CVM is running in a genuine TDX environment
- **Attestation on Nitro:**
  - When a dstack CVM (Nitro Enclave) starts, NSM automatically generates an Attestation Document
  - The Document contains PCR values and an NSM signature
  - KMS verifies the Document to confirm the Enclave is running in a genuine Nitro environment
- **Measurements and on-chain authorization:**
  - A workload's measurements must be registered on-chain before KMS will dispatch keys to it
  - This ensures only audited code versions can obtain keys
  - Updated code = new measurements = requires re-registration

#### `concepts/governance.md` — On-chain Governance Model

- **Why on-chain governance:**
  - Enforces "code is law": once deployed, application logic cannot be silently altered
  - All key authorization changes must be initiated on-chain, ensuring observability
  - Prevents "covert deployer attacks" (malicious deployers injecting code via invisible off-chain upgrades)
- **Core contracts:**
  - **`DstackKms`:** KMS policy contract — stores authorized workload measurements, admin roles, and configuration parameters
  - **`DstackApp`:** Application entry contract — holds a reference to DstackKms and enforces application-level checks
- **Governance workflow:**
  1. Draft a transaction (e.g., register new measurements, upgrade contracts, modify configuration)
  2. Submit to a Multisig (e.g., Safe) for approval
  3. Once the signature threshold is met, the transaction enters the Timelock queue
  4. After the Timelock delay expires, the transaction becomes executable
  5. Execution result is recorded on-chain; KMS syncs the latest state
- **Role of Multisig + Timelock:**
  - Multisig: prevents unilateral actions — requires multi-party approval
  - Timelock: enforces a mandatory delay, giving the community time to review and respond
- **Typical governance scenarios:**
  - Register new workload measurements (deploy a new application version)
  - Revoke a compromised measurement (emergency response)
  - Upgrade smart contracts
  - Adjust governance parameters
- **Security considerations:**
  - Signers should use hardware wallets
  - Set reasonable signature thresholds (≥ 2/3 recommended for production)
  - Conduct periodic governance health checks

#### `concepts/security-model.md` — Threat Model and Security Guarantees

- **Trust boundaries:**
  - Cloud platform (GCP / AWS): assumed untrusted
  - Host machine (EC2 instance): assumed untrusted
  - CVM / Enclave: protected by hardware-level isolation
  - KMS: runs in its own TEE, protected by hardware
  - On-chain contracts: protected by blockchain consensus
  - Multisig signers: partially trusted — impact limited by threshold and Timelock
- **Threat types:**
  - Malicious or curious cloud platform operators
  - Compromised host OS or hypervisor
  - Malicious CVM / Enclave workloads
  - Compromised RPC providers or network attackers
  - Compromised or colluding Multisig signers
- **Security guarantees:**
  - Keys are generated and dispatched only within verified TEEs — operators cannot access them
  - Governance actions follow Multisig + Timelock rules — fully auditable
  - Hardware-level isolation prevents the host from reading CVM / Enclave memory
  - Measurement changes require on-chain governance — preventing covert upgrades
- **Residual risks:**
  - TEE hardware microarchitectural side-channel attacks
  - Smart contract logic vulnerabilities

---

### II. How-to Guides

**Audience:** Developers and operators who need to complete a specific task.
**Goal:** Answer "how to do X". Each document covers one clear task.
**Principle:** Focus on steps. Do not explain implementation internals — link to Concepts when users need background.

#### `how-to/run-dstack-on-gcp.md` — Run a dstack CVM on GCP

- Prerequisites (GCP project, Service Account, Confidential VM zone)
- Install dstack-cloud CLI
- Create a project (`dstack-cloud new`)
- Configure docker-compose.yaml
- Deploy the CVM (`dstack-cloud deploy`)
- Verify the CVM is running
- Common issues

#### `how-to/run-dstack-on-nitro.md` — Run a dstack CVM on AWS Nitro

- Prerequisites (AWS account, Nitro-capable EC2 instance type, IAM configuration)
- Install dstack-cloud CLI
- Create a project (`dstack-cloud new`)
- Configure docker-compose.yaml
- Deploy the CVM as a Nitro Enclave (`dstack-cloud deploy`)
- Verify the CVM is running
- Common issues

#### `how-to/run-dstack-kms-on-gcp.md` — Run a dstack-kms CVM on GCP

- Prerequisites
- Create a KMS project (`dstack-cloud new`, specify TPM as key provider)
- Configure the KMS docker-compose.yaml
- Deploy the KMS CVM (`dstack-cloud deploy`)
- Complete KMS Bootstrap (first-time initialization)
- Verify KMS is running
- Common issues

#### `how-to/run-dstack-kms-on-nitro.md` — Run a dstack-kms CVM on AWS Nitro

- Prerequisites
- Create a KMS project (`dstack-cloud new`)
- Configure the KMS docker-compose.yaml (including VSOCK proxy)
- Deploy the KMS CVM as a Nitro Enclave (`dstack-cloud deploy`)
- Complete KMS Bootstrap (first-time initialization)
- Verify KMS is running
- Common issues

#### `how-to/deploy-onchain-kms.md` — Deploy On-chain KMS Smart Contracts

- Prerequisites (Hardhat, wallet, testnet/mainnet gas)
- Deploy DstackKms contract
- Deploy DstackApp contract
- Configure GovernanceSafe and Timelock
- Verify contract deployment
- Testnet vs. mainnet differences

#### `how-to/register-enclave-measurement.md` — Register Workload Measurements On-chain

- What are workload measurements (OS_IMAGE_HASH / PCR)
- Build the CVM image and obtain measurements
- Submit a governance proposal to register measurements
- Approval process (Multisig + Timelock)
- Verify registration succeeded

#### `how-to/manage-governance.md` — Governance Operations

- Create a governance proposal
- Multisig signing and approval
- Timelock wait and execution
- Emergency operations (revoke measurement authorization, replace signers)
- Governance health check

---

### III. Tutorials

**Audience:** New developers who want an end-to-end walkthrough.
**Goal:** Build confidence through a concrete scenario.

#### `tutorials/get-started.md` — Quick Start: Deploy Your First dstack CVM on AWS Nitro

- **Scenario:** Use dstack-cloud to deploy a Docker Compose application as a dstack CVM running in an AWS Nitro Enclave
- **Estimated time:** 30-60 minutes
- **Prerequisites:** AWS account, dstack-cloud CLI
- **Steps:**
  1. Install dstack-cloud CLI
  2. Create a project (`dstack-cloud new my-app`)
  3. Edit docker-compose.yaml to define your application
  4. Deploy the dstack CVM to AWS Nitro (`dstack-cloud deploy`)
  5. Check the status (`dstack-cloud status`)
  6. Access your application
- **Next steps:**
  - Set up KMS for key protection → [Run a dstack-kms CVM on AWS Nitro](../how-to/run-dstack-kms-on-nitro.md)
  - Register measurements for on-chain authorization → [Register Workload Measurements](../how-to/register-enclave-measurement.md)
  - Learn the underlying concepts → [Concept Guides](../concepts/overview.md)

---

### IV. Operations Manual

**Audience:** SRE, platform engineers, security operations.
**Goal:** Ensure system stability and fast incident recovery.

#### `operations/monitoring-alerting.md` — Monitoring and Alerting

- Key metrics (KMS response latency, key request success rate, attestation verification status, etc.)
- Log collection and analysis
- Dashboard configuration (Datadog)
- Alert rules and escalation policies (Incident.io)

#### `operations/upgrade.md` — Upgrade Procedures

- KMS image upgrade
- CVM / Enclave application upgrade (including governance flow for measurement changes)
- Smart contract upgrade (Proxy pattern)
- Pre-upgrade checklist and rollback plan

#### `operations/runbook.md` — Troubleshooting Runbook

- RA-TLS connection failures
- Attestation verification failures
- CVM / Enclave startup failures
- On-chain authorization failures
- KMS unavailable
- Governance transactions stuck
- VSOCK proxy failures (Nitro-specific)
- Emergency operations playbook

---

### V. Reference

**Audience:** All users who need to look up specific details.
**Goal:** Provide fast, accurate query access.

#### `reference/api-reference.md` — API Reference

- getKey(name) — full signature, parameters, return values, error codes
- Onboard API — Bootstrap, Finish
- Management API (if applicable)
- Authentication and authorization notes

#### `reference/configuration.md` — Configuration Reference

- app.json field descriptions
- dstack-specific fields in docker-compose.yaml
- Environment variables
- KMS configuration options

#### `reference/glossary.md` — Glossary

Core terminology definitions:
- **Infrastructure:** TEE, CVM, Intel TDX, SGX, AWS Nitro Enclave, NSM, VSOCK, EIF, TPM, PCCS
- **Security mechanisms:** Remote Attestation, RA-TLS, ZT-TLS, Measurement (PCR / RTMR), OS_IMAGE_HASH, Quote, Attestation Document
- **dstack components:** dstack SDK, dstack-cloud, Guest Agent, KMS (DeRoT), Gateway, VMM
- **On-chain governance:** DstackApp, DstackKms, Multisig (Safe), Timelock, GovernanceSafe
- **Cryptography:** MPC, KDF, SealingKey, RootKey

---

### VI. Appendix

**Audience:** Internal team. Project deliverable archive.
**Goal:** Not part of the official external documentation, but retained for internal reference.

#### `appendix/e2e-test-report.md` — E2E Test Report

- Test coverage
- Pre-production validation results
- Known limitations and accepted risks

#### `appendix/release-notes.md` — Release Notes

- Version history and changelog
- Compatibility notes
- Known issues

#### `appendix/code-walkthrough.md` — Code Walkthrough and KT Materials

- Key repository structure
- Core request paths
- Attestation module code walkthrough
- Guide for new contributors

---

## Design Principles

1. **Task-oriented:** Organized by "what users want to do", not "what components the system has"
2. **Document type separation:** Concept / How-to / Tutorial / Reference are strictly distinct
3. **Progressive disclosure:** Beginners start with Getting Started, then explore Concepts and How-to as needed
4. **Dual environment support:** GCP Confidential VMs and AWS Nitro Enclaves covered in parallel
5. **No Phala Cloud dependency:** All content based on the dstack open-source framework
6. **Concepts link to official docs:** Deep topics like KMS design, RA-TLS protocol point to Phala official documentation; this repo focuses on GCP/Nitro environment adaptation
7. **How-to focuses on steps:** How-to guides explain "how to do it" without diving into implementation internals — link to Concepts when background is needed
8. **Glossary-first:** Glossary is prominently placed under Reference; Concept documents link to Glossary on first use of a term
9. **Cross-referencing:** Documents link to each other to avoid information silos

## Mapping from Previous Structure

| Previous Document | New Location | Notes |
|-------------------|--------------|-------|
| 01-architecture-and-threat-model.md | concepts/overview.md + concepts/security-model.md | Split into overview and security model |
| 02-gcp-environment-and-cvm-setup-guide.md | how-to/run-dstack-on-gcp.md + how-to/run-dstack-kms-on-gcp.md | Split into dstack CVM and dstack-kms CVM |
| 03-kms-image-and-deployment-scripts.md | how-to/run-dstack-kms-on-gcp.md + how-to/run-dstack-kms-on-nitro.md | Split by environment |
| 04-nitro-integration-and-ra-tls-guide.md | concepts/nitro-enclave.md + concepts/attestation-integration.md + how-to/register-enclave-measurement.md | Split into concepts and operations |
| 05-governance-and-onchain-integration-design.md | concepts/governance.md | Core content retained |
| 06-governance-deployment-and-operations-manual.md | how-to/deploy-onchain-kms.md + how-to/manage-governance.md | Split into deployment and operations |
| 07-nitro-cicd-and-governance-integration.md | operations/upgrade.md + how-to/register-enclave-measurement.md | Merged into upgrade and registration |
| 08-monitoring-and-alerting-guide.md | operations/monitoring-alerting.md | Retained |
| 09-runbook-operations-upgrade-troubleshooting.md | operations/runbook.md + operations/upgrade.md | Split into troubleshooting and upgrade |
| 10-code-walkthrough-and-kt-materials.md | appendix/code-walkthrough.md | Moved to appendix |
| 11-e2e-test-and-preprod-validation-report.md | appendix/e2e-test-report.md | Moved to appendix |
| 12-final-kms-release-notes.md | appendix/release-notes.md | Moved to appendix |
