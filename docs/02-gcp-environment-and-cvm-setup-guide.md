## GCP Environment and CVM Setup Guide

### 1. Purpose and Audience

This guide is intended for infrastructure and SRE teams responsible for preparing and maintaining the GCP environment that hosts the dstack‑based KMS.

It focuses on:

- GCP project and IAM setup
- Network and firewall configuration
- Confidential VM (TDX) configuration
- Prerequisites for deploying the KMS image managed by `dstack-cloud`

### 2. Prerequisites

Before deploying the KMS, the following prerequisites should be met.

#### 2.1 GCP organization and billing

- A GCP organization or standalone project with billing enabled
- Permission to create and manage:
  - Projects (if creating a dedicated project for KMS)
  - VPC networks, subnets, and firewall rules
  - Compute Engine instances and disks

#### 2.2 Tools and access

- Local tools:
  - `gcloud` CLI configured with access to the target project
  - `dstack-cloud` CLI (for provisioning the KMS CVM and managing the project)
  - Optionally Terraform or other IaC tools, if infrastructure is managed as code
- Access:
  - Operator accounts with sufficient IAM roles (see Section 3)
  - Access to the container registry that will host the KMS image

#### 2.3 Operator permissions

Operators who run `dstack-cloud` and manage KMS infrastructure typically require:

- `compute.instances.*` (for instance creation, deletion, and metadata)
- `compute.disks.*` (for attached persistent disks)
- `compute.networks.*` and `compute.firewalls.*` (for network and firewall rules)
- `iam.serviceAccounts.actAs` for service accounts used by the KMS CVM (if applicable)

In most environments, this is provided through curated roles (for example, a “KMS operator” custom role or a combination of built‑in roles).

### 3. GCP Project and IAM

This section describes how to organize the GCP project(s) and IAM roles for the KMS.

#### 3.1 Project layout

Options include:

- **Single project per environment**: one project for each of dev, staging, pre‑production, and production.
- **Shared project with namespaces**: a single project with separate VPCs/subnets and naming conventions for different environments.

For security and blast‑radius reasons, a separate project per environment is recommended for production deployments.

#### 3.2 Service accounts

Define service accounts for:

- **KMS CVM instances**
  - Used by the confidential VM hosting the KMS runtime.
  - Requires access to:
    - Logging/monitoring APIs
    - Container registry (if using Workload Identity or image pull permissions beyond what `dstack-cloud` sets up)
- **CI/CD pipelines** (optional)
  - Used to build and publish KMS images and to run supporting scripts.
- **Monitoring and logging** (optional)
  - Used by external observability systems if they need to pull metrics/logs from GCP.

Service account keys should be avoided where possible in favor of Workload Identity and short‑lived credentials.

#### 3.3 IAM configuration

Apply least‑privilege principles:

- Grant operators and service accounts only the roles needed for:
  - Creating and managing KMS CVMs
  - Managing networks and firewall rules relevant to KMS
  - Accessing the container registry used for KMS images
- Avoid granting project‑wide owner roles to individuals for day‑to‑day operations.

Document which teams or roles are allowed to:

- Deploy or upgrade KMS instances
- Change firewall rules affecting KMS
- Access KMS logs and metrics

### 4. Network and Firewall

Proper network design and firewall rules are critical for isolating the KMS and controlling access to it.

#### 4.1 VPC and subnets

Design the network to separate:

- **KMS subnet**: where the KMS CVM(s) run
- **Management subnet**: jump hosts, bastion, or VPN endpoints used by operators
- **Application subnets**: Nitro host instances or other workloads that will call into the KMS

Decide whether KMS instances:

- Should have external IPs (often unnecessary and avoided in production)
- Will reach out to public RPC endpoints or use private connectivity to dedicated RPC providers

#### 4.2 Firewall rules

Define firewall rules for:

- **Ingress to KMS**
  - Allow HTTPS access on the KMS port (for example, `12001`) only from:
    - Nitro hosts that will terminate RA‑TLS connections, and/or
    - Authorized management networks (for diagnostics)
  - Optionally allow access to the `auth-api` HTTP port (for example, `18000`) for internal health checks and debugging.

- **Egress from KMS**
  - Allow outbound connections to:
    - Ethereum RPC endpoints (direct‑RPC mode)
    - Time sources and other dstack dependencies
    - Monitoring and logging endpoints (for example, Datadog agents or exporters)
  - Restrict outbound traffic to only what is needed.

Ensure that:

- SSH access to the KMS CVM is tightly controlled (or disabled where operationally feasible).
- Any management interfaces are reachable only from secure admin networks or VPNs.

### 5. Confidential VM Configuration

The KMS runs in a GCP Confidential VM (CVM) with TDX support, providing hardware‑backed memory encryption and attestation.

#### 5.1 Machine types

Select machine types and regions that support Confidential VM with TDX. At the time of writing:

- Not all regions or machine families support TDX.
- The GCP console and `gcloud` CLI indicate which machine types support TDX.

Choose instance types that:

- Support TDX Confidential VM
- Provide sufficient CPU and memory for the KMS workload and `dstack-cloud` runtime

#### 5.2 Enabling Confidential VM (TDX)

When defining the KMS instance (either manually or via `dstack-cloud`), ensure that:

- Confidential VM is enabled
- TDX is selected as the underlying technology (where applicable)

In many cases, `dstack-cloud` abstracts these details and sets the correct flags on the instance based on its configuration. Operators should still verify that:

- The instance is reported as a Confidential VM in GCP
- The expected TEE (TDX) is used

#### 5.3 Attestation and isolation

Attestation and isolation are primarily handled by:

- The dstack OS image and runtime, which expose attestation evidence to `dstack-kms`
- The KMS logic, which validates attestation and enforces policy based on it

From an infrastructure perspective, verification involves:

- Confirming that the KMS instance boots with Confidential VM enabled
- Running KMS diagnostics (for example, `GetMeta` or equivalent) to ensure that attestation information is available and consistent with expectations

### 6. Base Image and KMS Image Preparation

This section connects the environment setup with the image build and deployment process described in the KMS image documentation.

#### 6.1 Base OS image

- Obtain a compatible dstack OS image (for example, `dstack-cloud-0.6.0`) that supports GCP TDX.
- Configure `dstack-cloud` to reference this image when creating KMS projects.
- Ensure that the OS image is stored in, or accessible from, the target region(s).

Operators typically do not build the OS image themselves; instead, they consume images published by the dstack OS release process.

#### 6.2 KMS container image

- Build or obtain the KMS container image as described in the KMS image guide.
- Publish the image to a registry accessible from the KMS project:
  - GCP Artifact Registry or GCR, or
  - Another private registry reachable over the configured network

Ensure that:

- Image tags follow the agreed naming convention (for example, include environment and platform)
- The image reference used in `KMS_IMAGE` matches what is available in the registry

### 7. Verification and Smoke Tests

After preparing the environment and deploying a KMS instance, run basic checks to confirm that the CVM and network are correctly configured.

#### 7.1 Verify CVM and network

- Confirm in the GCP console or via `gcloud` that:
  - The KMS instance is running as a Confidential VM
  - The instance has the expected machine type, zone, and labels
- Verify that:
  - Required firewall rules are in place
  - Only expected networks/hosts can reach the KMS HTTPS port

#### 7.2 Verify KMS health

- From an authorized host or Nitro test workload:
  - Call the KMS health or metadata endpoint
  - Verify that the KMS reports:
    - The expected chain/network configuration
    - The expected `DstackKms` contract address
- Check logs to ensure there are no unexpected errors during startup.

#### 7.3 Verify connectivity to dependencies

- Confirm that the KMS can reach:
  - Ethereum RPC endpoints (or that the light client is syncing)
  - Monitoring/logging endpoints
- For Nitro integration, run a simple RA‑TLS test using the reference Nitro application to confirm:
  - Connectivity between Nitro hosts and the KMS
  - That basic key requests work in the test environment

