## KMS Image and Deployment Scripts

### 1. Purpose

This document describes how to build, version, and deploy the GCP‑compatible dstack‑based KMS runtime, and how to use the associated deployment scripts and tooling.

It is intended for platform engineers, SREs, and CI/CD owners who are responsible for:

- Preparing and maintaining the KMS runtime image(s)
- Deploying KMS instances into the target GCP environment
- Wiring the KMS to the governance contracts and Nitro workloads

### 2. Image Build Process

The KMS runtime consists of two layers:

- A **dstack OS image** that runs on GCP Confidential VMs (TDX)
- A **KMS container image** that bundles `dstack-kms` and supporting components (for example, `auth-api` and optional `helios` for light‑client mode)

#### 2.1 dstack OS image

The OS image is consumed by the `dstack-cloud` CLI when creating and deploying a KMS project.

- Operators download a published dstack OS image archive and register it in their local `dstack-cloud` configuration.
- The `dstack-cloud new` command then creates a project directory wired to this OS image.

Example (values are illustrative):

```bash
# Create a KMS project bound to a given dstack OS image
dstack-cloud new workshop-run/kms-prod \
  --os-image dstack-cloud-0.6.0 \
  --key-provider tpm \
  --instance-name dstack-kms
```

The generated project directory contains:

- `app.json` – application configuration (OS image, GCP project/zone, instance name, etc.)
- `docker-compose.yaml` – compose file used to run the KMS containers
- `prelaunch.sh` – script invoked by `dstack-cloud` before starting containers
- Support files used by `dstack-cloud` to manage the VM and its persistent disk

At this stage, the OS image is treated as an external artifact; building the OS image from source is handled by the dstack OS release process and is out of scope for this document.

#### 2.2 KMS container image

The KMS logic and its dependencies are packaged as a container image. A reference build script is provided in the original deployment guide under `workshop/kms/builder`.

Key characteristics:

- The image includes:
  - `dstack-kms` (KMS runtime)
  - `auth-api` (on-chain authorization and RA‑TLS entrypoint)
  - Optionally, `helios` for light‑client mode
- Source versions (for example, dstack and helios commits) are pinned in the build script and can be overridden via environment variables.

Typical build flow:

```bash
cd workshop/kms/builder

# Build dstack-kms image (using pinned revisions by default)
./build-image.sh <registry>/<repository>:<tag>

# Push to your container registry
docker push <registry>/<repository>:<tag>
```

In CI, this build step should:

- Pin dstack / helios revisions explicitly
- Produce a versioned image tag (see Section 3)
- Optionally sign or attest the image for provenance tracking

### 3. Versioning and Publishing

KMS images must be versioned in a way that is compatible with:

- Multiple environments (development, staging, pre‑production, production)
- Multiple chains or governance configurations
- Future upgrades and rollbacks

#### 3.1 Naming convention

A recommended container image tag format is:

```text
<registry>/<repo>:<kms-version>-gcp-tdx-<env>
```

For example:

- `registry.example.com/dstack-kms:v1.0.0-gcp-tdx-staging`
- `registry.example.com/dstack-kms:v1.0.0-gcp-tdx-prod`

Where:

- `<kms-version>` corresponds to a git tag or commit range in the dstack KMS codebase
- `gcp-tdx` indicates the target platform/TEE
- `<env>` distinguishes environment‑specific configuration, if necessary

The OS image (`dstack-cloud-0.6.0` in the example above) should also have a clear version and be referenced explicitly in `app.json` and/or `dstack-cloud` configuration.

#### 3.2 Publishing

Container images are typically published to:

- A private container registry reachable from the GCP project (for example, GCR/Artifact Registry or another private registry)

The `KMS_IMAGE` environment variable (see Section 5) is set to the full image reference and used by `docker-compose` in the KMS project.

#### 3.3 Retention and cleanup

Platform owners should define:

- How many historical KMS image versions are retained in the registry
- How long deprecated images remain available for rollback
- A process for archiving or deleting unused images in line with internal policies

### 4. Deployment Scripts and Infra Modules

The primary deployment tooling is the `dstack-cloud` CLI, which orchestrates:

- GCP Confidential VM provisioning using the dstack OS image
- Container startup using `docker-compose`
- Pre‑launch configuration via `prelaunch.sh`

The deployment guide also provides reference `docker-compose` templates for:

- **Direct RPC mode** – KMS connects to an external Ethereum RPC endpoint
- **Light‑client mode** – KMS runs a `helios` light client side‑by‑side

#### 4.1 Project structure

For each KMS deployment, the `dstack-cloud new` command generates a project directory, for example `workshop-run/kms-prod`, with:

- `app.json` – project metadata and OS image reference
- `docker-compose.yaml` – container definition (overridden with the templates below)
- `prelaunch.sh` – hook for writing environment variables into `.env`
- Internal files used by `dstack-cloud` (for example, VM and disk metadata)

#### 4.2 Compose templates

The deployment guide ships two compose templates:

- `docker-compose.direct.yaml` – for direct RPC mode
- `docker-compose.light.yaml` – for light‑client mode with `helios`

Operators copy the chosen template into the project directory as `docker-compose.yaml` and customize configuration through environment variables written by `prelaunch.sh`.

#### 4.3 Required parameters

The `prelaunch.sh` script is responsible for writing a `.env` file in the project directory. At a minimum, it should define:

- `KMS_HTTPS_PORT` – external port for KMS HTTPS API (for example, `12001`)
- `AUTH_HTTP_PORT` – port for internal `auth-api` HTTP endpoint (for example, `18000`)
- `KMS_IMAGE` – full container image reference for the KMS image
- `ETH_RPC_URL` – Ethereum RPC endpoint (direct RPC mode only; ignored in light‑client mode)
- `KMS_CONTRACT_ADDR` – address of the deployed `DstackKms` contract
- `DSTACK_REPO` / `DSTACK_REF` – repository and commit of the `dstack-cloud` code used inside the container, if required

In addition, the deployment configuration must have access to governance‑related parameters defined elsewhere (see governance documents), including:

- The admin Safe (multisig) address that owns or controls the KMS contracts
- The timelock module address attached to the Safe, if applicable
- Any other governance module addresses that the KMS needs to recognize

These are primarily used when deploying and configuring the contracts themselves; the KMS needs `KMS_CONTRACT_ADDR` and related addresses to be consistent with the governance design.

#### 4.4 Topologies

Typical topologies include:

- Single‑region, single‑instance KMS for development or early staging
- Single‑region, multi‑instance KMS using Onboard for key replication
- Multi‑region KMS deployments, where each region runs its own KMS instances with shared or distinct governance, depending on requirements

Multi‑node setups require additional planning for on‑chain authorization and RA‑TLS between KMS instances; details are covered in the Onboard sections of the original deployment guide and in the operations runbook.

### 5. Configuration Management

Runtime configuration for the KMS is managed primarily through environment variables and Docker volumes.

#### 5.1 Environment variables

The `.env` file generated by `prelaunch.sh` is the main configuration surface for:

- Network endpoints (for example, `ETH_RPC_URL`)
- Ports exposed on the host (`KMS_HTTPS_PORT`, `AUTH_HTTP_PORT`)
- On‑chain contract addresses (`KMS_CONTRACT_ADDR`)
- Image references (`KMS_IMAGE`, optional `HELIOS_IMAGE`)
- dstack/dstack‑cloud source references (`DSTACK_REPO`, `DSTACK_REF`)

This `.env` file is consumed by `docker-compose` at container startup. In production, the values written into `.env` should be sourced from:

- A configuration management system, or
- A secrets store / parameter store, rather than hard‑coding credentials in `prelaunch.sh`.

#### 5.2 Persistent data and keys

The KMS uses a persistent volume attached to the VM to store:

- Root and application keys generated during the initial Bootstrap or Onboard process
- Certificates and trust anchors used for HTTPS and RA‑TLS

This volume is managed by `dstack-cloud` and survives container restarts and VM re‑deployments. Operators must treat the underlying disk and any snapshots as sensitive assets, consistent with the threat model.

#### 5.3 Configuration changes and rollout

Configuration changes (for example, updating `ETH_RPC_URL` or `KMS_IMAGE`) should be:

- Managed via versioned configuration (for example, git‑tracked templates) and CI/CD pipelines
- Applied by updating `prelaunch.sh` or its configuration source, then triggering a redeploy (see Section 6)
- Validated via smoke tests and monitoring after rollout

### 6. Deployment Procedures

This section outlines standard procedures for:

- Initial deployment of a new KMS environment
- Upgrading the KMS image
- Rolling back to a previous image

All procedures assume that:

- The GCP environment is prepared as described in the environment guide
- Governance contracts (Safe multisig, timelock, `DstackKms`, `DstackApp`, etc.) are deployed and configured, with addresses available to operators

#### 6.1 Deploy a new KMS environment (direct RPC mode)

1. **Create a KMS project**
   - Use `dstack-cloud new` to create a project directory bound to the desired OS image.
2. **Select the compose template**
   - Copy the direct‑RPC compose template into the project directory as `docker-compose.yaml`.
3. **Configure `prelaunch.sh`**
   - Edit `prelaunch.sh` to write the `.env` file with:
     - `KMS_HTTPS_PORT`, `AUTH_HTTP_PORT`
     - `KMS_IMAGE` (pointing to the chosen KMS container image)
     - `ETH_RPC_URL` (production Ethereum RPC endpoint)
     - `KMS_CONTRACT_ADDR` (deployed `DstackKms` proxy address owned by the governance multisig)
     - Any additional required variables
4. **Deploy via `dstack-cloud`**
   - From the project directory, run:
     ```bash
     dstack-cloud deploy --delete
     ```
   - This provisions or reprovisions the GCP Confidential VM and starts the containers.
5. **Open required firewall ports**
   - Allow inbound access on:
     - KMS HTTPS port (for example, `12001`)
     - Auth‑API HTTP port (for example, `18000`, if needed for diagnostics)
6. **Bootstrap the KMS**
   - Monitor logs until containers are up.
   - Perform the interactive Bootstrap to generate keys and complete initialization as described in the deployment guide.
7. **Post‑deployment checks**
   - Verify `auth-api` and KMS health endpoints.
   - Confirm that metadata endpoints reflect the expected contract address and chain parameters.

#### 6.2 Deploy a new KMS environment (light‑client mode)

Follow the same steps as 6.1 with the following differences:

- Use the light‑client compose template as `docker-compose.yaml`.
- `ETH_RPC_URL` in `.env` may be ignored by the compose template, as the light‑client setup typically hard‑codes the internal `helios` endpoint.
- Open the additional port used by the light client (for example, `18545`) if required.

Bootstrap and verification procedures remain the same.

#### 6.3 Upgrading the KMS image

To upgrade the KMS container image for an existing environment:

1. **Build and publish a new KMS image**
   - Use the process in Section 2.2 to produce and publish a new versioned image.
2. **Update configuration**
   - Update the `KMS_IMAGE` value in the configuration source (for example, `prelaunch.sh` or a higher‑level config file).
3. **Redeploy**
   - From the project directory, rerun:
     ```bash
     dstack-cloud deploy
     ```
   - This restarts the VM and containers using the same persistent volume but the new image.
4. **Verify**
   - Check KMS health endpoints and logs.
   - Confirm that keys and configuration persisted correctly.
   - Monitor for errors or regressions.

If the upgrade involves changes to governance behavior or on‑chain integration, coordinate with the governance process described in the governance documents.

#### 6.4 Rollback

If a newly deployed KMS image must be rolled back:

1. **Select a known‑good image**
   - Identify the previous KMS image tag that is known to be stable.
2. **Update configuration**
   - Change `KMS_IMAGE` back to the known‑good tag.
3. **Redeploy**
   - Run `dstack-cloud deploy` from the project directory.
4. **Verify**
   - Confirm that the KMS is healthy and that there are no unexpected changes in behavior.

Rollbacks should be accompanied by an incident review and, where relevant, updates to CI/CD checks and tests to prevent recurrence.

