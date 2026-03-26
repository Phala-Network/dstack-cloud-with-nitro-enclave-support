## Monitoring and Alerting Guide (Datadog + Incident.io)

### 1. Purpose

This document describes the monitoring and alerting setup for the KMS stack using Datadog and Incident.io. It is intended for SREs and on-call engineers.

### 2. Observability Scope

Clarify which components are covered:

- KMS instances on GCP confidential VMs
- Nitro Enclave workloads
- On-chain interactions and governance events
- Supporting infrastructure (RPC, databases, CI/CD, etc.)

### 3. Metrics and Dashboards

Define:

- Key metrics for availability, latency, error rates, and capacity
- KMS-specific metrics (e.g., key request rates, attestation failures)
- Enclave-specific metrics (e.g., start/stop events, health checks)

Describe the main Datadog dashboards, their intended audience, and how to interpret them.

### 4. Logs and Traces

Document:

- What logs are collected from each component
- How logs are structured and tagged
- How traces (if used) connect calls across KMS, enclaves, and external services

### 5. Alerting and Incident Management

Describe:

- Alert rules (conditions, thresholds, and severities)
- Integration with Incident.io (how alerts become incidents, routing, and escalation)
- On-call schedules and response expectations

### 6. Operational Procedures

Provide guidance on:

- How to use dashboards and alerts during incident triage
- How to suppress or tune noisy alerts
- How to review monitoring and alerting configuration periodically

