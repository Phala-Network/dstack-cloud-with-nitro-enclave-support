## Nitro Enclave Integration and RA-TLS Guide

### 1. Purpose

This document explains how Nitro Enclave workloads integrate with the KMS using RA-TLS and the `dstack-agent-nitro` library. It is aimed at application developers and platform engineers responsible for enclave workloads.

This guide complements the reference Nitro application and CI/CD template provided in the `dstack-nitro-enclave-app-template` repository. That template can be used as a starting point for building and releasing enclave applications that integrate with this KMS.

### 2. Overview of RA-TLS in This Project

At a high level:

- The Nitro enclave establishes a mutual TLS (mTLS) connection to the KMS.
- The enclave’s attestation evidence and measurements are embedded into the TLS handshake.
- The KMS validates:
  - The TLS chain up to the pinned KMS root CA (from `root_ca.pem` in the enclave image)
  - The attestation quote and measurements against the on-chain `DstackKms` registry
  - Any additional governance constraints (for example, whether the measurement is enabled)
- Only if these checks pass does the KMS release keys to the enclave.

The RA-TLS flow binds:

- The enclave code and configuration (represented by measurements and `OS_IMAGE_HASH`)
- The network connection (TLS)
- The authorization policy in `DstackKms`

### 3. dstack-agent-nitro

The `dstack-agent-nitro` library (or agent) runs inside the enclave and is responsible for:

- Establishing the RA-TLS connection to the KMS
- Presenting attestation evidence to the KMS
- Managing local key handles and secure use of keys provided by the KMS

Conceptually, an enclave application using the agent will:

1. Initialize the agent with:
   - KMS URL
   - Application ID (`APP_ID`)
   - Root CA certificate for KMS TLS (`root_ca.pem`)
2. Ask the agent to open a RA-TLS session with the KMS.
3. Use the agent’s API to request keys and perform cryptographic operations.

Pseudo-code (illustrative only):

```pseudo
// Initialize agent
agent = DstackNitroAgent.new(
    kms_url = "https://kms.example.com:12001",
    app_id = "0xapp...",
    root_ca_path = "/app/root_ca.pem",
)

// Establish RA-TLS session (performs attestation and TLS handshake)
session = agent.connect()

// Request a named key
key = session.getKey("my-signing-key")

// Use key for an operation (e.g., sign or decrypt)
signature = session.sign(key, message)
```

Actual APIs may differ; the goal of this section is to show the pattern: initialize → connect (RA-TLS) → use keys via the agent.

### 4. getKey(name) API

The `getKey(name)` API is the primary way Nitro enclave applications obtain keys from the KMS.

Semantics:

- Input:
  - `name`: logical name of the key, scoped by `APP_ID` and governance policy
- Behavior:
  - Validates the RA-TLS session and associated attestation evidence
  - Checks on-chain state in `DstackKms` to ensure:
    - The enclave’s measurement (or `OS_IMAGE_HASH`) is authorized for this key
    - The key is not disabled or restricted by governance
  - Returns a key handle or key material (depending on implementation)

Typical error scenarios:

- **Measurement mismatch**
  - The enclave’s measurement is not registered or is disabled in `DstackKms`.
  - Expected outcome: `getKey(name)` fails with an authorization error; logs indicate unauthorized measurement.

- **Lack of governance approval**
  - The key or measurement was not approved via governance (for example, not yet passed through timelock or executed).
  - Expected outcome: `getKey(name)` fails, and KMS logs show a governance policy violation.

- **Attestation failures**
  - RA-TLS attestation quote invalid, expired, or inconsistent with expected hardware.
  - Expected outcome: the RA-TLS handshake fails or `getKey(name)` is rejected.

In all cases, enclave applications should distinguish between:

- Retries for transient issues (for example, network glitches)
- Hard failures due to authorization or policy (for example, measurement not authorized)

### 5. End-to-End Flow

At a high level, the end-to-end flow for RA-TLS and key retrieval is:

1. **Build and register the enclave image**
   - Build the EOF using the Nitro template CI.
   - Compute `OS_IMAGE_HASH`.
   - Register the measurement via governance (Safe + timelock) in `DstackKms`.
2. **Deploy the KMS**
   - Ensure KMS is running and connected to the target chain.
   - Verify KMS is configured with the correct `DstackKms` address.
3. **Deploy the enclave application**
   - Configure `KMS_URL`, `APP_ID`, and `root_ca.pem`.
   - Launch the enclave.
4. **RA-TLS handshake**
   - Enclave agent connects to KMS using TLS, presenting attestation evidence.
   - KMS verifies attestation and measurements.
5. **Key request**
   - Enclave calls `getKey(name)` through the agent.
   - KMS evaluates policy in `DstackKms` and governance state.
   - If authorized, KMS returns the key (or key handle) to the enclave.
6. **Key usage and logging**
   - Enclave uses the key for the intended cryptographic operations.
   - KMS and enclave logs capture the request and any relevant metrics.

Sequence diagrams can be added to this section as needed during implementation.

### 6. Demo Scenario

Define a simple demo scenario that:

- Uses the `dstack-nitro-enclave-app-template` as the Nitro application
- Establishes RA-TLS with the KMS
- Successfully calls `getKey(name)` and uses the key (for example, to sign or decrypt data)

Step-by-step outline:

1. **Prepare KMS**
   - Deploy a KMS instance in a test environment, as described in the KMS deployment guide.
   - Obtain the KMS root CA certificate and ensure governance contracts are deployed.

2. **Prepare Nitro app**
   - Fork or copy the `dstack-nitro-enclave-app-template` repository.
   - Replace `app/root_ca.pem` with the KMS root CA.
   - Set `KMS_URL` and `APP_ID` to point to the test KMS.

3. **Build and release EIF**
   - Use the provided GitHub Actions workflow to build the enclave image and publish a release containing:
     - EIF file
     - Measurements (PCRs) and `OS_IMAGE_HASH`
     - Sigstore attestation bundle

4. **Register measurement**
   - For development/test:
     - Optionally, use the direct `kms:add-image` task as shown in the Nitro template README.
   - For production-like flows:
     - Use `OS_IMAGE_HASH` as input to a governance proposal.
     - Execute the proposal via Safe + timelock to register the image in `DstackKms`.

5. **Run the demo**
   - Launch the enclave using the built EIF.
   - Trigger the example logic (for example, `dstack-util get-keys`) to:
     - Establish RA-TLS with the KMS
     - Request a key with `getKey(name)`
   - Observe logs on both the enclave and KMS side to confirm:
     - Attestation and measurement validation succeeded
     - Key retrieval behaved as expected

The demo should be reproducible on a developer or test environment and serves as the basis for the "Verified RA-TLS demo" deliverable.

As a concrete reference, the `dstack-nitro-enclave-app-template` repository provides:

- A minimal Nitro enclave application that can be wired to `dstack-agent-nitro` and the KMS
- A CI workflow that:
  - Builds an enclave image (EIF)
  - Computes measurements and an `OS_IMAGE_HASH` value
  - Publishes a GitHub release including measurement and image metadata
  - Produces Sigstore attestations that can be used to verify the provenance of the enclave image

In production, the measurements and `OS_IMAGE_HASH` produced by this pipeline should not be registered on-chain directly by an externally owned account. Instead, they should be used as inputs to governance proposals executed through the multisig and timelock setup described in the governance documents.
