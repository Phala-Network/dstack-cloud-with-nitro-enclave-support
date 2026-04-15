# Run a dstack-kms CVM on GCP

This guide explains how to deploy a dstack-kms (Key Management Service) instance on GCP as a Confidential VM with Intel TDX, following the production on-chain governance workflow.

> **Reference:** This guide is based on the [official deployment guide](https://github.com/Phala-Network/dstack-cloud-deployment-guide/blob/main/guide_EN.md).

> **Platform Requirement:** `dstack-cloud deploy` must be run on **Linux**. macOS is not supported because the FAT32 shared disk image created by macOS `dosfstools` fails GCP image validation.

## Prerequisites

- A GCP project with Confidential VM (TDX) quota
- `gcloud` CLI installed and authenticated (`gcloud auth login`)
- `docker` / `docker compose`
- `node` + `npm` (for smart contract deployment)
- `jq`
- **`dstack-cloud` CLI** installed:
  ```bash
  curl -fsSL -o ~/.local/bin/dstack-cloud \
    https://raw.githubusercontent.com/Phala-Network/meta-dstack-cloud/main/scripts/bin/dstack-cloud
  chmod +x ~/.local/bin/dstack-cloud
  ```
- **For on-chain governance mode:**
  - A blockchain RPC endpoint (e.g., `https://sepolia.base.org`)
  - A wallet with sufficient balance (e.g., ~0.003 ETH on Base Sepolia)
  - Deployed smart contracts (DstackKms, DstackApp) — see [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md)
  - You'll need the contract addresses (`KMS_CONTRACT_ADDR` and `APP_CONTRACT_ADDR`) before configuring the KMS

> **Note:** This guide uses pre-built KMS Docker images. If you need to build KMS from source or customize the configuration, see the [official KMS build guide](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/tutorials/kms-build-configuration.md).

## Step 1: Configure dstack-cloud

```bash
dstack-cloud config-edit
```

This creates/edits `~/.config/dstack-cloud/config.json`:

```json
{
  "image_search_paths": ["/path/to/your/images"],
  "gcp": {
    "project": "your-gcp-project",
    "zone": "us-central1-a",
    "bucket": "gs://your-bucket-dstack"
  }
}
```

- **`image_search_paths`**: Directory where OS images are stored after `pull`. Must match the path used in Step 2.
- **`bucket`**: GCS bucket for storing deployment artifacts. If it doesn't exist, `dstack-cloud` will create it automatically.

## Step 2: Pull the OS Image

Download the dstack OS image (for GCP Confidential VM / TDX):

```bash
dstack-cloud pull https://github.com/Phala-Network/meta-dstack-cloud/releases/download/v0.6.0-test/dstack-cloud-0.6.0.tar.gz
```

The image is saved to the directory specified in `image_search_paths` (Step 1). The internal image name is `dstack-cloud-0.6.0`.

## Step 3: Create a KMS Project

```bash
mkdir -p workshop-run
dstack-cloud new workshop-run/kms-prod \
  --os-image dstack-cloud-0.6.0 \
  --key-provider tpm \
  --instance-name dstack-kms

cd workshop-run/kms-prod
```

Key flags:
- `--os-image dstack-cloud-0.6.0` — The OS image pulled in Step 2
- `--key-provider tpm` — Use the TPM on the GCP Confidential VM as the root of trust
- `--instance-name` — A human-readable name for the KMS instance

The `new` command generates the following files:
- `app.json` — Project configuration
- `docker-compose.yaml` — Container orchestration template (generic, to be replaced)
- `prelaunch.sh` — Pre-launch script (empty by default)

## Step 4: Build the KMS Docker Image

Build (or pull) the KMS Docker image. You can use the pre-built image or build from source:

```bash
# Option 1: Use pre-built image (no build required)
# Image: dstacktee/dstack-kms:latest

# Option 2: Build from source
cd kms/builder
./build-image.sh dstack-kms:latest
docker push dstack-kms:latest  # Push to your registry
```

> **Note:** Replace `dstack-kms:latest` with your own registry path (e.g., `cr.your-domain.com/dstack-kms:latest`) if you need to push to a private registry.

## Step 5: Configure the KMS

Replace the generated `docker-compose.yaml` and `prelaunch.sh` with the KMS configuration. Choose the connection mode that fits your setup:

### Option A: Direct RPC Connection

```bash
# Copy the direct RPC compose template
cp /path/to/deployment-guide/workshop/kms/docker-compose.direct.yaml docker-compose.yaml
```

Edit `prelaunch.sh` to inject environment variables:

```bash
cat > prelaunch.sh <<'EOF'
#!/bin/sh
cat > .env <<'ENVEOF'
KMS_HTTPS_PORT=12001
AUTH_HTTP_PORT=18000
KMS_IMAGE=dstacktee/dstack-kms:latest
ETH_RPC_URL=https://sepolia.base.org
KMS_CONTRACT_ADDR=<YOUR_CONTRACT_ADDRESS>
APP_CONTRACT_ADDR=<YOUR_APP_CONTRACT_ADDRESS>
ENVEOF
EOF
```

The `prelaunch.sh` script generates a `.env` file at deploy time. The `dstack-cloud deploy` process copies the `.env` into the shared disk, making it available to the containers.

### Option B: Light Client Connection

If you prefer the KMS to run its own light client (helios) for blockchain access:

```bash
# Copy the light client compose template
cp /path/to/deployment-guide/workshop/kms/docker-compose.light.yaml docker-compose.yaml
```

The `prelaunch.sh` is the same as Option A, with the same variables.

### Environment Variables Reference

| Variable | Description | Required |
|----------|-------------|----------|
| `KMS_HTTPS_PORT` | Port for KMS HTTPS service | Yes |
| `AUTH_HTTP_PORT` | Internal port for auth-api (debug) | Yes |
| `KMS_IMAGE` | Docker image name for KMS | Yes |
| `ETH_RPC_URL` | Ethereum RPC endpoint | Yes (on-chain mode) |
| `KMS_CONTRACT_ADDR` | DstackKms contract address | Yes (on-chain mode) |
| `APP_CONTRACT_ADDR` | DstackApp contract address | Yes (on-chain mode) |
| `DSTACK_REPO` | dstack repo URL (for attestation) | Optional |
| `DSTACK_REF` | dstack repo git ref (for attestation) | Optional |

## Step 6: Deploy the KMS CVM

```bash
dstack-cloud deploy --delete
```

The `--delete` flag removes any previous deployment with the same instance name before creating a new one.

Deployment process:
1. Uploads the boot image to GCS (first time only, ~187MB)
2. Creates a GCP image from the OS tar.gz
3. Creates a FAT32 shared disk image with your docker-compose, prelaunch.sh, and .env
4. Provisions a GCP Confidential VM (TDX) instance
5. Attaches the boot image and shared disk

First deployment takes 5-10 minutes.

## Step 7: Open Firewall

```bash
dstack-cloud fw allow 12001
dstack-cloud fw allow 18000
```

For Light Client mode, also allow the helios RPC port:

```bash
dstack-cloud fw allow 18545
```

Port reference:

| Port | Purpose | Required |
|------|---------|----------|
| `12001/tcp` | KMS API | Yes |
| `18000/tcp` | Internal auth-api (debug) | Yes |
| `18545/tcp` | Helios eth RPC (light client mode) | Light Client only |

> **For production:** Consider restricting firewall rules to specific IP ranges instead of `0.0.0.0/0`.

## Step 8: Bootstrap (First-time Only)

When the KMS starts for the first time, it enters **Onboard mode** (HTTP, unauthenticated). You must initialize it before it can serve key requests.

### 8.1 Get the KMS Domain/IP

```bash
dstack-cloud status
```

Note the domain or IP address shown in the output.

### 8.2 Register KMS Measurements (Required for On-chain Mode)

> **Critical:** If using on-chain governance mode, you **must** register the KMS measurements **before** running Bootstrap. Otherwise, Bootstrap will fail because `auth-eth` cannot verify the KMS.

1. Get the KMS attestation info:

```bash
KMS_URL="http://<KMS_DOMAIN>:12001"

curl -s "http://<KMS_DOMAIN>:12001/prpc/Onboard.GetAttestationInfo?json" | jq .
```

2. Register the measurements on-chain:

```bash
npx hardhat kms:add-image <OS_IMAGE_HASH> --network custom
npx hardhat kms:add <MR_AGGREGATED> --network custom
npx hardhat kms:add-device <DEVICE_ID> --network custom
```

> **Important:** The `device_id` from serial console logs is a dummy value. Use the actual value from the RPC/Web UI.

### 8.3 Run Bootstrap

```bash
KMS_URL="http://<KMS_DOMAIN>:12001"

curl -s "$KMS_URL/prpc/Onboard.Bootstrap?json" \
  -d '{"domain": "<KMS_DOMAIN>"}' | tee bootstrap-info.json | jq .
```

Save the output (`bootstrap-info.json`) — it contains:

- **`ca_pubkey`** — Root CA public key for certificate chain verification
- **`k256_pubkey`** — secp256k1 public key for Ethereum signatures
- **`attestation`** — HEX-encoded TDX attestation report

> **Important:** Backup `bootstrap-info.json`. You'll need it for KMS contract registration and verification.

### 8.4 Finish Bootstrap

```bash
curl "$KMS_URL/finish"
sleep 5
```

KMS will restart and switch to **Normal mode** (HTTPS with RA-TLS).

## Step 9: Verify KMS is Running

```bash
# Verify auth-api is running (HTTP, debug port)
curl -s "http://<KMS_DOMAIN>:18000/" | jq .

# Verify KMS is in Normal mode (HTTPS, requires -k for self-signed)
curl -sk "https://<KMS_DOMAIN>:12001/prpc/GetMeta?json" -d '{}' | jq .
```

If using RA-TLS, the connection will fail without a valid attestation — this is expected. Workloads connecting via the dstack SDK will automatically handle attestation.

## Common Issues

| Issue | Solution |
|-------|----------|
| **macOS: "The tar archive is not a valid image"** | `dstack-cloud deploy` is only supported on Linux. macOS `dosfstools` generates FAT32 images that fail GCP validation. Use a Linux machine or VM. |
| **macOS: `mkfs.fat` or `mcopy` errors on shared disk** | Same macOS limitation. Even `mtools` (mcopy/mdel) has compatibility issues with 8MB FAT32 images. Use Linux instead. |
| Bootstrap fails with "TPM not available" | Ensure the VM is running as a Confidential VM (TDX). Check the GCP console to confirm Confidential VM is enabled. |
| Deploy fails with "KMS is not enabled" | Check `app.json` in your project directory. If `env_file` is set to an empty string `""`, remove that key entirely. An empty `env_file` causes a path resolution bug. |
| KMS port responds but all APIs return 404 | The shared disk may contain wrong/old config. Use `dstack-cloud deploy --delete` to recreate from scratch. |
| Cannot SSH into the KMS instance | By design — dstack TDX VMs do not enable SSH daemon for security. To fix config issues, redeploy with correct configuration. |
| Bootstrap succeeds but `/finish` fails | The KMS may need a few moments to process the bootstrap. Wait and retry. |
| KMS returns "missing field 'status'" | `auth-eth` cannot connect to Ethereum RPC. Verify `ETH_RPC_URL` and `KMS_CONTRACT_ADDR` are correct in `.env` (generated by `prelaunch.sh`). |
| Contract deployment says "failed" but tx succeeded | Known issue on public RPCs. Check the transaction hash on-chain — the contract may have been deployed successfully. |

> **For detailed troubleshooting including build issues and advanced diagnostics, see the [official KMS troubleshooting guide](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/tutorials/troubleshooting-kms-deployment.md).**

## Cleanup

To stop or remove the KMS deployment:

```bash
dstack-cloud stop    # Stop the instance (keeps resources)
dstack-cloud remove  # Remove the instance and associated resources
```

To clean up GCP resources manually:

```bash
gcloud compute instances delete dstack-kms --zone=us-central1-a
gcloud compute images delete dstack-kms-boot
gcloud compute images delete dstack-kms-shared
gcloud compute firewall-rules delete allow-kms-12001
```

## Next Steps

Now that your KMS is running:

- **[Register Workload Measurements](register-enclave-measurement.md)** — Authorize your workloads to access keys from KMS
- **[KMS and Key Delivery](../concepts/kms-and-key-delivery.md)** — Understand KMS operating modes and key delivery flow
- **[Attestation Integration](../concepts/attestation-integration.md)** — Learn how TDX + vTPM attestation works

If you haven't deployed smart contracts yet (for on-chain governance), see [Deploy On-chain KMS Smart Contracts](deploy-onchain-kms.md).
