# Quick Start: Deploy Your First Dstack App on GCP

This tutorial walks you through deploying a Docker application as a Confidential VM on Google Cloud Platform using dstack-cloud.

**Estimated time:** 10-15 minutes

**What you will do:**

1. Install dstack-cloud CLI
2. Configure GCP credentials
3. Create a project and define your workload
4. Deploy to GCP Confidential VM
5. Verify the deployment and attestation

## Prerequisites

Before you begin, make sure you have:

- A GCP account with **Confidential VM quota** (Intel TDX)
- `gcloud` CLI installed and authenticated (`gcloud auth login`)
- Docker installed on your local machine

### Verify GCP Confidential VM Quota

Confidential VMs with Intel TDX may not be available in all regions. Check your quota:

```bash
gcloud compute project-info describe --project=your-project-id
```

Look for `N2D_CPUS` or similar quotas in your preferred region. If you need to request quota, visit the [GCP Console](https://console.cloud.google.com/iam-admin/quotas).

## Step 1: Install dstack-cloud CLI

Clone the repository and add the CLI to your PATH:

```bash
# Clone the repository
git clone https://github.com/Phala-Network/meta-dstack-cloud.git

# Add CLI to PATH
export PATH="$PATH:$(pwd)/meta-dstack-cloud/scripts/bin"

# Verify installation
dstack-cloud --help
```

> **Note:** The CLI is currently distributed via the repository. Binary releases will be available soon.

## Step 2: Configure GCP Credentials

Set up your GCP project and zone:

```bash
dstack-cloud config-edit
```

This opens the global configuration file. Configure the GCP section:

```toml
[gcp]
project = "your-gcp-project-id"
zone = "us-central1-a"
machine_type = "n2d-standard-4"
```

Replace `your-gcp-project-id` with your actual GCP project ID. The `n2d-standard-4` machine type supports Intel TDX.

## Step 3: Create a Project

Create a new dstack-cloud project:

```bash
dstack-cloud new my-first-app
cd my-first-app
```

This creates a project directory with the following structure:

```
my-first-app/
├── app.json            # Application metadata
├── docker-compose.yaml # Your container definition
├── .env                # Environment variables (encrypted)
└── prelaunch.sh        # Optional pre-launch script (e.g., setup, data download)
```

The `prelaunch.sh` script runs before your containers start. You can use it for setup tasks, downloading data, or other initialization steps.

## Step 4: Define Your Workload

Edit `docker-compose.yaml` to define your application. Here's a simple web server example:

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
```

For AI workloads with GPU:

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    runtime: nvidia
    command: --model Qwen/Qwen2.5-7B-Instruct
    ports:
      - "8000:8000"
```

> **Note:** GPU workloads require NVIDIA Confidential Computing support and additional quota.

## Step 5: Add Secrets (Optional)

If your application needs secrets (API keys, database credentials), add them to `.env`:

```bash
API_KEY=your-secret-key
DATABASE_URL=postgres://user:pass@host:5432/db
```

**Important:** These values are encrypted before leaving your machine and only decrypted inside the TEE.

## Step 6: Deploy

Deploy your application to GCP:

```bash
dstack-cloud deploy
```

The CLI will:

1. Build and push your container configuration
2. Create a Confidential VM with Intel TDX
3. Boot dstack OS (the confidential guest OS)
4. Start your containers
5. Configure automatic disk encryption

The deployment may take a few minutes. Once complete, note the **App ID** and **Public URL** displayed in the output.

## Step 7: Check Status

Monitor your deployment:

```bash
# Check deployment status
dstack-cloud status

# View container logs
dstack-cloud logs

# Follow logs in real-time
dstack-cloud logs --follow
```

## Step 8: Configure Firewall

Allow traffic to your application ports:

```bash
# Allow HTTPS (port 443)
dstack-cloud fw allow 443

# Allow your application port
dstack-cloud fw allow 8080

# List all firewall rules
dstack-cloud fw list
```

## Step 9: Access Your Application

Access your application via the public URL:

```bash
# Using curl
curl https://app-abc123.your-gateway-domain.com

# Or open in browser
open https://app-abc123.your-gateway-domain.com
```

For specific ports:

```
https://app-abc123-8080.your-gateway-domain.com
```

## Step 10: Verify Attestation (Important)

Verify that your application is running in a genuine TEE:

```bash
# Get attestation from your app
curl https://app-abc123.your-gateway-domain.com/attestation

# Verify using dstack-verifier
dstack-verifier verify <attestation-data>
```

The attestation proves:
- The workload runs in genuine Intel TDX hardware
- The exact code and measurements match expectations
- The boot chain integrity is verified via TDX + vTPM

For detailed verification, see [Attestation Integration](../concepts/attestation-integration.md).

## Understanding What Happened

When you deployed your application:

1. **Confidential VM Created** — A GCP VM with Intel TDX was provisioned
2. **dstack OS Booted** — A minimal, attested guest OS started inside the TEE
3. **Automatic Disk Encryption** — All disk I/O is encrypted with keys managed by the Guest Agent
4. **TEE Attestation** — The Guest Agent provides attestation proof via the TDX + vTPM mechanism
5. **TLS Certificate** — Gateway automatically provisions ACME certificates for your domain

### Key Delivery via KMS

dstack uses an external **Key Management Service (dstack-kms)** to deliver keys to your confidential workloads. The KMS runs in its own TEE and only dispatches keys to workloads that pass attestation verification.

## Managing Your Deployment

Your application now runs in a hardware-protected environment where even the cloud provider cannot access the memory or data.

Common management commands:

```bash
# List all deployments
dstack-cloud list

# Stop a deployment
dstack-cloud stop

# Start a stopped deployment
dstack-cloud start

# Remove a deployment completely
dstack-cloud remove
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Deployment stuck at "Creating VM" | Check GCP Confidential VM quota. Verify credentials with `gcloud auth list`. |
| Container fails to start | Check logs with `dstack-cloud logs`. Verify `docker-compose.yaml` syntax. Ensure image is accessible in your chosen region. |
| Cannot access application | Check firewall rules with `dstack-cloud fw list`. Verify port mappings in `docker-compose.yaml`. Check container health status in logs. |
| Attestation verification fails | Ensure you're using production mode (not debug). Check that measurements match expected values. |

## Next Steps

Now that you've deployed your first confidential workload:

- **[Run a Workload on GCP](../how-to/run-dstack-on-gcp.md)** — Detailed deployment options and configurations
- **[Attestation Integration](../concepts/attestation-integration.md)** — Understand TDX + vTPM attestation in depth
- **[Security Model](../concepts/security-model.md)** — Trust boundaries and security guarantees
- **[Deploy on AWS Nitro](../how-to/run-dstack-on-nitro.md)** — Alternative deployment on AWS Nitro Enclaves

For more information:
- **[dstack-cloud Usage Guide](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/usage.md)** — Complete CLI reference
- **[GCP Attestation Details](https://github.com/Phala-Network/dstack-cloud/blob/master/docs/attestation-gcp.md)** — Technical deep-dive on TDX + TPM attestation
