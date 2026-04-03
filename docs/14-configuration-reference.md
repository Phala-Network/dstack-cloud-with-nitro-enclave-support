## Configuration Reference

This document summarizes key configuration parameters used across the KMS, Nitro applications, governance contracts, and supporting infrastructure.

It is intended as a quick reference for operators and integrators.

### 1. KMS Runtime Configuration (.env)

These parameters are typically written by `prelaunch.sh` into the KMS project `.env` file and consumed by `docker-compose`.

| Name               | Component | Description                                                | Example                                      | Sensitive | Source          |
|--------------------|-----------|------------------------------------------------------------|----------------------------------------------|----------|-----------------|
| `KMS_IMAGE`        | KMS       | Container image used for the KMS runtime                  | `registry.example.com/dstack-kms:v1.0.0`     | No       | Config / CI     |
| `KMS_HTTPS_PORT`   | KMS       | External HTTPS port for KMS API                           | `12001`                                      | No       | Config          |
| `AUTH_HTTP_PORT`   | KMS       | Internal HTTP port for `auth-api`                         | `18000`                                      | No       | Config          |
| `ETH_RPC_URL`      | KMS       | Ethereum RPC endpoint (direct RPC mode)                   | `https://base-sepolia.example.com`           | Yes      | Secret / Config |
| `KMS_CONTRACT_ADDR`| KMS       | On-chain `DstackKms` contract address                     | `0x1234...abcd`                              | No       | Config          |
| `HELIOS_IMAGE`     | KMS       | Optional Helios light-client container image              | `registry.example.com/helios:v0.5.0`         | No       | Config          |
| `DSTACK_REPO`      | KMS       | dstack or dstack-cloud repository reference               | `https://github.com/Dstack-TEE/dstack-cloud` | No       | Config / CI     |
| `DSTACK_REF`       | KMS       | Commit or tag of dstack/dstack-cloud                      | `14963a2ccb0ec7be...`                        | No       | Config / CI     |

Additional environment variables may be introduced as the implementation evolves; they should be documented alongside the deployment scripts.

### 2. Nitro Enclave Application Configuration

These parameters are used when building and configuring Nitro enclave applications that integrate with the KMS.

| Name / File         | Component   | Description                                             | Example                                  | Sensitive | Source      |
|---------------------|------------|---------------------------------------------------------|------------------------------------------|----------|------------|
| `KMS_URL`           | Nitro app   | URL of the KMS HTTPS endpoint                           | `https://kms.example.com:12001`          | No       | Config     |
| `APP_ID`            | Nitro app   | Application identifier used for key scoping             | `0xapp...id`                             | No       | Config     |
| `app/root_ca.pem`   | Nitro app   | Root CA certificate for KMS TLS pinning                 | PEM file                                 | Yes      | Secret     |
| `OS_IMAGE_HASH`     | Nitro app   | Combined PCR hash computed from EIF measurements        | `0xabc1...def2`                          | No       | CI output  |

Changing `KMS_URL`, `APP_ID`, or `app/root_ca.pem` affects the enclave measurements and therefore the `OS_IMAGE_HASH`. These changes must be coordinated with governance to update the set of authorized measurements.

### 3. Governance and On-chain Addresses

These values are typically stored in configuration management systems and used by both on-chain deployment scripts and off-chain components.

| Name                      | Component        | Description                                      | Example        | Sensitive | Source          |
|---------------------------|------------------|--------------------------------------------------|----------------|----------|-----------------|
| `GOV_SAFE_ADDRESS`        | Governance       | Address of the governance multisig (Safe)       | `0xSafe...001` | No       | On-chain / Config |
| `GOV_TIMELOCK_ADDRESS`    | Governance       | Address of the timelock module attached to Safe | `0xTime...002` | No       | On-chain / Config |
| `DSTACK_KMS_ADDRESS`      | Governance/KMS   | `DstackKms` contract proxy address              | `0xKms...003`  | No       | On-chain / Config |
| `DSTACK_APP_ADDRESS`      | Governance/App   | `DstackApp` contract address                     | `0xApp...004`  | No       | On-chain / Config |

Governance deployment and changes to these addresses must follow the processes described in the governance documents.

### 4. Monitoring and Metrics (Examples)

The exact metric names depend on the implementation, but the following categories should be covered:

#### 4.1 KMS metrics

- Request counts and error rates:
  - `kms_requests_total{status="ok|error"}`
  - `kms_ra_tls_failures_total`
- Latency:
  - `kms_request_latency_seconds{quantile="0.5|0.95|0.99"}`

#### 4.2 Nitro and RA-TLS metrics

- Enclave lifecycle:
  - `nitro_enclave_instances_running`
  - `nitro_enclave_restarts_total`
- RA-TLS status:
  - `nitro_ra_tls_success_total`
  - `nitro_ra_tls_failure_total`

#### 4.3 Governance metrics

- Timelock queue:
  - `governance_timelock_pending_total`
  - `governance_timelock_oldest_age_seconds`
- Transaction outcomes:
  - `governance_tx_success_total`
  - `governance_tx_failure_total`

These metric names are examples and should be adapted to the actual monitoring implementation. The monitoring guide (`08-monitoring-and-alerting-guide.md`) provides more context on how to use and alert on these metrics.

