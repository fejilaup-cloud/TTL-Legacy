# Requirements Document

## Introduction

This feature replaces the single-admin upgrade authority in the TTL-Legacy vault contract with a decentralized governance system. Token holders can create proposals for protocol changes (contract upgrades, configuration updates, pause/unpause), vote on them using a governance token, and have approved proposals executed on-chain. The existing admin role is retained for emergency operations but its scope over protocol upgrades is transferred to the DAO.

## Glossary

- **Governance_Contract**: A new Soroban smart contract that manages proposals, voting, and execution of protocol changes.
- **TtlVaultContract**: The existing vault contract at `contracts/ttl_vault/src/lib.rs` that manages time-locked vaults.
- **Governance_Token**: A Stellar token (SEP-41 compliant) whose balance determines a voter's voting power.
- **Proposal**: An on-chain record describing a protocol change, its current status, and accumulated votes.
- **Proposal_Id**: A unique monotonically increasing `u64` identifier assigned to each Proposal.
- **Proposer**: A Governance_Token holder who creates a Proposal.
- **Voter**: A Governance_Token holder who casts a vote on an active Proposal.
- **Voting_Power**: The number of Governance_Token units held by a Voter at the time of vote casting, expressed in stroops.
- **Voting_Period**: The fixed duration in seconds during which votes may be cast on a Proposal.
- **Quorum**: The minimum total Voting_Power (yes + no votes) required for a Proposal to be eligible for execution.
- **Approval_Threshold**: The minimum ratio of yes votes to total votes (expressed in basis points) required for a Proposal to pass.
- **Execution_Delay**: The mandatory waiting period in seconds between a Proposal passing and it becoming executable.
- **Timelock**: The period defined by Execution_Delay during which a passed Proposal is queued but not yet executable.
- **ProposalType**: An enum describing the action a Proposal requests: `UpgradeContract`, `SetMinCheckInInterval`, `SetMaxCheckInInterval`, `PauseContract`, `UnpauseContract`, or `UpdateGovernanceConfig`.
- **ProposalStatus**: An enum with values `Active`, `Passed`, `Failed`, `Executed`, `Cancelled`.
- **MIN_PROPOSAL_DEPOSIT**: The minimum Governance_Token amount a Proposer must lock to create a Proposal, returned on execution or cancellation.
- **DAO_Admin**: An address retained for emergency operations (pause, cancel proposals) that cannot execute upgrades unilaterally.

## Requirements

### Requirement 1: Governance Token Integration

**User Story:** As a token holder, I want my Governance_Token balance to determine my voting power, so that stakeholders with more skin in the game have proportional influence over protocol decisions.

#### Acceptance Criteria

1. THE Governance_Contract SHALL accept a Governance_Token address at initialization and store it as the sole token used for voting power calculation.
2. WHEN a Voter casts a vote, THE Governance_Contract SHALL read the Voter's current Governance_Token balance from the token contract and record it as the Voter's Voting_Power for that Proposal.
3. WHEN a Voter attempts to vote with a Governance_Token balance of zero, THEN THE Governance_Contract SHALL return `GovernanceError::InsufficientVotingPower`.
4. THE Governance_Contract SHALL NOT transfer or lock Governance_Tokens from Voters during voting; Voting_Power is read-only at vote time.
5. WHEN a Proposer creates a Proposal, THE Governance_Contract SHALL transfer `MIN_PROPOSAL_DEPOSIT` Governance_Tokens from the Proposer to the Governance_Contract as a deposit.
6. IF a Proposer does not hold at least `MIN_PROPOSAL_DEPOSIT` Governance_Tokens, THEN THE Governance_Contract SHALL return `GovernanceError::InsufficientDeposit`.

### Requirement 2: Proposal Creation

**User Story:** As a token holder, I want to create a governance proposal for a protocol change, so that the community can vote on whether to apply it.

#### Acceptance Criteria

1. WHEN a Proposer calls `create_proposal` with a valid `ProposalType`, a description string, and sufficient deposit, THE Governance_Contract SHALL create a new Proposal with status `Active`, assign it a unique `Proposal_Id`, and record the current ledger timestamp as the start time.
2. THE Governance_Contract SHALL assign `Proposal_Id` values as a monotonically increasing sequence starting at 1, with each new Proposal receiving the next integer.
3. WHEN `create_proposal` is called with a description whose byte length exceeds `MAX_DESCRIPTION_LEN` (512 bytes), THEN THE Governance_Contract SHALL return `GovernanceError::DescriptionTooLong`.
4. WHEN `create_proposal` is called with a `ProposalType::UpgradeContract`, THE Governance_Contract SHALL require the new WASM hash (`BytesN<32>`) to be included in the proposal payload.
5. WHEN `create_proposal` is called with a `ProposalType::SetMinCheckInInterval` or `ProposalType::SetMaxCheckInInterval`, THE Governance_Contract SHALL require the new interval value (in seconds) to be included in the proposal payload.
6. WHEN `create_proposal` is called while the Governance_Contract is paused, THEN THE Governance_Contract SHALL return `GovernanceError::ContractPaused`.
7. WHEN `create_proposal` succeeds, THE Governance_Contract SHALL emit a `proposal_created` event containing the `Proposal_Id`, `Proposer` address, and `ProposalType`.
8. THE Governance_Contract SHALL store the Voting_Period end timestamp as `start_time + VOTING_PERIOD` on each Proposal at creation time.

### Requirement 3: Vote Casting

**User Story:** As a token holder, I want to cast a yes or no vote on an active proposal, so that I can participate in governance decisions.

#### Acceptance Criteria

1. WHEN a Voter calls `cast_vote` with a `Proposal_Id` and a boolean `support` value, THE Governance_Contract SHALL record the vote and add the Voter's current Governance_Token balance to the Proposal's yes or no vote tally.
2. WHEN `cast_vote` is called on a Proposal whose status is not `Active`, THEN THE Governance_Contract SHALL return `GovernanceError::ProposalNotActive`.
3. WHEN `cast_vote` is called after the Proposal's Voting_Period end timestamp, THEN THE Governance_Contract SHALL return `GovernanceError::VotingPeriodEnded`.
4. WHEN a Voter calls `cast_vote` on a Proposal they have already voted on, THEN THE Governance_Contract SHALL return `GovernanceError::AlreadyVoted`.
5. WHEN `cast_vote` succeeds, THE Governance_Contract SHALL emit a `vote_cast` event containing the `Proposal_Id`, `Voter` address, `support` value, and `Voting_Power` recorded.
6. WHILE a Proposal is `Active` and within its Voting_Period, THE Governance_Contract SHALL accept votes from any Voter with non-zero Governance_Token balance.

### Requirement 4: Vote Counting and Proposal Resolution

**User Story:** As a community member, I want proposals to be automatically resolved based on vote counts after the voting period ends, so that outcomes are determined transparently and without manual intervention.

#### Acceptance Criteria

1. WHEN `finalize_proposal` is called on a Proposal whose Voting_Period has ended and whose total Voting_Power (yes + no) meets or exceeds `QUORUM`, and whose yes votes expressed in basis points of total votes meets or exceeds `APPROVAL_THRESHOLD`, THE Governance_Contract SHALL set the Proposal status to `Passed` and record the earliest executable timestamp as `current_time + EXECUTION_DELAY`.
2. WHEN `finalize_proposal` is called on a Proposal whose Voting_Period has ended and whose vote totals do not meet `QUORUM` or `APPROVAL_THRESHOLD`, THE Governance_Contract SHALL set the Proposal status to `Failed` and return the Proposer's deposit.
3. WHEN `finalize_proposal` is called on a Proposal whose Voting_Period has not yet ended, THEN THE Governance_Contract SHALL return `GovernanceError::VotingPeriodNotEnded`.
4. WHEN `finalize_proposal` is called on a Proposal whose status is not `Active`, THEN THE Governance_Contract SHALL return `GovernanceError::ProposalNotActive`.
5. WHEN a Proposal transitions to `Passed` or `Failed`, THE Governance_Contract SHALL emit a `proposal_finalized` event containing the `Proposal_Id`, final status, total yes votes, and total no votes.
6. THE Governance_Contract SHALL compute the yes vote ratio as `(yes_votes * 10_000) / (yes_votes + no_votes)` in basis points, using integer arithmetic with no rounding in favor of passage.

### Requirement 5: Proposal Execution

**User Story:** As a community member, I want passed proposals to be executable after the timelock expires, so that approved changes are applied on-chain in a trustless manner.

#### Acceptance Criteria

1. WHEN `execute_proposal` is called on a Proposal with status `Passed` and the current timestamp is greater than or equal to the Proposal's executable timestamp, THE Governance_Contract SHALL invoke the corresponding action on the TtlVaultContract and set the Proposal status to `Executed`.
2. WHEN `execute_proposal` is called on a Proposal with status `Passed` but the current timestamp is less than the Proposal's executable timestamp, THEN THE Governance_Contract SHALL return `GovernanceError::TimelockNotExpired`.
3. WHEN `execute_proposal` is called on a Proposal whose status is not `Passed`, THEN THE Governance_Contract SHALL return `GovernanceError::ProposalNotPassed`.
4. WHEN a `ProposalType::UpgradeContract` Proposal is executed, THE Governance_Contract SHALL call `upgrade` on the TtlVaultContract with the WASM hash stored in the Proposal payload.
5. WHEN a `ProposalType::SetMinCheckInInterval` or `ProposalType::SetMaxCheckInInterval` Proposal is executed, THE Governance_Contract SHALL call the corresponding setter on the TtlVaultContract with the interval value stored in the Proposal payload.
6. WHEN a `ProposalType::PauseContract` Proposal is executed, THE Governance_Contract SHALL call `pause` on the TtlVaultContract.
7. WHEN a `ProposalType::UnpauseContract` Proposal is executed, THE Governance_Contract SHALL call `unpause` on the TtlVaultContract.
8. WHEN `execute_proposal` succeeds, THE Governance_Contract SHALL return the Proposer's `MIN_PROPOSAL_DEPOSIT` and emit a `proposal_executed` event containing the `Proposal_Id` and `ProposalType`.
9. WHEN `execute_proposal` is called and the TtlVaultContract call fails, THEN THE Governance_Contract SHALL propagate the error and leave the Proposal status as `Passed`.

### Requirement 6: Proposal Cancellation

**User Story:** As a proposer or DAO admin, I want to cancel an active proposal before it is executed, so that erroneous or malicious proposals can be stopped.

#### Acceptance Criteria

1. WHEN the Proposer calls `cancel_proposal` on a Proposal with status `Active`, THE Governance_Contract SHALL set the Proposal status to `Cancelled` and return the Proposer's deposit.
2. WHEN the DAO_Admin calls `cancel_proposal` on a Proposal with status `Active` or `Passed`, THE Governance_Contract SHALL set the Proposal status to `Cancelled` and return the Proposer's deposit.
3. WHEN `cancel_proposal` is called by an address that is neither the Proposer nor the DAO_Admin, THEN THE Governance_Contract SHALL return `GovernanceError::Unauthorized`.
4. WHEN `cancel_proposal` is called on a Proposal with status `Executed`, `Failed`, or `Cancelled`, THEN THE Governance_Contract SHALL return `GovernanceError::ProposalNotCancellable`.
5. WHEN `cancel_proposal` succeeds, THE Governance_Contract SHALL emit a `proposal_cancelled` event containing the `Proposal_Id` and the address that cancelled it.

### Requirement 7: Governance Configuration

**User Story:** As a DAO participant, I want governance parameters to be configurable via a governance proposal, so that the community can adjust rules without a centralized admin.

#### Acceptance Criteria

1. THE Governance_Contract SHALL store `VOTING_PERIOD`, `QUORUM`, `APPROVAL_THRESHOLD`, `EXECUTION_DELAY`, and `MIN_PROPOSAL_DEPOSIT` as mutable configuration values.
2. WHEN the Governance_Contract is initialized, THE Governance_Contract SHALL require all configuration values to be provided and SHALL validate that `APPROVAL_THRESHOLD` is between 5001 and 10000 basis points inclusive.
3. WHEN a `ProposalType::UpdateGovernanceConfig` Proposal is executed, THE Governance_Contract SHALL update the stored configuration values with those in the Proposal payload.
4. WHEN `UpdateGovernanceConfig` payload contains an `APPROVAL_THRESHOLD` outside the range 5001–10000 basis points, THEN THE Governance_Contract SHALL return `GovernanceError::InvalidConfig` at proposal creation time.
5. WHEN `UpdateGovernanceConfig` payload contains a `VOTING_PERIOD` of zero seconds, THEN THE Governance_Contract SHALL return `GovernanceError::InvalidConfig` at proposal creation time.
6. THE Governance_Contract SHALL expose a `get_config` read function that returns all current governance configuration values without requiring authorization.

### Requirement 8: Proposal and Vote Queries

**User Story:** As a frontend developer, I want to query proposal state and individual vote records, so that I can build a governance dashboard showing current and historical proposals.

#### Acceptance Criteria

1. THE Governance_Contract SHALL expose a `get_proposal` function that returns the full Proposal record for a given `Proposal_Id` without requiring authorization.
2. WHEN `get_proposal` is called with a `Proposal_Id` that does not exist, THEN THE Governance_Contract SHALL return `GovernanceError::ProposalNotFound`.
3. THE Governance_Contract SHALL expose a `get_vote` function that returns the vote record (support value and Voting_Power) for a given `(Proposal_Id, Voter)` pair without requiring authorization.
4. WHEN `get_vote` is called for a `(Proposal_Id, Voter)` pair with no recorded vote, THEN THE Governance_Contract SHALL return `GovernanceError::VoteNotFound`.
5. THE Governance_Contract SHALL expose a `get_proposal_count` function that returns the total number of Proposals ever created.
6. THE Governance_Contract SHALL expose a `has_voted` function that returns `true` if the given Voter has voted on the given Proposal, and `false` otherwise, without requiring authorization.

### Requirement 9: TtlVaultContract Authorization Update

**User Story:** As a contract operator, I want the TtlVaultContract to accept governance-authorized calls for upgrades and configuration changes, so that the DAO can execute approved proposals without the existing admin.

#### Acceptance Criteria

1. THE TtlVaultContract SHALL accept the Governance_Contract address as an authorized caller for `upgrade`, `set_min_check_in_interval`, `set_max_check_in_interval`, `pause`, and `unpause`.
2. WHEN the TtlVaultContract is initialized, THE TtlVaultContract SHALL store the Governance_Contract address alongside the existing admin address.
3. WHILE the Governance_Contract address is set, THE TtlVaultContract SHALL treat calls from the Governance_Contract address as having admin-equivalent authority for the functions listed in criterion 1.
4. THE TtlVaultContract SHALL retain the existing admin address for emergency operations (pause, unpause) and admin transfer, but SHALL NOT allow the admin to call `upgrade` directly once a Governance_Contract address is set.
5. IF the Governance_Contract address is not set, THEN THE TtlVaultContract SHALL fall back to requiring the existing admin for all operations, preserving backwards compatibility.

### Requirement 10: Governance Contract Storage and TTL

**User Story:** As a contract operator, I want governance data to persist reliably on-chain with appropriate TTL settings, so that proposal and vote records remain accessible throughout the governance lifecycle.

#### Acceptance Criteria

1. THE Governance_Contract SHALL store Proposal records in persistent storage with a TTL sufficient to outlive the `VOTING_PERIOD + EXECUTION_DELAY + 30 days` window.
2. THE Governance_Contract SHALL store vote records in persistent storage with the same TTL as the associated Proposal.
3. WHEN `cast_vote` or `finalize_proposal` is called, THE Governance_Contract SHALL extend the TTL of the associated Proposal and all its vote records.
4. THE Governance_Contract SHALL store configuration values in instance storage and extend instance TTL on every state-mutating call.
5. WHEN a Proposal reaches `Executed`, `Failed`, or `Cancelled` status, THE Governance_Contract SHALL retain the Proposal record in storage for auditability and SHALL NOT delete it.

### Requirement 11: Governance Data Serialization Round-Trip

**User Story:** As a contract developer, I want Proposal and vote records to survive serialization and deserialization through Soroban storage without data loss, so that governance state is always consistent.

#### Acceptance Criteria

1. THE Governance_Contract SHALL serialize Proposal records to persistent storage using Soroban's native XDR encoding.
2. FOR ALL valid Proposal records written via `create_proposal` or `finalize_proposal`, reading the same record via `get_proposal` SHALL return a value with identical field values (round-trip property).
3. FOR ALL valid vote records written via `cast_vote`, reading the same record via `get_vote` SHALL return a vote with identical `support` and `Voting_Power` values (round-trip property).
