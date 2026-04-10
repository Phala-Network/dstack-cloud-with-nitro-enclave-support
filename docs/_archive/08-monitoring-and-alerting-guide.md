## Monitoring and Alerting Guide (Datadog + Incident.io)

### 1. Purpose

This document describes the monitoring and alerting setup for the KMS stack using Datadog and Incident.io. It is intended for SREs and on-call engineers.

This guide assumes the governance design and operational procedures described in:

- `05-governance-and-onchain-integration-design.md`
- `06-governance-deployment-and-operations-manual.md`

### 2. Observability Scope

Clarify which components are covered:

- KMS instances on GCP confidential VMs
- Nitro Enclave workloads
- On-chain interactions and governance events
- Supporting infrastructure (RPC, databases, CI/CD, etc.)
- Governance and multisig contracts (for example, the Safe and timelock module that control KMS-related contracts and policies)

### 3. Metrics and Dashboards

Define and implement metrics and dashboards in the following areas:

- **KMS service**
  - Availability and latency of key APIs (for example, `getKey(name)`), including:
    - Request rate
    - p50/p95/p99 latency
    - Error rates (by error type)
  - Attestation-related metrics (for example, RA-TLS failures, invalid measurements, authorization denials).

- **Nitro Enclave workloads**
  - Enclave instance lifecycle (start/stop/restart counts)
  - Health checks and heartbeat signals from enclave workloads
  - RA-TLS client-side failures or timeouts (as observed from enclave logs/metrics)

- **On-chain and governance**
  - Governance and multisig metrics, such as:
    - Number of pending governance transactions in the timelock queue
    - Time-to-execution for critical governance actions
    - Failure rates for governance-related transactions
  - RPC health and latency toward the networks used by governance and KMS contracts.

Define Datadog dashboards grouped by audience, for example:

- **KMS Overview Dashboard**: high-level health (availability, error rates, key request latency).
- **Enclave & RA-TLS Dashboard**: enclave lifecycle, RA-TLS success/failure, attestation metrics.
- **Governance & On-chain Dashboard**: Safe / timelock activity, pending transactions, RPC health.

### 4. Logs and Traces

Document:

- What logs are collected from each component
- How logs are structured and tagged
- How traces (if used) connect calls across KMS, enclaves, and external services

At a minimum:

- KMS logs should include:
  - Request identifiers and correlation IDs
  - Outcome of attestation and authorization checks
  - References to on-chain state (for example, which measurement entry was used)
- Enclave logs should include:
  - RA-TLS handshake outcomes
  - Key usage events (at an appropriate level of abstraction, without leaking sensitive material)
- Governance logs should capture:
  - Multisig and timelock events (for example, new proposal submitted, approvals collected, queued, executed, cancelled)
  - These can often be ingested from on-chain events into Datadog or a similar logging backend.

### 5. Alerting and Incident Management

Describe:

- Alert rules (conditions, thresholds, and severities)
- Integration with Incident.io (how alerts become incidents, routing, and escalation)
- On-call schedules and response expectations

Include alerts related to governance and multisig, such as:

- Critical governance actions stuck in the timelock queue beyond an expected window
- Unexpected or frequent changes to timelock parameters (e.g., cooldown shortened below agreed thresholds)
- Repeated failures of governance transactions or unusually high governance activity that may indicate misconfiguration or abuse

### 6. Operational Procedures

Provide guidance on:

- How to use dashboards and alerts during incident triage
- How to suppress or tune noisy alerts
- How to review monitoring and alerting configuration periodically

For governance-related alerts, operators should follow the runbooks described in:

- `06-governance-deployment-and-operations-manual.md` (governance operations and error handling)
- `09-runbook-operations-upgrade-troubleshooting.md` (system-wide troubleshooting and emergency procedures)
