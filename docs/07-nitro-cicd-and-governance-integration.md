## Nitro CI/CD and Governance Integration

### 1. Purpose

This document describes the CI/CD pipelines used to build and deploy Nitro Enclave applications and how those pipelines integrate with the governance system.

### 2. Build Pipeline (CI)

Document:

- How enclave application code is built
- How enclave images are produced and signed
- How measurements (e.g., PCRs or hashes) are computed and recorded

Describe how these steps are automated in CI and what artifacts are generated.

### 3. Deploy Pipeline (CD)

Describe:

- How enclave workloads are deployed to the target environment
- How configuration (including KMS endpoint and governance-related parameters) is applied
- How rollbacks are handled in case of issues

### 4. Governance Integration

Explain how CI/CD interacts with governance, for example:

- Automatically generating proposal payloads when a new enclave image is built
- Submitting or assisting with proposals to update allowed measurements or configuration
- Ensuring that deployment only proceeds once governance approval is in place

### 5. Security and Auditability

Document:

- How build and deploy pipelines are secured (e.g., access to secrets, restricted runners)
- What logs and audit trails are produced (e.g., build logs, proposal IDs, transaction hashes)
- How to trace a running enclave workload back to its source code and governance approvals

