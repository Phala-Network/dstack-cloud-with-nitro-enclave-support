## Implementation Guide (From POC to Production)

### 1. Purpose

This guide provides an end‑to‑end view of how to deploy and operate the dstack‑based KMS solution, from a minimal proof‑of‑concept (POC) to a production‑grade deployment.

It is meant to be the first document new teams read, with deep links into the more detailed design and operations documents.

### 2. Recommended Reading Order

For a first implementation, we recommend:

1. This document (`13-implementation-guide.md`)
2. Architecture and threat model (`01-architecture-and-threat-model.md`)
3. GCP environment and CVM setup (`02-gcp-environment-and-cvm-setup-guide.md`)
4. KMS image and deployment scripts (`03-kms-image-and-deployment-scripts.md`)
5. Governance and on-chain integration design (`05-governance-and-onchain-integration-design.md`)
6. Governance deployment and operations (`06-governance-deployment-and-operations-manual.md`)
7. Nitro integration and RA-TLS (`04-nitro-integration-and-ra-tls-guide.md`)
8. Nitro CI/CD and governance integration (`07-nitro-cicd-and-governance-integration.md`)
9. Monitoring and runbook (`08-monitoring-and-alerting-guide.md`, `09-runbook-operations-upgrade-troubleshooting.md`)

### 3. POC Path (Single Region, Direct RPC)

The goal of the POC is to demonstrate:

- A KMS instance running on GCP Confidential VM (TDX)
- A Nitro enclave app performing RA-TLS to the KMS
- Key retrieval via `getKey(name)` gated by governance state

#### 3.1 Prepare GCP environment

- Follow `02-gcp-environment-and-cvm-setup-guide.md` to:
  - Create or select a GCP project
  - Configure VPC, subnets, and basic firewall rules
  - Ensure Confidential VM (TDX) support is available

For a POC, a single project and a single region are sufficient.

#### 3.2 Deploy minimal governance stack

- Follow `05-governance-and-onchain-integration-design.md` and `06-governance-deployment-and-operations-manual.md` to:
  - Deploy a governance multisig (Safe) on the chosen network
  - Optionally deploy a timelock module (for POC, a shorter delay may be used)
  - Deploy the `DstackKms` and `DstackApp` contracts
  - Transfer ownership to the governance multisig

In early POCs, timelock configuration can be simplified, but the overall pattern (multisig owning the contracts) should be preserved.

#### 3.3 Build and deploy the KMS

- Build or select a KMS image as described in `03-kms-image-and-deployment-scripts.md`:
  - For a POC, direct RPC mode is usually sufficient.
- Use `dstack-cloud` to:
  - Create a KMS project directory
  - Configure `prelaunch.sh` with:
    - `KMS_IMAGE`
    - `KMS_HTTPS_PORT` and `AUTH_HTTP_PORT`
    - `ETH_RPC_URL` pointing to the testnet RPC
    - `KMS_CONTRACT_ADDR` pointing to the `DstackKms` contract
  - Deploy the KMS CVM and complete the Bootstrap process.

#### 3.4 Deploy a reference Nitro enclave app

- Use the `dstack-nitro-enclave-app-template` repository as described in:
  - `04-nitro-integration-and-ra-tls-guide.md`
  - `07-nitro-cicd-and-governance-integration.md`
- For the POC:
  - Replace the KMS root CA certificate (`app/root_ca.pem`) with the KMS’s root CA
  - Set `KMS_URL` and `APP_ID` to point to the POC KMS
  - Build and release an EIF using the provided CI workflow

#### 3.5 Register the enclave measurement

- Read the `OS_IMAGE_HASH` from the Nitro app release.
- For a POC:
  - You may use the development path described in the Nitro template README (direct `kms:add-image` call) or
  - Preferably, create a governance transaction via the multisig to register the measurement.

#### 3.6 Run the RA-TLS demo

- Launch the Nitro enclave.
- Use the reference app to:
  - Establish RA-TLS to the KMS
  - Request a key via `getKey(name)`
- Confirm that:
  - Requests from the authorized measurement succeed
  - Requests from an unauthorized measurement fail

Use `11-e2e-test-and-preprod-validation-report.md` as a checklist for POC‑level tests (E2E-001/010, GOV-001 etc.).

### 4. Production Path (Hardened Deployment)

The production rollout path builds on the POC but with stricter controls and additional components.

#### 4.1 Harden governance

- Use `05` and `06` to:
  - Configure the production Safe with:
    - More owners across independent teams or organizations
    - A higher signature threshold
  - Enable and configure the timelock module with:
    - Production‑appropriate cooldown delays (for example, 24–72 hours)
    - Reasonable expiration times for queued transactions
  - Document the governance roles and processes (who can propose, sign, and execute).

All production changes to `DstackKms`, `DstackApp`, and related contracts should go through the Safe + timelock flow.

#### 4.2 Harden the KMS deployment

- Follow `02` and `03` to:
  - Place KMS instances in dedicated subnets with tight firewall rules
  - Remove unnecessary external IPs and restrict SSH access
  - Use a private container registry for KMS images
  - Version KMS images and maintain a rollback strategy
- Consider running:
  - Light‑client mode with `helios` to reduce RPC dependencies
  - Multiple KMS instances (for example, using Onboard) for redundancy and key replication

#### 4.3 Harden Nitro CI/CD and registration

- Use `07` and the Nitro template to:
  - Ensure Nitro CI builds and releases are reproducible and attested (Sigstore)
  - Treat `OS_IMAGE_HASH` as an input to governance proposals rather than calling KMS registration contracts directly from CI
  - Require governance approval (multisig + timelock) for all new or updated enclave measurements in production

#### 4.4 Monitoring, alerting, and runbooks

- Implement the monitoring and alerting setup described in:
  - `08-monitoring-and-alerting-guide.md`
  - `09-runbook-operations-upgrade-troubleshooting.md`
- At a minimum:
  - Monitor KMS availability, latency, and error rates
  - Monitor RA‑TLS and attestation failures
  - Monitor governance transactions, timelock queues, and failures
  - Define on‑call rotation and incident response procedures

#### 4.5 Pre-production validation and go-live

- Use `11-e2e-test-and-preprod-validation-report.md` to:
  - Run a full E2E test suite in a pre‑production environment (mirroring production as closely as possible)
  - Validate governance flows (Safe + timelock), Nitro integration, and monitoring
  - Capture results and sign‑offs from both vendor and customer teams

After successful pre‑production validation, repeat the same deployment and governance steps in the production environment, with appropriate modifications (for example, production RPC endpoints and Safe signer sets).

### 5. Summary Checklists

#### 5.1 POC checklist

- [ ] GCP project, network, and CVM support verified (`02`)
- [ ] Governance Safe and basic KMS contracts deployed (`05`, `06`)
- [ ] Single‑region KMS deployed in direct‑RPC mode (`03`)
- [ ] Reference Nitro app built and released (`04`, Nitro template)
- [ ] Enclave measurement registered (dev or governance path)
- [ ] RA‑TLS demo completed (`getKey(name)` success/failure cases)
- [ ] Basic monitoring and logs accessible

#### 5.2 Production checklist

- [ ] Production governance Safe set up with appropriate owners and thresholds (`05`, `06`)
- [ ] Timelock configured with agreed delays and expiration (`05`, `06`)
- [ ] KMS deployed with hardened network, registry, and OS image practices (`02`, `03`)
- [ ] Nitro CI/CD integrated with governance (no direct EOA registrations) (`07`)
- [ ] Monitoring, alerting, and runbooks implemented and tested (`08`, `09`)
- [ ] E2E test suite executed in pre‑production and results documented (`11`)
- [ ] Production go‑live executed following approved change management processes

