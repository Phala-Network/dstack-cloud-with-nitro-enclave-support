# dstack-cloud Documentation

Documentation for deploying dstack-compatible confidential workloads on GCP (Intel TDX) and AWS Nitro Enclaves with dstack.

## Quick Start

**New to dstack?** Start here:

- **[Quick Start Tutorial](docs/tutorials/get-started.md)** — Deploy your first confidential workload on GCP in 10-15 minutes

## Documentation Structure

### Tutorials

| Document | Description |
|----------|-------------|
| [Quick Start](docs/tutorials/get-started.md) | Deploy your first dstack app on GCP |

### Concepts

| Document | Description |
|----------|-------------|
| [Overview](docs/concepts/overview.md) | Architecture, components, and how dstack works |
| [Security Model](docs/concepts/security-model.md) | Trust boundaries and security guarantees |
| [Attestation Integration](docs/concepts/attestation-integration.md) | TDX + vTPM and NSM attestation mechanisms |
| [KMS and Key Delivery](docs/concepts/kms-and-key-delivery.md) | How keys are delivered to confidential workloads |
| [Nitro Enclave](docs/concepts/nitro-enclave.md) | AWS Nitro Enclave specifics and VSOCK communication |
| [Governance](docs/concepts/governance.md) | On-chain governance with Safe and Timelock |

### How-to Guides

| Document | Description |
|----------|-------------|
| [Run a Workload on GCP](docs/how-to/run-dstack-on-gcp.md) | Deploy Docker apps as CVMs on GCP with Intel TDX |
| [Run a Workload on AWS Nitro](docs/how-to/run-dstack-on-nitro.md) | Deploy Docker apps as Nitro Enclaves |
| [Run dstack-kms on GCP](docs/how-to/run-dstack-kms-on-gcp.md) | Set up your own KMS instance |
| [Register Enclave Measurement](docs/how-to/register-enclave-measurement.md) | Whitelist workloads for key retrieval |
| [Deploy On-chain KMS](docs/how-to/deploy-onchain-kms.md) | Deploy KMS contract with Timelock governance |
| [Manage Governance](docs/how-to/manage-governance.md) | Operate Safe and Timelock for production |

### Operations

| Document | Description |
|----------|-------------|
| [Monitoring & Alerting](docs/operations/monitoring-alerting.md) | Observability setup |
| [Runbook](docs/operations/runbook.md) | Operational procedures |
| [Upgrade](docs/operations/upgrade.md) | Upgrade procedures |

### Reference

| Document | Description |
|----------|-------------|
| [API Reference](docs/reference/api-reference.md) | KMS and dstack-util APIs |
| [Configuration](docs/reference/configuration.md) | Configuration options |
| [Glossary](docs/reference/glossary.md) | Terms and definitions |

### Appendix

| Document | Description |
|----------|-------------|
| [Code Walkthrough](docs/appendix/code-walkthrough.md) | Source code explanations |
| [E2E Test Report](docs/appendix/e2e-test-report.md) | End-to-end testing results |
| [Release Notes](docs/appendix/release-notes.md) | Version history and changes |

## Key Features

- **Confidential Computing** — Run workloads in hardware-protected TEEs (Intel TDX on GCP, Nitro Enclaves on AWS)
- **Remote Attestation** — Prove your workload runs in genuine hardware
- **Key Management** — Secure key delivery from KMS running in its own TEE
- **On-chain Governance** — Production-grade governance with Safe multisig and Timelock

## Supported Platforms

| Platform | TEE Technology | Key Delivery |
|----------|---------------|--------------|
| **GCP** | Intel TDX | dstack-agent (automatic) |
| **AWS** | Nitro Enclave | dstack-util via VSOCK Proxy |

## KMS Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **Phala Official KMS** | Hosted by Phala Network | Quick start, development |
| **Self-hosted KMS** | Deploy your own | Production, compliance |

Self-hosted KMS can be deployed on:
- GCP (Intel TDX CVM)
- Intel TDX Bare Metal server

## Related Resources

- [dstack-cloud GitHub](https://github.com/Phala-Network/dstack-cloud) — Main repository
- [dstack-nitro-enclave-app-template](https://github.com/Phala-Network/dstack-nitro-enclave-app-template) — Nitro Enclave template
- [dstack GitHub](https://github.com/Dstack-TEE/dstack) — Core dstack and KMS contracts

## License

MIT
