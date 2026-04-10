# Run a dstack-kms CVM on GCP

This guide explains how to deploy a dstack-kms (Key Management Service) instance on GCP as a Confidential VM with Intel TDX.

## Prerequisites

- A GCP project with Confidential VM (TDX) quota
- `gcloud` CLI installed and authenticated
- `dstack-cloud` CLI installed
- **For on-chain governance mode:**
  - A blockchain RPC endpoint (e.g., `https://sepolia.base.org`)
  - Deployed smart contracts (DstackKms, DstackApp) — see [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md)
  - You'll need the contract addresses (`KMS_CONTRACT_ADDR` and `APP_CONTRACT_ADDR`) before configuring the KMS

> **Note:** This guide uses pre-built KMS Docker images. If you need to build KMS from source or customize the configuration, see the [official KMS build guide](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/tutorials/kms-build-configuration.md).

> **Development Mode:** If you just want to test KMS without on-chain governance, you can use the Local Key Provider mode (default). In this case, you don't need blockchain contracts. However, production deployments should use on-chain governance for proper access control.

## Step 1: Configure dstack-cloud

```bash
dstack-cloud config-edit
```

```toml
[gcp]
project = "YOUR_PROJECT_ID"
zone = "us-central1-a"
bucket = "gs://YOUR_BUCKET_NAME"
```

## Step 2: Pull the OS Image

```bash
dstack-cloud pull --os-image dstack-cloud-0.6.0
```

## Step 3: Create a KMS Project

```bash
dstack-cloud new kms-prod \
  --os-image dstack-cloud-0.6.0 \
  --key-provider tpm \
  --instance-name dstack-kms

cd kms-prod
```

Key flags:
- `--key-provider tpm` — Use the TPM on the GCP Confidential VM as the root of trust for key generation
- `--instance-name` — A human-readable name for the KMS instance

> **Note:** GCP deployment uses `dstack-cloud` CLI which simplifies the deployment process compared to manual CVM deployment on TDX hosts. The CLI handles VM provisioning, OS image booting, and network configuration automatically.

## Step 4: Configure the KMS Docker Compose

The project directory contains a `docker-compose.yaml` template for the KMS.

> **Key Environment Variables:**
> - `KMS_HTTPS_PORT` — Port for KMS HTTPS service (default: 12001)
> - `ETH_RPC_URL` — Ethereum RPC endpoint for blockchain access (required for on-chain mode)
> - `KMS_CONTRACT_ADDR` — Address of the DstackKms smart contract (required for on-chain mode)
> - `APP_CONTRACT_ADDR` — Address of the DstackApp smart contract (required for on-chain mode)

> **Important:** If you plan to use on-chain governance, you must deploy the smart contracts first and obtain their addresses. See [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md).

Choose the connection mode that fits your setup:

### Option A: Direct RPC Connection

Edit `docker-compose.yaml`:

```yaml
services:
  dstack-kms:
    image: phalanetwork/dstack-kms:latest
    environment:
      - KMS_HTTPS_PORT=12001
      - ETH_RPC_URL=https://sepolia.base.org
      - KMS_CONTRACT_ADDR=0xYOUR_CONTRACT_ADDRESS
      - APP_CONTRACT_ADDR=0xYOUR_APP_CONTRACT_ADDRESS
    ports:
      - "12001:12001"
```

### Option B: Light Client Connection

If you prefer the KMS to run its own light client (helios) for blockchain access:

```yaml
services:
  dstack-kms:
    image: phalanetwork/dstack-kms:latest
    environment:
      - KMS_HTTPS_PORT=12001
      - USE_LIGHT_CLIENT=true
      - KMS_CONTRACT_ADDR=0xYOUR_CONTRACT_ADDRESS
      - APP_CONTRACT_ADDR=0xYOUR_APP_CONTRACT_ADDRESS
    ports:
      - "12001:12001"
```

## Step 5: Configure Environment Variables

Edit the `.env` file or use the `prelaunch.sh` script to inject environment variables:

```bash
#!/bin/bash
# prelaunch.sh
export KMS_HTTPS_PORT=12001
export ETH_RPC_URL=https://sepolia.base.org
export KMS_CONTRACT_ADDR=0xYOUR_CONTRACT_ADDRESS
export APP_CONTRACT_ADDR=0xYOUR_APP_CONTRACT_ADDRESS
```

## Step 6: Deploy the KMS CVM

```bash
dstack-cloud deploy
```

Wait for the CVM to start. First deployment takes 5-10 minutes.

## Step 7: Open Firewall

Allow access to the KMS port:

```bash
dstack-cloud fw allow 12001
```

This opens the KMS HTTPS port for external access.

> **Note:** 
> - The auth-eth service runs on an internal port (9200) and does not need to be exposed externally
> - If using Light Client mode (Option B), no additional ports are required — the light client only makes outbound connections to Ethereum peers
> - For production deployments, consider restricting access to specific IP ranges using GCP VPC firewall rules

## Step 8: Bootstrap (First-time Only)

When the KMS starts for the first time, it enters **Onboard mode** (HTTP, unauthenticated). You must initialize it before it can serve key requests.

### 8.1 Get the KMS Domain/IP

```bash
dstack-cloud status
```

Note the domain or IP address shown in the output (e.g., `https://kms-xxx.example.com` or `http://<IP>:12001`).

### 8.2 Register KMS Measurements (Required for On-chain Mode)

> **Critical:** If using on-chain governance mode, you **must** register the KMS measurements **before** running Bootstrap. Otherwise, Bootstrap will fail because `auth-eth` cannot verify the KMS.

1. Get the KMS attestation info (including `mrAggregated`):

```bash
KMS_URL="http://<KMS_DOMAIN>:12001"

curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  "$KMS_URL/prpc/Onboard.GetAttestationInfo?json" | jq .
```

2. Register the `mrAggregated` to the `DstackKms` contract:

```bash
# Using cast (foundry)
cast send <DSTACK_KMS_CONTRACT> "addKmsAggregatedMr(bytes32)" <MR_AGGREGATED_VALUE>
```

> **Note:** This step adds your KMS instance to the allowed list. For production, this typically requires a governance proposal. See [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md) for details.

### 8.3 Run Bootstrap

```bash
KMS_URL="http://<KMS_DOMAIN>:12001"

# Bootstrap: generate root key pair and attestation
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"domain": "<KMS_DOMAIN>"}' \
  "$KMS_URL/prpc/Onboard.Bootstrap?json" | tee bootstrap-info.json | jq .
```

Save the output (`bootstrap-info.json`) — it contains:

- **`ca_pubkey`** — Root CA public key for certificate chain verification
- **`k256_pubkey`** — secp256k1 public key for Ethereum signatures
- **`attestation`** — HEX-encoded TDX attestation report (cryptographic proof of genuine Intel TDX)

> **Important:** Backup `bootstrap-info.json`. You'll need it for KMS contract registration and verification.

### 8.4 Register KMS Public Key On-chain (Optional)

For on-chain governance mode, register the KMS public key to the smart contract:

```bash
# Extract k256 public key from bootstrap-info.json
KMS_PUBKEY=$(cat bootstrap-info.json | jq -r '.k256_pubkey')

# Register via governance proposal (production) or direct call (development)
# See deploy-onchain-kms.md for detailed governance process
```

This step allows applications to verify the KMS identity on-chain.

### 8.5 Finish Bootstrap

```bash
curl "$KMS_URL/finish"
sleep 5
```

KMS will restart and switch to **Normal mode** (HTTPS with RA-TLS). From now on, it only accepts connections from verified workloads.

## Step 9: Verify KMS is Running

After bootstrap, verify the KMS is in Normal mode:

```bash
curl -k https://<KMS_DOMAIN>:12001/health
```

If using RA-TLS, the connection will fail without a valid attestation — this is expected. Workloads connecting via the dstack SDK will automatically handle attestation.

## Common Issues

| Issue | Solution |
|-------|----------|
| Bootstrap fails with "TPM not available" | Ensure the VM is running as a Confidential VM (TDX). Check the GCP console to confirm Confidential VM is enabled. |
| Cannot reach KMS after deploy | Verify the firewall rule: `dstack-cloud fw allow 12001`. Check GCP VPC firewall. |
| Bootstrap succeeds but `/finish` fails | The KMS may need a few moments to process the bootstrap. Wait and retry. |
| Workloads cannot connect after bootstrap | Ensure workloads are using RA-TLS (not plain HTTPS). Verify the workload's attestation is registered on-chain if on-chain mode is enabled. |
| KMS returns "unauthorized measurement" | The workload's measurements are not registered on-chain. Register them via governance. |
| KMS returns "missing field 'status'" | `auth-eth` cannot connect to Ethereum RPC. Verify `ETH_RPC_URL` and `KMS_CONTRACT_ADDR` are set correctly in `docker-compose.yaml`. |
| KMS health check returns "Connection refused" | Check if the port mapping is correct in `docker-compose.yaml`. For Option B (Light Client), ensure the service is listening on the expected port. |

> **For detailed troubleshooting including build issues and advanced diagnostics, see the [official KMS troubleshooting guide](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/tutorials/troubleshooting-kms-deployment.md).**

## Next Steps

Now that your KMS is running:

- **[Register Workload Measurements](register-enclave-measurement.md)** — Authorize your workloads to access keys from KMS
- **[KMS and Key Delivery](../concepts/kms-and-key-delivery.md)** — Understand KMS operating modes and key delivery flow
- **[Attestation Integration](../concepts/attestation-integration.md)** — Learn how TDX + vTPM attestation works

If you haven't deployed smart contracts yet (for on-chain governance), see [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md).
