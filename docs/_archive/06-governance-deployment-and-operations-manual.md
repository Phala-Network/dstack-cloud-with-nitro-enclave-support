## Governance Deployment and Operations Manual

### 1. Purpose

This manual is meant for operators and governance participants who deploy and manage the on-chain governance stack for the KMS system.

### 2. Prerequisites

Operators and governance participants should ensure the following before deploying or operating the governance stack:

- **Tools and access**
  - Access to the governance multisig web interface (for example, the Safe web app) on the relevant networks
  - Wallet software that supports multisig signing and hardware wallets
  - Reliable RPC endpoints for the target networks (testnet, pre-production, mainnet)
- **Security posture**
  - Each signer uses a hardware wallet or similarly strong key protection
  - Clear internal processes for who is allowed to sign which types of transactions
  - Multi-factor authentication for accounts that control wallets and infrastructure
  - Secure out-of-band channels (for example, secure chat or ticketing) to coordinate governance actions

### 3. Deployment Procedures

This section provides step-by-step procedures for deploying the governance multisig, timelock module, and KMS-related contracts.

#### 3.1 Deploying the Governance Multisig (Safe)

1. **Select network and owners**
   - Decide which network to deploy the governance multisig on (for example, the target L1/L2 network used by the KMS contracts).
   - Identify the initial set of owners (signers), ensuring representation from both vendor and customer.
2. **Create the multisig**
   - Open the multisig web interface (for example, the Safe web app) and create a new multisig wallet.
   - Add all owner addresses and configure the required signature threshold (for example, 4-of-7).
3. **Fund the multisig**
   - Send a small amount of the network’s native token to the multisig to cover gas for future governance transactions.
4. **Record metadata**
   - Record the multisig address, owners, and threshold in internal configuration (and optionally in an on-chain registry).
   - Share the multisig address with:
     - KMS deployment owners (to wire it into contract deployments)
     - CI/CD owners (if CI/CD needs to generate transactions targeting the multisig)

#### 3.2 Attaching the Timelock Module

1. **Deploy or install the timelock module**
   - Use the governance tooling (for example, a Delay/Timelock module compatible with the multisig) to deploy or install the timelock module.
2. **Configure timelock parameters**
   - Set the cooldown delay (for example, 48 hours in production, shorter in non-production environments).
   - Set an expiration or maximum queue time (for example, 7–14 days) after which pending transactions are no longer executable.
3. **Connect the timelock to the multisig**
   - Configure the multisig so that governance transactions affecting KMS-related contracts must flow through the timelock module.
   - Verify that the module is active and that only module-routed transactions can change critical KMS state.
4. **Record configuration**
   - Record the timelock module address and its parameters alongside the multisig metadata.

#### 3.3 Deploying Governance and KMS-related Contracts

1. **Deploy contracts**
   - Deploy `DstackKms`, `DstackApp`, and any supporting contracts to the chosen network(s).
2. **Transfer ownership to governance multisig**
   - For each contract with an owner or admin role, transfer ownership from the deployer to the governance multisig.
   - Where applicable, configure contracts to recognize the timelock/module as part of the governance flow.
3. **Initialize governance state**
   - Set initial parameters (for example, initial allowed enclave measurements, KMS configuration flags).
   - Use the governance multisig to perform these initializations where possible, ensuring early use of the governance path.
4. **Verify wiring**
   - Confirm that:
     - Ownership of key contracts is held by the governance multisig.
     - The timelock module is active and correctly configured.
     - No critical admin function remains under a single EOA or non-governance contract.

### 4. Routine Governance Operations

This section describes how to perform routine governance actions using the multisig and timelock.

#### 4.1 Creating and Submitting Governance Transactions

1. Identify the required change (for example, add an enclave measurement, update a KMS parameter).
2. Use the multisig interface to create a new transaction:
   - Select the target contract (for example, `DstackKms`).
   - Fill in the function and parameters (using ABI integration or prepared calldata).
3. Add a clear, human-readable description of the change for reviewers.
4. Submit the transaction to the multisig so that other signers can review and approve it.

#### 4.2 Reviewing and Approving Transactions

1. Signers review pending transactions in the multisig UI, checking:
   - Target contract address
   - Function and parameters
   - Description and any associated change tickets
2. If the transaction is correct, signers approve it using their wallets.
3. Once the required threshold of approvals is reached, the transaction:
   - Is forwarded to the timelock module (if configured as a module), or
   - Becomes eligible to be scheduled in the timelock queue, depending on the exact integration pattern.

#### 4.3 Timelock Queue and Execution

1. After a transaction is accepted by the timelock module, it is placed in a queue with:
   - An earliest execution time (current time + cooldown delay)
   - An optional expiration time
2. Operators monitor the timelock queue for:
   - Pending critical governance actions
   - Their earliest execution times
3. Once the cooldown has elapsed, an authorized address (for example, a signer or automation bot) calls the execute function on the timelock or multisig to finalize the transaction.
4. After execution, operators:
   - Verify on-chain state (for example, reading back the updated measurement list or parameter).
   - Update any internal documentation or tickets associated with the change.

#### 4.4 Common Governance Operations

Typical operations and their high-level flow:

- **Adding an authorized enclave measurement**
  - Draft `addMeasurement` (or equivalent) on `DstackKms` via the multisig.
  - Collect required signatures, pass through timelock, then execute.
  - Confirm that the KMS instances pick up the new measurement from on-chain state.

- **Removing or disabling an enclave measurement**
  - Draft `removeMeasurement` or `disableMeasurement` transaction.
  - Follow the same multisig + timelock approval and execution flow.

- **Rotating signers or updating thresholds**
  - Draft a transaction on the multisig itself (for example, `addOwnerWithThreshold`, `removeOwner`, or `changeThreshold`).
  - Consider using a longer timelock and additional off-chain review due to the sensitive nature of signer changes.

- **Updating KMS configuration parameters**
  - Draft transactions to update configuration stored in `DstackKms` or `DstackApp`.
  - Coordinate with KMS operators and CI/CD to ensure rollout timing is understood.

### 5. Handling Errors and Exceptional Events

This section defines procedures for handling failures and exceptional events in governance operations.

#### 5.1 Failed Transactions

If a governance transaction fails during execution (for example, due to a revert or out-of-gas error):

1. Inspect the failure:
   - Review transaction logs and error messages.
   - Check contract state to understand why the call reverted.
2. Decide whether to:
   - Retry with corrected parameters or higher gas limit, or
   - Abandon the change and, if necessary, draft a compensating transaction.
3. Document the failure and resolution in the relevant ticketing or change management system.

#### 5.2 Misconfigured Parameters Detected After Deployment

If a configuration or policy change has been executed but is later found to be incorrect:

1. Assess impact:
   - Determine whether the misconfiguration affects security, availability, or compatibility.
2. Draft a corrective governance transaction:
   - For example, restoring previous values or applying new safe values.
3. Fast-track review and signatures as appropriate (still respecting timelock policies unless a documented emergency path exists).
4. Monitor the system after the fix is executed.

#### 5.3 Multisig and Timelock-specific Issues

For issues involving the multisig or timelock:

- **Pending transaction is wrong but not yet executed**
  1. Identify the transaction in the timelock queue.
  2. Use the timelock’s mechanisms (for example, skipping or cancelling the transaction, or advancing a nonce) to ensure it cannot be executed.
  3. Draft a replacement transaction with corrected parameters and route it through the normal governance flow.

- **Signer key lost or compromised**
  1. Immediately notify all governance participants through agreed secure channels.
  2. Draft a transaction to update the multisig owners or thresholds (for example, removing the compromised signer and adding a new one).
  3. Apply a higher level of scrutiny and potentially a longer timelock for these changes.
  4. After execution, verify the new signer set and thresholds.

- **Timelock misconfiguration**
  1. If timelock delays have been set too low or otherwise misconfigured, draft a transaction to restore safe values.
  2. Consider temporarily reducing the scope of allowed governance changes until the timelock is corrected.

### 6. Operational Checklists

Use the following checklists to standardize governance operations.

#### 6.1 Pre-deployment Checks

- [ ] Governance multisig (Safe) created on the correct network
- [ ] Owners and signature threshold match the agreed governance model
- [ ] Timelock module deployed/installed and configured with agreed delay and expiration
- [ ] Governance multisig and timelock addresses recorded in internal configuration
- [ ] Hardware wallets set up and tested for all signers
- [ ] RPC endpoints and monitoring for governance contracts configured

#### 6.2 Post-deployment Verification

- [ ] Ownership of `DstackKms`, `DstackApp`, and related contracts transferred to the governance multisig
- [ ] No critical admin function remains under a single EOA or non-governance contract
- [ ] An initial governance transaction (for example, a no-op or low-risk change) has been successfully executed through the full multisig + timelock flow
- [ ] On-chain state after execution matches expectations
- [ ] Documentation and runbooks updated with final contract addresses and governance configuration

#### 6.3 Periodic Governance Health Reviews

- [ ] Signer keys are still under control of the intended individuals/teams
- [ ] Signer availability is sufficient to meet the threshold within expected timelines
- [ ] Timelock parameters (delay, expiration) remain aligned with current risk appetite
- [ ] No unexpected changes to governance contract ownership or configuration
- [ ] Monitoring and alerting for governance events are functioning and reviewed regularly
