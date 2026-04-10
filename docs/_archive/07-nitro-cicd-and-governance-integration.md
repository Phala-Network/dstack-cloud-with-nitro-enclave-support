## Nitro CI/CD and Governance Integration

### 1. Purpose

This document describes the CI/CD pipelines used to build and deploy Nitro Enclave applications and how those pipelines integrate with the governance system.

It builds on the reference Nitro application and CI/CD template provided in the `dstack-nitro-enclave-app-template` repository, which can be used as a baseline implementation and adapted to the customer’s environment and governance model.

### 2. Build Pipeline (CI)

The reference build pipeline (CI) should cover at least the following steps:

- **Build the enclave application**
  - Compile the application code and its dependencies for the Nitro target.
  - Ensure reproducible builds where possible (for example, pinned toolchain versions).

- **Produce the enclave image**
  - Build an EIF (enclave image file) from the application and runtime.
  - Optionally sign the image or record content hashes for integrity tracking.

- **Compute measurements and metadata**
  - Extract PCR values or equivalent measurements from the EIF.
  - Compute an `OS_IMAGE_HASH` or similar aggregate identifier for use in on-chain registration.

- **Publish artifacts**
  - Store the EIF in a registry or artifact store.
  - Publish measurement metadata (including `OS_IMAGE_HASH`) as part of a release (for example, a GitHub release), together with any provenance attestations.

The `dstack-nitro-enclave-app-template` repository contains a GitHub Actions workflow that performs these steps and creates a release containing the image metadata and measurements. That workflow can be used as a reference implementation when designing the customer’s CI setup.

### 3. Deploy Pipeline (CD)

Describe:

- How enclave workloads are deployed to the target environment
- How configuration (including KMS endpoint and governance-related parameters) is applied
- How rollbacks are handled in case of issues

### 4. Governance Integration

Explain how CI/CD interacts with governance, for example:

- Automatically generating proposal payloads when a new enclave image is built (for example, transactions to update allowed enclave measurements or configuration in governance contracts)
- Submitting or assisting with proposals to update allowed measurements or configuration, targeting the governance multisig (for example, a Safe), rather than interacting with contracts directly from CI/CD identities
- Ensuring that deployment only proceeds once governance approval is in place and, where applicable, once the timelock delay has elapsed and the transaction has been executed

The key integration point with governance is the measurement metadata produced by CI (for example, `OS_IMAGE_HASH`). Instead of having CI call governance contracts directly:

- The CI pipeline should:
  - Produce a structured artifact (for example, a JSON file or on-chain transaction payload) that encodes the proposed update to allowed measurements.
  - Attach this artifact to the release or store it in a location accessible to governance operators.
- Governance operators (or an automated bot with tightly controlled permissions) should:
  - Use this artifact to create a transaction targeting the governance multisig (Safe).
  - Let the change pass through the timelock and standard governance flow before it becomes active.

Any direct on-chain registration commands shown in reference templates (for example, CLI tasks that call `kms:add-image` from an EOA) should be treated as development examples only. In production, registration of new enclave images MUST go through the governance path defined in the governance documents.

### 5. Security and Auditability

Document:

- How build and deploy pipelines are secured (e.g., access to secrets, restricted runners)
- What logs and audit trails are produced (e.g., build logs, proposal IDs, transaction hashes)
- How to trace a running enclave workload back to its source code and governance approvals
