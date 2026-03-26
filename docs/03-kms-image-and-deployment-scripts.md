## KMS Image and Deployment Scripts

### 1. Purpose

This document describes how to build, manage, and deploy the GCP-compatible KMS image and the associated deployment scripts or infrastructure-as-code definitions.

It is intended for platform engineers and CI/CD owners.

### 2. Image Build Process

Describe:

- The source repositories and components that go into the dstack-kms image
- The build tools used (e.g., Docker, Packer, custom scripts)
- Build-time configuration options (e.g., environment, network endpoints, feature flags)

Include a high-level build pipeline outline that can be implemented in CI.

### 3. Versioning and Publishing

Document:

- Image naming convention (including environment, version, and TEE type)
- How images are published to GCP (e.g., custom images, image families, or container registries)
- Retention and cleanup policy for old images

### 4. Deployment Scripts and Infra Modules

Describe:

- The structure of deployment scripts (e.g., shell scripts, Terraform modules, Helm charts)
- Required input parameters (e.g., project ID, network, image version, governance contract addresses)
  - This should explicitly include governance-related parameters such as:
    - The admin Safe (multisig) address that owns or controls the KMS contracts
    - The timelock module address attached to the Safe, if applicable
    - Any other governance module addresses that the KMS needs to recognize
- Typical deployment topologies (single-region, multi-region, staging vs. production)

Where applicable, this section should include examples of command lines or configuration snippets, but keep secrets and environment-specific details out of the repository.

### 5. Configuration Management

Document:

- How runtime configuration is passed to KMS instances (e.g., environment variables, config files, secret stores)
- How sensitive configuration and keys are handled
- How configuration changes are rolled out safely (e.g., through CI/CD pipelines)

### 6. Deployment Procedures

Provide step-by-step procedures for:

- Deploying a new environment (e.g., staging, pre-production, production)
- Upgrading the KMS image in an existing environment
- Rolling back to a previous image in case of issues

New environment deployment procedures should assume that the governance Safe and timelock module have already been deployed and configured according to the governance deployment manual, and that their addresses are available as inputs to the deployment scripts.

Each procedure should include checks and roll-back criteria.
