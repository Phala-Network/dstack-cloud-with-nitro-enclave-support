## Operations Runbook (Deployment, Upgrade, Troubleshooting)

### 1. Purpose

This runbook provides step-by-step procedures for operating the KMS system, including deployments, upgrades, and common troubleshooting scenarios.

### 2. Day-to-day Operations

Routine tasks include:

- Provisioning new environments (staging, pre-production, production) using the procedures in the deployment guide
- Rotating keys or updating configuration via governance-controlled changes and KMS redeployments
- Verifying daily/weekly health of the system using dashboards, logs, and spot checks

### 3. Deployment Procedures

Summarize the standard deployment procedure, referencing:

- KMS image and deployment scripts
- Nitro enclave CI/CD pipelines
- Governance updates when required

Include pre-deployment checks and post-deployment validation steps.

### 4. Upgrade Procedures

Document:

- How to upgrade the KMS image safely
- How to roll out changes to Nitro workloads
- How to coordinate upgrades with governance (e.g., new measurements, new parameters)

Define clear rollback plans and criteria for aborting an upgrade.

### 5. Troubleshooting Common Issues

This section provides example troubleshooting flows for common issues. Each flow outlines:

- How to detect the issue
- Where to look (logs, dashboards, metrics)
- Possible root causes and remediation steps

#### 5.1 RA-TLS failures or attestation errors

- **Symptoms**
  - Enclave applications log RA-TLS handshake failures.
  - KMS logs show attestation validation errors or unauthorized measurement errors.
  - Metrics such as `kms_ra_tls_failures_total` increase.

- **Checks**
  1. Verify that the enclave is using the correct `KMS_URL`, `APP_ID`, and `root_ca.pem`.
  2. Confirm that the `OS_IMAGE_HASH` for the running enclave image is correctly computed and matches what is expected.
  3. Check on-chain `DstackKms` state to ensure the measurement is registered and enabled.
  4. Verify that the KMS instance is using the correct `KMS_CONTRACT_ADDR` and connected to the intended chain.

- **Root causes and remediation**
  - Measurement not registered or disabled:
    - Register or re-enable via governance, wait for timelock, then retry.
  - Mismatched root CA or KMS URL:
    - Update enclave configuration, rebuild and redeploy the enclave image, then retry.
  - Chain or RPC issues:
    - Investigate and restore RPC connectivity, then retry.

#### 5.2 Enclave workloads failing to start or connect

- **Symptoms**
  - Nitro hosts report failure to start the enclave.
  - Enclave instances start but fail to reach the KMS (connection timeouts).

- **Checks**
  1. Inspect Nitro host logs and the enclave template’s logs for startup errors.
  2. Verify that the KMS HTTPS port is reachable from the Nitro hosts (firewall, routing).
  3. Confirm that the KMS instance is running and healthy.

- **Root causes and remediation**
  - Misconfigured Nitro host or EIF:
    - Correct configuration, rebuild EIF if needed, and restart the enclave.
  - Network/firewall misconfiguration:
    - Update firewall rules to allow required traffic between Nitro hosts and KMS, then test connectivity.

#### 5.3 On-chain authorization failures

- **Symptoms**
  - KMS logs indicate that authorization checks failed for `getKey(name)` despite RA-TLS succeeding.
  - Enclave receives authorization errors rather than attestation errors.

- **Checks**
  1. Inspect the `DstackKms` contract state to verify:
     - The enclave measurement is mapped to the expected key or application.
     - The key is not disabled.
  2. Confirm that the correct `DstackKms` contract address is configured in the KMS.

- **Root causes and remediation**
  - Missing or incorrect on-chain configuration:
    - Create and execute a governance proposal to correct the state.
  - Using the wrong contract or network:
    - Update KMS configuration to point to the correct `DstackKms` address and network, then redeploy.

#### 5.4 Governance and multisig issues

- **Transactions stuck in timelock or multisig queue**
  - **Symptoms**
    - Governance dashboards show pending transactions that are not executed.
    - Changes expected by operators (for example, new measurement registration) do not take effect.
  - **Checks**
    1. Inspect the Safe UI and timelock queue to see pending transactions and their earliest execution times.
    2. Verify that the required number of signatures has been collected.
  - **Remediation**
    - Collect missing signatures.
    - After cooldown, execute the transaction.
    - If the transaction is mistaken, follow the cancellation/skip procedures defined in the governance manual and submit a corrected transaction.

- **Failed governance executions**
  - **Symptoms**
    - On-chain governance transactions revert or run out of gas.
  - **Checks**
    1. Review transaction failure reasons and logs.
    2. Validate parameters and target addresses in the proposed transaction.
  - **Remediation**
    - Correct parameters and resubmit through the normal governance flow.
    - Adjust gas limits as necessary.

### 6. Emergency Procedures

Define emergency playbooks for:

- Rapidly disabling access for a compromised workload or measurement
- Failing over between environments or regions (if applicable)
- Recovering from data corruption or configuration mistakes

For governance-specific emergencies, describe:

- How to quickly block or revoke authorization for a compromised enclave measurement through the multisig and timelock
- How to coordinate an emergency change of multisig owners or thresholds while preserving control over KMS-related contracts

### 7. Checklists

Include concise checklists for:

- Pre-release and pre-change reviews
- Post-change validation
- Quarterly or periodic operational reviews
