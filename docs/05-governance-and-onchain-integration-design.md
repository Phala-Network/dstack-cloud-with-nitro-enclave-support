## Governance and On-chain Integration Design

### 1. Purpose

This document defines the governance and on-chain integration design for the KMS system. It is intended for smart contract developers, security reviewers, and platform owners.

### 2. On-chain Components

Describe the key smart contracts involved, for example:

- DstackApp
- DstackKms
- Governance contracts (e.g., timelock, Safe or other multisig)

For each contract, summarize its responsibilities and how it interacts with the others.

### 3. Governance Model

Define:

- Roles (e.g., proposers, executors, guardians)
- Multisig parameters (e.g., number of signers, threshold)
- Timelock parameters (e.g., delay, grace period)

Describe how governance decisions are made, approved, and executed.

### 4. KMS and Governance Integration

Explain how the KMS:

- Reads or validates governance state from the chain
- Enforces governance decisions (e.g., which measurements or enclave identities are allowed to receive keys)
- Reacts to changes such as adding/removing key providers or updating policies

Clearly state the assumptions about chain availability and how failures are handled.

### 5. Upgrade and Change Management

Describe:

- How contract upgrades are performed (if upgradable)
- How KMS image or configuration upgrades are connected to governance (e.g., via proposals)
- Safety mechanisms to avoid lock-in or accidental bricking of the system

### 6. Security Considerations

Outline:

- Potential governance-related attack vectors (e.g., signer compromise, proposal flooding, mistaken parameter changes)
- Mitigations (e.g., timelocks, multi-stage approvals, limits)
- Monitoring and alerting hooks relevant to governance events

