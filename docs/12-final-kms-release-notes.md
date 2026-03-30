## Final KMS Release Notes

### 1. Overview

These release notes describe the final KMS image(s) and associated artifacts delivered as part of the project.

### 2. Delivered Images and Artifacts

List:

- Final KMS image identifiers (e.g., image family, version tags)
- Supported cloud environments and TEE types
- Associated Nitro enclave images (where applicable)
- Relevant smart contract versions and addresses (if appropriate to document here)
- Governance stack components, including:
  - The multisig wallet (for example, Safe) that owns or controls KMS-related contracts
  - The timelock module attached to the multisig and its key parameters (e.g., default delay)

### 3. Changes Since Earlier Versions

Summarize major changes compared to earlier images or prototypes, such as:

- New features or capabilities
- Security or hardening improvements
- Performance enhancements
- Breaking changes or deprecations

### 4. Compatibility and Requirements

Document:

- Minimum supported environment versions (e.g., GCP machine types, OS versions, TEE capabilities)
- Required contract or governance versions
- Any migration steps required when upgrading from previous versions

For governance, explicitly state:

- Dependencies on the chosen multisig and timelock implementations
- Any assumptions about available tooling (for example, Safe web interface support on the target network)

### 5. Known Issues

List:

- Known bugs or edge cases
- Limitations that users should be aware of
- Workarounds or mitigations where available

### 6. Upgrade and Rollback Guidance

Provide:

- High-level guidance on upgrading to this final image from earlier versions
- Rollback considerations and constraints

Reference the operations runbook for detailed procedures.
