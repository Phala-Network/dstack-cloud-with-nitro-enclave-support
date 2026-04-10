# Run a Workload on GCP

This guide explains how to deploy a Docker application as a dstack CVM (Confidential Virtual Machine) on GCP using Intel TDX.

## Prerequisites

- A GCP project with Confidential VM quota enabled
  - Intel TDX Confidential VMs are available in select zones (e.g., `us-central1-a`)
- `gcloud` CLI installed and authenticated
  ```bash
  gcloud auth login
  gcloud config set project YOUR_PROJECT_ID
  ```
- Docker installed on your local machine
- `dstack-cloud` CLI installed
  ```bash
  curl -fsSL -o ~/.local/bin/dstack-cloud \
    https://raw.githubusercontent.com/Phala-Network/meta-dstack-cloud/main/scripts/bin/dstack-cloud
  chmod +x ~/.local/bin/dstack-cloud
  ```

## Step 1: Configure dstack-cloud

Edit the dstack-cloud configuration:

```bash
dstack-cloud config-edit
```

Set the following values:

```toml
[gcp]
project = "YOUR_PROJECT_ID"
zone = "us-central1-a"
bucket = "gs://YOUR_BUCKET_NAME"

[kms]
url = "https://your-kms-host:12001"

[gateway]
url = "https://your-gateway-host"
```

### KMS Options

You have two options for the KMS:

| Option | Description | When to Use |
|--------|-------------|-------------|
| **Phala Official KMS** | Use the KMS hosted by Phala Network | Quick start, development, testing |
| **Self-hosted KMS** | Deploy your own KMS instance | Production, compliance requirements, full control |

For self-hosted KMS, you can deploy on:
- **GCP** — See [Run a dstack-kms CVM on GCP](run-dstack-kms-on-gcp.md)
- **Intel TDX Bare Metal** — Contact Phala for deployment guide

![GCP KMS Key Delivery](../images/kms-key-delivery-gcp-v5.png)

When your GCP CVM starts, the dstack-agent inside contacts the KMS via RA-TLS to retrieve encrypted keys. The KMS verifies the CVM's attestation before dispatching keys.

### Configuration Fields

- **`[kms] url`** — Address of your dstack-kms instance. The CVM's Guest Agent contacts this URL to retrieve keys after attestation. Required if you are using KMS mode (recommended for production).
- **`[gateway] url`** — Address of the dstack gateway. The gateway handles TLS termination and routes traffic to your CVMs.

The `[gcp] bucket` is used to store CVM images during the build and deploy process. Create it if it does not exist:

```bash
gsutil mb gs://YOUR_BUCKET_NAME
```

## Step 2: Pull the OS Image

Download the dstack OS image to your local machine:

```bash
dstack-cloud pull --os-image dstack-cloud-0.6.0
```

## Step 3: Create a Project

```bash
dstack-cloud new my-gcp-app --os-image dstack-cloud-0.6.0
cd my-gcp-app
```

## Step 4: Define Your Application

Edit `docker-compose.yaml` to define your application:

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
```

## Step 5: (Optional) Add Environment Variables

If your application needs secrets or configuration, add them to the `.env` file:

```
API_KEY=your-api-key-here
DATABASE_URL=postgres://user:pass@host:5432/db
```

Environment variables are encrypted before leaving your machine and decrypted only inside the CVM.

## Step 6: Deploy

```bash
dstack-cloud deploy
```

dstack-cloud will:

1. Build the CVM image (packages your containers into dstack OS)
2. Upload the image to your GCP bucket
3. Create a Confidential VM instance with Intel TDX enabled
4. Start the VM and run your containers

First deployment typically takes 5-10 minutes.

## Step 7: Open Firewall

Allow external access to your application:

```bash
# Allow HTTPS (port 443)
dstack-cloud fw allow 443

# Allow your application port
dstack-cloud fw allow 8080

# List all firewall rules
dstack-cloud fw list
```

## Step 8: Verify

Check the deployment status:

```bash
dstack-cloud status
```

View logs:

```bash
dstack-cloud logs --follow
```

Access your application at the URL shown in the status output (typically `https://<app-id>.<gateway-domain>`).

## Managing Your Deployment

### View Logs

```bash
dstack-cloud logs
dstack-cloud logs --follow    # Real-time streaming
```

### Stop / Start

```bash
dstack-cloud stop
dstack-cloud start
```

### Remove

```bash
dstack-cloud remove
```

This deletes the GCP VM and associated resources.

## Common Issues

| Issue | Solution |
|-------|----------|
| "Confidential VM quota exceeded" | Request TDX quota increase in GCP Console → IAM & Admin → Quotas |
| "Permission denied" on bucket | Run `gsutil iam ch allUsers:objectViewer gs://YOUR_BUCKET_NAME` or use a service account with storage permissions |
| VM starts but container exits immediately | Check logs: `dstack-cloud logs`. Ensure the Docker image is valid and the container can start. |
| Cannot access the application URL | Verify the firewall rule: `dstack-cloud fw allow <port>`. Also check GCP VPC firewall rules. |
| Attestation fails | Ensure the VM is running as a Confidential VM (TDX). Check the GCP console for the `confidential-compute` flag. |

## Next Steps

- **[Run a dstack-kms CVM on GCP](run-dstack-kms-on-gcp.md)** — Set up KMS for key management
- **[Concept: Overview](../concepts/overview.md)** — Understand the architecture
- **[Run a Workload on AWS Nitro](run-dstack-on-nitro.md)** — Deploy on AWS instead
