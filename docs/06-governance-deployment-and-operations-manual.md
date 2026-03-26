## Governance Deployment and Operations Manual

### 1. Purpose

This manual is meant for operators and governance participants who deploy and manage the on-chain governance stack for the KMS system.

### 2. Prerequisites

Document:

- Required tools (e.g., wallet software, CLI tools, RPC endpoints)
- Supported networks (testnet, pre-production, mainnet)
- Access requirements and security best practices for operators

### 3. Deployment Procedures

Provide step-by-step procedures for:

- Deploying governance contracts (DstackApp, DstackKms, multisig, timelock)
- Initializing governance state (e.g., assigning roles, configuring parameters)
- Verifying that the deployment matches expectations (addresses, parameters, ownership)

### 4. Routine Governance Operations

Describe how to:

- Create, review, and approve governance proposals
- Execute proposals once timelocks and approvals are satisfied
- Perform typical operations such as:
  - Adding/removing authorized enclave measurements
  - Rotating signers or updating thresholds
  - Updating KMS configuration parameters stored on-chain

### 5. Handling Errors and Exceptional Events

Define procedures for:

- Failed transactions (e.g., out of gas, reverts)
- Misconfigured parameters detected after deployment
- Emergency actions (e.g., pausing a feature, temporarily blocking a measurement)

### 6. Operational Checklists

Include checklists for:

- Pre-deployment checks
- Post-deployment verification
- Periodic governance health reviews (e.g., signer key hygiene, contract ownership status)

