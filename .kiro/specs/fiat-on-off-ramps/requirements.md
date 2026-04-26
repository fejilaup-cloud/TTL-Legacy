# Requirements Document

## Introduction

This feature integrates fiat on/off-ramp capabilities into the TTL-Legacy vault system, allowing users to deposit fiat currency (e.g. USD, EUR) directly into a vault and withdraw vault assets back to fiat. The integration is built on Stellar's native anchor ecosystem using SEP-6 (Deposit and Withdrawal API), SEP-24 (Hosted Deposit and Withdrawal), and SEP-31 (Cross-Border Payments) protocols. The vault contract itself remains on-chain and token-agnostic; fiat conversion is handled by off-chain anchor providers that issue Stellar-native stablecoins (e.g. USDC, EURC) which are then deposited into the vault. KYC/AML verification is delegated to the anchor provider and surfaced to the client via SEP-12. Settlement is confirmed on-chain via token transfer events. Dispute resolution is handled off-chain through the anchor's support channels, with the vault contract providing an immutable audit trail of all on-chain transactions.

## Glossary

- **Vault**: A time-locked asset container managed by the `TtlVaultContract`, identified by a `vault_id`.
- **Owner**: The Stellar address that created and controls a vault.
- **Beneficiary**: The Stellar address designated to receive vault assets upon release.
- **Anchor**: A Stellar ecosystem service provider that bridges fiat currency to Stellar-native tokens, implementing one or more SEP protocols.
- **Anchor_Provider**: A specific Anchor selected and configured for use with TTL-Legacy (e.g. Circle for USDC, Moneygram for cash).
- **Ramp_Session**: A transient record of a single fiat deposit or withdrawal interaction with an Anchor_Provider, identified by a `session_id`.
- **SEP-6**: The Stellar Ecosystem Proposal defining a non-interactive REST API for fiat deposit and withdrawal.
- **SEP-24**: The Stellar Ecosystem Proposal defining an interactive (hosted iframe/popup) flow for fiat deposit and withdrawal.
- **SEP-31**: The Stellar Ecosystem Proposal defining a direct payment API for cross-border fiat transfers.
- **SEP-12**: The Stellar Ecosystem Proposal defining a KYC data collection API used by anchors.
- **KYC**: Know Your Customer — identity verification required by financial regulations before fiat transactions.
- **AML**: Anti-Money Laundering — compliance checks performed by the Anchor_Provider on fiat transactions.
- **Stellar_Asset**: A Stellar-native token issued by an anchor representing fiat value (e.g. USDC issued by Circle on Stellar).
- **Ramp_Direction**: An enum with values `Deposit` (fiat → vault) and `Withdrawal` (vault → fiat).
- **Ramp_Status**: An enum with values `Pending`, `KycRequired`, `Processing`, `Completed`, `Failed`, `Disputed`.
- **Settlement**: The on-chain token transfer that finalizes a fiat deposit (anchor → vault) or withdrawal (vault → anchor).
- **Ramp_Registry**: The off-chain or client-side store of `Ramp_Session` records keyed by `session_id`.
- **MAX_MEMO_LEN**: Maximum byte length of a Stellar transaction memo used in ramp flows, set to 28 (Stellar text memo limit).
- **Whitelist**: The set of token contract addresses approved for use in vaults, managed by the `TtlVaultContract` admin.

## Requirements

### Requirement 1: Select and Configure Anchor Providers

**User Story:** As a contract operator, I want to configure which anchor providers are approved for fiat on/off-ramp operations, so that only vetted providers can be used to bridge fiat into vaults.

#### Acceptance Criteria

1. THE TtlVaultContract admin SHALL whitelist each Stellar_Asset issued by an approved Anchor_Provider using the existing `whitelist_token` function before that asset can be deposited into a vault.
2. WHEN an Owner attempts to create a vault with a `token_address` that is not whitelisted, THEN THE TtlVaultContract SHALL return `ContractError::InvalidAmount` (existing behavior — no new error code required).
3. THE Ramp_Registry SHALL store the SEP server URL, supported SEP protocols, and supported fiat currencies for each configured Anchor_Provider.
4. WHEN an Anchor_Provider's SEP-1 TOML file is fetched, THE Ramp_Registry SHALL validate that the provider supports at least one of SEP-6, SEP-24, or SEP-31 before storing the provider configuration.
5. IF an Anchor_Provider's SEP-1 TOML file cannot be fetched or parsed, THEN THE Ramp_Registry SHALL reject the provider configuration and return a descriptive error to the operator.

### Requirement 2: Initiate a Fiat Deposit (On-Ramp)

**User Story:** As a vault owner, I want to initiate a fiat deposit into my vault through an anchor provider, so that I can fund the vault with fiat currency without manually acquiring Stellar tokens first.

#### Acceptance Criteria

1. WHEN an Owner initiates a fiat deposit for a `vault_id` and a supported `Stellar_Asset`, THE Ramp_Registry SHALL create a `Ramp_Session` with `direction = Deposit`, `status = Pending`, and a unique `session_id`.
2. WHEN a `Ramp_Session` is created for a SEP-24 provider, THE client SHALL open the anchor's interactive deposit URL in a popup or iframe, passing the Owner's Stellar address and the vault's `token_address` as parameters.
3. WHEN a `Ramp_Session` is created for a SEP-6 provider, THE client SHALL call the anchor's `/deposit` endpoint with the Owner's Stellar address, asset code, and a Stellar transaction memo equal to the `session_id` (truncated to `MAX_MEMO_LEN` bytes).
4. WHEN the anchor requires KYC before processing the deposit, THE Ramp_Registry SHALL update the `Ramp_Session` status to `KycRequired` and provide the SEP-12 KYC URL to the Owner.
5. WHILE a `Ramp_Session` has `status = KycRequired`, THE client SHALL poll the anchor's transaction status endpoint at intervals of no less than 5 seconds until the status changes.
6. IF the anchor rejects the deposit request with an error response, THEN THE Ramp_Registry SHALL update the `Ramp_Session` status to `Failed` and store the anchor's error message.

### Requirement 3: KYC/AML Verification Flow

**User Story:** As a vault owner, I want to complete KYC/AML verification through the anchor provider, so that I can satisfy regulatory requirements and proceed with fiat transactions.

#### Acceptance Criteria

1. WHEN an Anchor_Provider requires KYC for a `Ramp_Session`, THE client SHALL redirect the Owner to the anchor's SEP-12 KYC collection URL.
2. WHEN the Owner submits KYC data to the anchor via SEP-12, THE anchor SHALL process the verification independently of the TtlVaultContract — the contract SHALL NOT store any personally identifiable information.
3. WHEN the anchor confirms KYC approval for a `Ramp_Session`, THE Ramp_Registry SHALL update the `Ramp_Session` status from `KycRequired` to `Processing`.
4. IF the anchor rejects KYC for a `Ramp_Session`, THEN THE Ramp_Registry SHALL update the `Ramp_Session` status to `Failed` and store the rejection reason (excluding any PII).
5. THE TtlVaultContract SHALL NOT store, process, or transmit any KYC or AML data — all identity verification SHALL be handled exclusively by the Anchor_Provider.
6. WHERE an Anchor_Provider supports KYC reuse across sessions, THE client SHALL pass the Owner's previously verified customer ID to avoid redundant KYC submissions.

### Requirement 4: Settle a Fiat Deposit On-Chain

**User Story:** As a vault owner, I want the fiat deposit to be automatically settled into my vault once the anchor confirms the fiat receipt, so that my vault balance reflects the deposited amount without manual intervention.

#### Acceptance Criteria

1. WHEN the anchor transfers the Stellar_Asset to the Owner's Stellar address as part of deposit settlement, THE Owner SHALL call `deposit` on the TtlVaultContract to transfer the received tokens from the Owner's address into the vault.
2. WHEN `deposit` is called with a whitelisted `Stellar_Asset` token address, THE TtlVaultContract SHALL accept the deposit and increase the vault's balance by the transferred amount (existing behavior).
3. WHEN the on-chain `deposit` transaction is confirmed, THE Ramp_Registry SHALL update the `Ramp_Session` status to `Completed` and record the Stellar transaction hash.
4. IF the anchor's on-chain transfer fails or is not received within the anchor's stated processing time, THEN THE Ramp_Registry SHALL update the `Ramp_Session` status to `Failed` and surface the failure to the Owner.
5. WHEN a deposit settlement is completed, THE TtlVaultContract SHALL emit a `deposit` event containing the `vault_id`, deposited amount, and new vault balance (existing behavior — no new event required).

### Requirement 5: Initiate a Fiat Withdrawal (Off-Ramp)

**User Story:** As a vault owner, I want to withdraw vault assets and receive fiat currency in my bank account, so that I can convert on-chain vault holdings back to spendable fiat.

#### Acceptance Criteria

1. WHEN an Owner initiates a fiat withdrawal for a `vault_id` and a `amount`, THE Ramp_Registry SHALL create a `Ramp_Session` with `direction = Withdrawal`, `status = Pending`, and a unique `session_id`.
2. WHEN a `Ramp_Session` is created for a SEP-24 withdrawal, THE client SHALL open the anchor's interactive withdrawal URL, passing the Owner's Stellar address and asset code.
3. WHEN a `Ramp_Session` is created for a SEP-6 withdrawal, THE client SHALL call the anchor's `/withdraw` endpoint to obtain the anchor's Stellar receiving address and memo.
4. WHEN the anchor's receiving address and memo are obtained, THE Owner SHALL call `withdraw` on the TtlVaultContract to transfer tokens from the vault to the Owner's address, then send those tokens to the anchor's receiving address with the specified memo.
5. IF the vault balance is less than the requested withdrawal amount, THEN THE TtlVaultContract SHALL return `ContractError::InsufficientBalance` (existing behavior).
6. WHEN the anchor confirms receipt of the on-chain token transfer, THE Ramp_Registry SHALL update the `Ramp_Session` status to `Processing` while the anchor processes the fiat payout.
7. WHEN the anchor completes the fiat payout, THE Ramp_Registry SHALL update the `Ramp_Session` status to `Completed` and record the anchor's external transaction reference.

### Requirement 6: Transaction Settlement Confirmation

**User Story:** As a vault owner, I want to receive confirmation when a fiat transaction has fully settled, so that I know when funds are available in my bank account or vault.

#### Acceptance Criteria

1. WHEN a `Ramp_Session` transitions to `Completed`, THE Ramp_Registry SHALL record the completion timestamp, the on-chain Stellar transaction hash, and the anchor's external transaction reference.
2. THE client SHALL poll the anchor's SEP-6 or SEP-24 transaction status endpoint at intervals of no less than 5 seconds and no more than 60 seconds until the `Ramp_Session` reaches a terminal status (`Completed` or `Failed`).
3. WHEN the anchor's transaction status endpoint returns `completed`, THE Ramp_Registry SHALL transition the `Ramp_Session` to `Completed`.
4. WHEN the anchor's transaction status endpoint returns `error` or `expired`, THEN THE Ramp_Registry SHALL transition the `Ramp_Session` to `Failed` and store the anchor's error message.
5. THE Ramp_Registry SHALL retain all `Ramp_Session` records in a terminal status for a minimum of 90 days to support audit and dispute resolution.

### Requirement 7: Error Handling

**User Story:** As a vault owner, I want clear error messages when a fiat transaction fails, so that I can understand what went wrong and take corrective action.

#### Acceptance Criteria

1. IF an Anchor_Provider's API returns an HTTP error response during session initiation, THEN THE Ramp_Registry SHALL store the HTTP status code and error body in the `Ramp_Session` and set `status = Failed`.
2. IF a network timeout occurs while communicating with an Anchor_Provider, THEN THE client SHALL retry the request up to 3 times with exponential backoff before setting the `Ramp_Session` status to `Failed`.
3. IF the on-chain `deposit` call fails after the anchor has already transferred tokens to the Owner's address, THEN THE client SHALL surface a recovery prompt instructing the Owner to retry the `deposit` call with the received token amount.
4. IF the on-chain `withdraw` call fails after a withdrawal session has been initiated, THEN THE Ramp_Registry SHALL retain the `Ramp_Session` in `Pending` status and surface a recovery prompt to the Owner.
5. WHEN a `Ramp_Session` transitions to `Failed`, THE Ramp_Registry SHALL emit a client-side event containing the `session_id`, `direction`, and failure reason.

### Requirement 8: Dispute Resolution

**User Story:** As a vault owner, I want to raise a dispute for a failed or incorrect fiat transaction, so that I can recover funds or correct errors with the anchor provider's support team.

#### Acceptance Criteria

1. WHEN an Owner raises a dispute for a `Ramp_Session`, THE Ramp_Registry SHALL update the `Ramp_Session` status to `Disputed` and record the dispute timestamp and Owner-provided description.
2. WHEN a dispute is raised, THE Ramp_Registry SHALL provide the Owner with the anchor's support contact information and the `session_id` as a reference number.
3. THE Ramp_Registry SHALL make the on-chain Stellar transaction hash available for any `Ramp_Session` that reached on-chain settlement, to serve as immutable evidence in dispute resolution.
4. WHEN a dispute is resolved by the Anchor_Provider, THE Ramp_Registry SHALL allow the operator to update the `Ramp_Session` status from `Disputed` to `Completed` or `Failed` with a resolution note.
5. THE TtlVaultContract SHALL NOT be involved in dispute resolution — all disputes SHALL be handled off-chain between the Owner and the Anchor_Provider, with the on-chain transaction record serving as the audit trail.

### Requirement 9: Ramp Session Serialization Round-Trip

**User Story:** As a client developer, I want ramp session records to survive serialization and deserialization without data loss, so that session state is reliably persisted and retrieved across client sessions.

#### Acceptance Criteria

1. THE Ramp_Registry SHALL serialize `Ramp_Session` records to persistent client-side storage using a documented format (e.g. JSON).
2. FOR ALL valid `Ramp_Session` records stored by the Ramp_Registry, deserializing a previously serialized record SHALL produce a `Ramp_Session` with field values byte-for-byte identical to the original (round-trip property).
3. FOR ALL valid `Ramp_Session` records, serializing then deserializing then serializing again SHALL produce output identical to the first serialization (idempotence property).
4. WHEN a `Ramp_Session` record cannot be deserialized due to schema mismatch, THE Ramp_Registry SHALL surface a descriptive error and SHALL NOT silently discard the record.

### Requirement 10: On/Off-Ramp Integration Documentation

**User Story:** As a client developer, I want comprehensive documentation of the fiat on/off-ramp integration, so that I can implement compatible client-side flows and onboard new anchor providers.

#### Acceptance Criteria

1. THE integration documentation SHALL describe the end-to-end deposit flow: anchor selection → session initiation → KYC (if required) → anchor fiat receipt → on-chain token transfer → vault deposit.
2. THE integration documentation SHALL describe the end-to-end withdrawal flow: session initiation → vault withdrawal → on-chain token transfer to anchor → anchor fiat payout.
3. THE integration documentation SHALL list all supported Anchor_Providers, their supported SEP protocols, supported fiat currencies, and their SEP-1 TOML URLs.
4. THE integration documentation SHALL specify the `Ramp_Session` data schema, all `Ramp_Status` values, and valid status transitions.
5. WHERE a new Anchor_Provider is to be added, THE integration documentation SHALL provide a step-by-step guide covering: SEP-1 TOML validation, token whitelisting via `whitelist_token`, and client-side provider configuration.
