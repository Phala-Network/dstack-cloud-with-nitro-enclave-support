## Documentation Plan

This document summarizes the documentation set required for the dstack-based KMS deployment, integration, and operations project. It is intended to give a single overview of what exists, what each document is for, and how they relate.

### 1. Architecture & Threat Model

**File:** `docs/01-architecture-and-threat-model.md`  
**Purpose:** Describe the overall system architecture, trust boundaries, and threat model for the KMS solution built on top of dstack, GCP confidential VMs, Nitro Enclaves, and on-chain governance.

### 2. GCP Environment & CVM Setup Guide

**File:** `docs/02-gcp-environment-and-cvm-setup-guide.md`  
**Purpose:** Provide a step-by-step guide for infrastructure/SRE teams to prepare the GCP project, network, IAM, and confidential VM (TDX/SGX) environment required by the KMS system.

### 3. KMS Image & Deployment Scripts

**File:** `docs/03-kms-image-and-deployment-scripts.md`  
**Purpose:** Explain how to build, version, publish, and deploy the GCP-compatible dstack-kms image, and how to use the accompanying deployment scripts or infrastructure-as-code modules.

### 4. Nitro Integration & RA-TLS Guide

**File:** `docs/04-nitro-integration-and-ra-tls-guide.md`  
**Purpose:** Describe how Nitro Enclave workloads integrate with the KMS using RA-TLS, including how the `dstack-agent-nitro` library is used and how the `getKey(name)` API works end to end.

### 5. Governance & On-chain Integration Design

**File:** `docs/05-governance-and-onchain-integration-design.md`  
**Purpose:** Define the governance and on-chain integration design, including DstackApp and DstackKms smart contract extensions, multisig + timelock patterns, and security considerations.

### 6. Governance Deployment & Operations Manual

**File:** `docs/06-governance-deployment-and-operations-manual.md`  
**Purpose:** Provide a practical manual for deploying and operating the governance smart contracts across testnet and mainnet, and executing typical governance actions safely.

### 7. Nitro CI/CD & Governance Integration

**File:** `docs/07-nitro-cicd-and-governance-integration.md`  
**Purpose:** Document the CI/CD pipelines that build and deploy Nitro Enclave applications and how they integrate with on-chain governance (e.g., proposal creation and execution flows).

### 8. Monitoring & Alerting Guide

**File:** `docs/08-monitoring-and-alerting-guide.md`  
**Purpose:** Describe the monitoring and alerting setup using Datadog and Incident.io, including key metrics, logs, dashboards, and escalation policies.

### 9. Operations Runbook

**File:** `docs/09-runbook-operations-upgrade-troubleshooting.md`  
**Purpose:** Provide clear runbooks for day-to-day operations, upgrades, and common troubleshooting scenarios for the KMS, Nitro workloads, and governance stack.

### 10. Code Walkthrough & Knowledge Transfer Materials

**File:** `docs/10-code-walkthrough-and-kt-materials.md`  
**Purpose:** Capture the structure of key repositories and components, core request paths, and guidance for future contributors; also serve as supporting material for code walkthrough and KT sessions.

### 11. E2E Test & Pre-prod Validation Report

**File:** `docs/11-e2e-test-and-preprod-validation-report.md`  
**Purpose:** Summarize the end-to-end test coverage, pre-production validation results, and known limitations or accepted risks, forming the formal test report.

### 12. Final KMS Release Notes

**File:** `docs/12-final-kms-release-notes.md`  
**Purpose:** Document the final KMS image(s), supported environments, major changes compared to earlier images, and known issues, serving as the final deliverable release notes.

