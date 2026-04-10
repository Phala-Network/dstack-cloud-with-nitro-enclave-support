## Governance and On-chain Integration Design

### 1. Purpose

This document defines the governance and on-chain integration design for the KMS system. It is intended for smart contract developers, security reviewers, and platform owners.

### 2. On-chain Components

The governance and on-chain integration is built around the following components:

- **`DstackKms`**  
  On-chain registry and policy contract for the KMS. It stores:
  - The list of authorized enclave measurements / identities
  - References to KMS instances or KMS configuration (if needed on-chain)
  - Governance-controlled parameters that affect how the off-chain KMS behaves  
  All admin functions are restricted to the governance layer (multisig + timelock).

- **`DstackApp`**  
  Application-facing contract that integrates with the KMS. Typical responsibilities include:
  - Acting as the on-chain entrypoint for applications that rely on the KMS
  - Holding references to `DstackKms` and other governance-controlled components
  - Enforcing any application-specific checks before delegating to the KMS

- **`GovernanceSafe` (multisig wallet)**  
  A multisig wallet (for example, a Safe) that:
  - Owns or controls `DstackKms`, `DstackApp`, and related contracts
  - Is the only entity allowed to execute sensitive admin functions
  - Is operated by a group of signers from both the vendor and the customer

- **`GovernanceTimelock` (Delay/Timelock module)**  
  A timelock module attached to the multisig that:
  - Enforces a mandatory delay between approval and execution of sensitive transactions
  - Maintains a queue of pending governance actions
  - Optionally enforces expiration for stale transactions

Other supporting contracts (for example, upgrade proxies, registry contracts, or modules for specialized functionality) may be used as needed, but the core principle is that the governance multisig and timelock ultimately control all KMS-related admin actions.

### 3. Multisig and Timelock Selection

The governance design assumes:

- A widely adopted multisig wallet (such as Safe) is used as the primary governance controller:
  - Mature implementation, strong ecosystem and tooling support
  - Audited smart contracts and long track record in production
  - Rich signer UX (hardware wallets, multiple networks, transaction queue)
- A timelock module (such as a Delay/Timelock module attached to the multisig) is used to provide:
  - A minimum cooldown period for sensitive operations
  - Optional expiration for queued transactions
  - A clear, auditable queue of upcoming governance actions

Recommended baseline configuration (to be adjusted by agreement with the customer):

- **Multisig owners and threshold**
  - Production: 5–7 owners drawn from at least two organizations (vendor + customer)
  - Threshold: at least 2/3 of owners (for example, 4-of-6 or 4-of-7)
  - Non-production: may use fewer owners or a lower threshold, but should still avoid single-signer control

- **Timelock parameters**
  - Cooldown delay:
    - Production: 24–72 hours typical for KMS-critical changes
    - Non-production: shorter delays (for example, 1–4 hours) to enable faster iteration
  - Expiration:
    - Transactions that are not executed within a certain window (for example, 7–14 days) should expire and be resubmitted rather than executed unexpectedly.

If an “emergency path” is required (for example, to rapidly disable a compromised enclave measurement), it should be implemented as a separate, clearly documented governance flow with stricter operational controls, rather than by bypassing the timelock entirely.

### 4. Governance Model

The governance model defines who can propose, approve, and execute changes, and how decisions flow through the multisig and timelock:

- **Roles**
  - *Signers (multisig owners)*: Individuals or teams with keys in the multisig. They review and approve transactions.
  - *Proposers*: Any signer (and optionally automated systems like CI) that prepare transactions to be routed through the multisig.
  - *Executors*: Any address permitted by the multisig/timelock setup to call `execute` once all conditions (signatures + delay) are met. In practice this is typically a signer or a designated bot.
  - *Observers*: Stakeholders who do not sign but monitor governance events (e.g., security, audit, compliance).

- **Multisig parameters**
  - Number of signers and required threshold, as described above.
  - Representation from both vendor and customer to avoid unilateral control.
  - Clear ownership of signer keys (e.g., which team or function each key belongs to).

- **Timelock parameters**
  - Cooldown delay and expiration values that reflect the risk of the change being governed.
  - A policy for which changes are subject to which delay (for example, all KMS policy changes must respect the full delay, whereas certain low-risk operations may use a shorter delay if supported).

Governance decisions are made by:

1. Drafting a transaction (or set of transactions) targeting `DstackKms`, `DstackApp`, or related contracts.
2. Submitting the transaction to the multisig for review and approval.
3. Once the required threshold of signers approves, the transaction is enqueued in the timelock.
4. After the timelock delay, the transaction becomes executable and can be executed by an authorized executor.

### 5. KMS and Governance Integration

The KMS system integrates with governance at two levels:

- **On-chain access control**
  - Admin functions on `DstackKms` and `DstackApp` use access control patterns (for example, `onlyGovernance` modifiers) that restrict changes to the governance multisig.
  - No EOA or non-governance contract is allowed to directly mutate critical KMS state (authorized measurements, admin roles, etc.).

- **Off-chain enforcement**
  - When handling requests (for example, `getKey(name)`), the off-chain KMS consults the on-chain state in `DstackKms` to determine whether:
    - The enclave measurement or identity is currently authorized.
    - Any relevant flags (for example, disabled/paused) are set.
  - Changes to the on-chain state propagate to the KMS behavior without requiring manual configuration changes on the KMS side, minimising configuration drift.

Key governance-controlled operations include:

- Adding, updating, or removing authorized enclave measurements or identities.
- Changing KMS-related configuration stored on-chain (for example, maximum key lifetimes or policy flags).
- Updating references to KMS instances (for example, if new instances or regions are introduced).

Sensitive state changes (such as updating allowed enclave measurements or changing KMS administrators) are only accepted when executed by the configured governance multisig, through the timelock-controlled path. Where practical, the system design assumes that:

- Changes become effective only after the timelock delay elapses and the transaction is executed.
- Off-chain systems that cache governance state (for example, KMS instances) refresh their view on an appropriate schedule.

### 6. Upgrade and Change Management

Upgrade and change management covers both smart contracts and off-chain components that rely on governance.

- **Contract upgrades**
  - If upgradeable proxy patterns are used, the proxy admin must be controlled by the governance multisig.
  - Upgrade operations (changing implementation addresses) are:
    - Drafted as multisig transactions
    - Approved by the required threshold of signers
    - Enforced by the timelock delay before execution
  - The design should avoid any “backdoor” upgrade paths outside of the multisig + timelock flow.

- **Policy and configuration changes**
  - Changes to authorized enclave measurements, KMS configuration parameters, and other on-chain settings follow the standard governance flow (proposal → multisig approval → timelock → execution).
  - For changes that need to be coordinated with off-chain deployments (for example, rolling out a new enclave image with a new measurement), the timing of governance execution should be aligned with the CI/CD plan.

- **Mapping change types to governance flows**
  - Low-risk changes (for example, enabling extra telemetry flags) may be grouped and scheduled regularly.
  - High-risk changes (for example, adding a new category of authorized workloads) may require additional review, a longer timelock, or explicit sign-off from designated roles.

### 7. Security Considerations

Governance introduces its own attack surface and operational risks. Key considerations include:

- **Signer compromise or collusion**
  - A compromised signer key may be used to approve malicious transactions.
  - If enough signers collude, they could push through a harmful change during the timelock period.
  - Mitigations:
    - Use hardware wallets and strong operational security for all signers.
    - Choose thresholds that limit the impact of one or two compromised keys.
    - Monitor governance activity and timelock queues for unexpected or high-risk changes.

- **Timelock misconfiguration or bypass**
  - Setting timelock delays too low or disabling them can undermine the ability to detect and respond to malicious actions.
  - Misconfigured or outdated timelock parameters may cause legitimate changes to be delayed or blocked.
  - Mitigations:
    - Treat timelock parameters themselves as sensitive and require strong governance to change them.
    - Monitor timelock configuration and alert on changes or anomalies.

- **Operational risks**
  - Loss of signer keys or lack of available signers may make it difficult or impossible to execute necessary governance actions.
  - Complexity in the governance setup (multiple Safes, modules, and roles) can lead to misconfigured ownership or stuck funds.
  - Mitigations:
    - Regular governance health reviews (for example, confirming signer availability and ownership structure).
    - Clear runbooks for rotating signers, adjusting thresholds, and recovering from misconfigurations.

Monitoring and alerting for governance events (as detailed in the monitoring guide) should complement this design by providing early warning of unusual activity and supporting incident response.
