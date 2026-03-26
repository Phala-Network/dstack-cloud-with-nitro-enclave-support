## GCP Environment and CVM Setup Guide

### 1. Purpose and Audience

This guide is intended for infrastructure and SRE teams responsible for preparing and maintaining the GCP environment that hosts the dstack-based KMS. It focuses on:

- GCP project and IAM setup
- Network and firewall configuration
- Confidential VM (TDX/SGX) configuration
- Prerequisites for deploying the KMS image

### 2. Prerequisites

This section should list:

- Required GCP organization and billing setup
- Required CLI tools (e.g., `gcloud`, Terraform if used)
- Required roles and permissions for operators

### 3. GCP Project and IAM

Describe how to:

- Create or select the GCP project(s) used for KMS
- Define service accounts and roles for:
  - KMS instances
  - CI/CD pipelines
  - Monitoring and logging
- Apply least privilege principles for IAM configuration

### 4. Network and Firewall

Describe:

- VPC and subnet layout for KMS, management, and supporting services
- Firewall rules for:
  - Ingress to the KMS API (if any)
  - Egress to RPC providers, monitoring endpoints, and other dependencies
- Any network segmentation or isolation requirements

### 5. Confidential VM Configuration

Explain how to:

- Select appropriate machine types that support TDX/SGX confidential computing
- Configure images and boot parameters required by dstack-kms
- Validate that confidential computing and remote attestation are enabled

### 6. Base Image and KMS Image Preparation

This section should connect to the KMS image build process and explain:

- Which base OS image is used
- How the dstack-kms image is derived from the base image
- How images are stored (e.g., in an internal image registry or custom image catalog)

### 7. Verification and Smoke Tests

Finally, define simple steps to verify that:

- A new CVM instance can be created successfully
- Attestation and isolation settings are correct
- Basic connectivity (to RPC, monitoring, and CI/CD) is functioning

