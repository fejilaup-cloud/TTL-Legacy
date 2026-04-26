# Implementation Plan: DAO Governance for Protocol Upgrades

## Overview

Implement a new `GovernanceContract` Soroban smart contract and minimally modify `TtlVaultContract` to support decentralized governance. The work is split into: (1) scaffolding the new contract crate, (2) data types and storage, (3) core governance logic, (4) cross-contract execution, (5) vault authorization changes, and (6) wiring everything together.

## Tasks

- [ ] 1. Scaffold the governance contract crate
  - Create `contracts/ttl_governance/Cargo.toml` referencing `soroban-sdk` and `proptest` (dev-dependency)
  - Add the new crate to the workspace `Cargo.toml`
  - Create `contracts/ttl_governance/src/lib.rs` with an empty `#[contract] pub struct GovernanceContract;` and `#[contractimpl]` block
  - Create `contracts/ttl_governance/src/test.rs` with `#[cfg(test)]` module stub
  - _Requirements: 2.1 (contract must exist to accept proposals)_

- [ ] 2. Define governance types and storage keys
  - [ ] 2.1 Create `contracts/ttl_governance/src/types.rs` with all types from the design
    - `GovernanceConfig`, `ProposalType`, `ProposalPayload`, `Proposal`, `ProposalStatus`, `VoteRecord`, `GovDataKey`, `GovernanceError`
    - All constants: `MAX_DESCRIPTION_LEN`, `GOV_TTL_THRESHOLD`, `GOV_PROPOSAL_TTL`, `GOV_INSTANCE_TTL`, `GOV_INSTANCE_TTL_THRESHOLD`
    - All event topic symbols: `PROPOSAL_CREATED_TOPIC`, `VOTE_CAST_TOPIC`, `PROPOSAL_FINALIZED_TOPIC`, `PROPOSAL_EXECUTED_TOPIC`, `PROPOSAL_CANCELLED_TOPIC`
    - _Requirements: 2.1, 3.1, 4.1, 5.1, 6.1, 7.1, 8.1_

  - [ ]* 2.2 Write property test for config round-trip (Property 13)
    - **Property 13: Config Round-Trip**
    - **Validates: Requirements 7.1, 7.3**
    - Generate random valid `GovernanceConfig` values; assert `get_config()` returns identical fields after init

  - [ ]* 2.3 Write property test for proposal record round-trip (Property 14)
    - **Property 14: Proposal Record Round-Trip**
    - **Validates: Requirements 8.1, 11.2**
    - Generate random valid proposals at various lifecycle stages; assert `get_proposal(id)` returns identical fields

  - [ ]* 2.4 Write property test for vote record round-trip (Property 15)
    - **Property 15: Vote Record Round-Trip**
    - **Validates: Requirements 8.3, 11.3**
    - Generate random voters, support values, and non-zero balances; assert `get_vote` returns identical `support` and `voting_power`

- [ ] 3. Implement `GovernanceContract::initialize` and config queries
  - Implement `initialize(env, governance_token, dao_admin, vault_contract, config)` storing all values in instance storage
  - Validate `config.approval_threshold` is in range 5001–10000 and `config.voting_period > 0`; return `GovernanceError::InvalidConfig` otherwise
  - Implement `get_config(env) -> GovernanceConfig` reading from instance storage (no auth)
  - Extend instance TTL on every state-mutating call
  - _Requirements: 7.1, 7.2, 7.6_

  - [ ]* 3.1 Write unit tests for initialize and config validation
    - Test boundary values: `approval_threshold` = 5001, 10000, 5000, 10001; `voting_period` = 0, 1
    - Test `AlreadyInitialized` error on second call
    - _Requirements: 7.2, 7.4, 7.5_

- [ ] 4. Implement proposal creation (`create_proposal`)
  - Implement `create_proposal(env, proposer, proposal_type, description, payload) -> u64`
  - Validate description byte length ≤ `MAX_DESCRIPTION_LEN`; return `GovernanceError::DescriptionTooLong` otherwise
  - Validate payload matches `ProposalType` (WASM hash for `UpgradeContract`, interval for `SetMin/MaxCheckInInterval`, config for `UpdateGovernanceConfig`, `None` for pause/unpause); return `GovernanceError::InvalidPayload` otherwise
  - Validate `UpdateGovernanceConfig` payload config values at creation time; return `GovernanceError::InvalidConfig` if invalid
  - Check contract not paused; return `GovernanceError::ContractPaused` otherwise
  - Transfer `MIN_PROPOSAL_DEPOSIT` from proposer to governance contract; return `GovernanceError::InsufficientDeposit` if balance insufficient
  - Assign monotonically increasing `Proposal_Id` (increment `ProposalCount`)
  - Store `Proposal` in persistent storage with `GOV_PROPOSAL_TTL`; emit `proposal_created` event
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 1.5, 1.6_

  - [ ]* 4.1 Write property test for proposal creation invariants (Property 4)
    - **Property 4: Proposal Creation Invariants**
    - **Validates: Requirements 2.1, 2.8**
    - Generate random valid proposal inputs; assert `status == Active` and `voting_end_time == start_time + VOTING_PERIOD`

  - [ ]* 4.2 Write property test for monotonically increasing proposal IDs (Property 5)
    - **Property 5: Proposal IDs Are Monotonically Increasing**
    - **Validates: Requirements 2.2, 8.5**
    - Generate N (1–50) valid proposals; assert IDs are 1..N and `get_proposal_count() == N`

  - [ ]* 4.3 Write property test for proposal deposit transferred on creation (Property 3)
    - **Property 3: Proposal Deposit Transferred on Creation**
    - **Validates: Requirements 1.5**
    - Generate random proposer with balance ≥ `MIN_PROPOSAL_DEPOSIT`; assert `proposer_balance_after == proposer_balance_before - MIN_PROPOSAL_DEPOSIT`

- [ ] 5. Implement vote casting (`cast_vote`)
  - Implement `cast_vote(env, voter, proposal_id, support)`
  - Load proposal; return `GovernanceError::ProposalNotFound` if missing
  - Return `GovernanceError::ProposalNotActive` if status ≠ `Active`
  - Return `GovernanceError::VotingPeriodEnded` if `current_time > voting_end_time`
  - Return `GovernanceError::AlreadyVoted` if `Vote(proposal_id, voter)` key exists
  - Read voter's governance token balance; return `GovernanceError::InsufficientVotingPower` if zero
  - Record `VoteRecord { support, voting_power }` in persistent storage; accumulate yes/no tally on proposal
  - Extend TTL of proposal and vote record; emit `vote_cast` event
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 1.2, 1.3, 1.4_

  - [ ]* 5.1 Write property test for voting power equals token balance at vote time (Property 1)
    - **Property 1: Voting Power Equals Token Balance at Vote Time**
    - **Validates: Requirements 1.2**
    - Generate random voter with non-zero balance; assert `vote_record.voting_power == token_balance_at_vote_time`

  - [ ]* 5.2 Write property test for voter token balance unchanged after voting (Property 2)
    - **Property 2: Voter Token Balance Unchanged After Voting**
    - **Validates: Requirements 1.4**
    - Generate random voter and balance; assert `balance_before == balance_after`

  - [ ]* 5.3 Write property test for vote tally accumulation (Property 6)
    - **Property 6: Vote Tally Accumulates Correctly**
    - **Validates: Requirements 3.1**
    - Generate random set of distinct voters with random non-zero balances and support values; assert yes/no tallies equal sums of respective voting powers

  - [ ]* 5.4 Write property test for double-vote prevention (Property 7)
    - **Property 7: Double-Vote Prevention**
    - **Validates: Requirements 3.4**
    - Generate random voter and active proposal; assert second `cast_vote` returns `AlreadyVoted` and tallies are unchanged

- [ ] 6. Implement proposal finalization (`finalize_proposal`)
  - Implement `finalize_proposal(env, proposal_id)`
  - Return `GovernanceError::ProposalNotFound` if missing; `GovernanceError::ProposalNotActive` if status ≠ `Active`
  - Return `GovernanceError::VotingPeriodNotEnded` if `current_time <= voting_end_time`
  - Compute yes ratio as `(yes_votes * 10_000) / (yes_votes + no_votes)` (integer division; treat zero total as failed)
  - If `(yes + no) >= QUORUM` and `ratio >= APPROVAL_THRESHOLD`: set status to `Passed`, set `executable_at = current_time + EXECUTION_DELAY`
  - Otherwise: set status to `Failed`, transfer `MIN_PROPOSAL_DEPOSIT` back to proposer
  - Extend proposal TTL; emit `proposal_finalized` event with final status, yes votes, no votes
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ]* 6.1 Write property test for finalization to Passed (Property 8)
    - **Property 8: Finalization to Passed When Quorum and Threshold Met**
    - **Validates: Requirements 4.1**
    - Generate proposals with vote distributions satisfying quorum and threshold; assert `status == Passed` and `executable_at == finalize_time + EXECUTION_DELAY`

  - [ ]* 6.2 Write property test for finalization to Failed and deposit returned (Property 9)
    - **Property 9: Finalization to Failed When Criteria Not Met, Deposit Returned**
    - **Validates: Requirements 4.2**
    - Generate proposals failing quorum or threshold; assert `status == Failed` and proposer balance restored

  - [ ]* 6.3 Write property test for yes vote ratio formula (Property 10)
    - **Property 10: Yes Vote Ratio Formula Correctness**
    - **Validates: Requirements 4.6**
    - Generate random `yes_votes` and `no_votes` (both > 0); assert computed ratio equals `(yes * 10_000) / (yes + no)` using integer division

- [ ] 7. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Implement proposal execution (`execute_proposal`)
  - Implement `execute_proposal(env, proposal_id)`
  - Return `GovernanceError::ProposalNotFound` if missing; `GovernanceError::ProposalNotPassed` if status ≠ `Passed`
  - Return `GovernanceError::TimelockNotExpired` if `current_time < executable_at`
  - Dispatch cross-contract call based on `ProposalType`:
    - `UpgradeContract` → call `TtlVaultContract::upgrade(wasm_hash)`
    - `SetMinCheckInInterval` → call `TtlVaultContract::set_min_check_in_interval(interval)`
    - `SetMaxCheckInInterval` → call `TtlVaultContract::set_max_check_in_interval(interval)`
    - `PauseContract` → call `TtlVaultContract::pause()`
    - `UnpauseContract` → call `TtlVaultContract::unpause()`
    - `UpdateGovernanceConfig` → update stored `GovernanceConfig` in instance storage
  - On success: set status to `Executed`, transfer `MIN_PROPOSAL_DEPOSIT` back to proposer, emit `proposal_executed` event
  - Propagate vault contract errors without swallowing; leave status as `Passed` on failure
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 7.3_

  - [ ]* 8.1 Write property test for execution invariants (Property 11)
    - **Property 11: Execution Invariants**
    - **Validates: Requirements 5.1, 5.8**
    - Generate random passed proposals past their timelock; assert `status == Executed`, proposer balance increased by `MIN_PROPOSAL_DEPOSIT`, and `proposal_executed` event emitted

- [ ] 9. Implement proposal cancellation (`cancel_proposal`)
  - Implement `cancel_proposal(env, caller, proposal_id)`
  - Return `GovernanceError::ProposalNotFound` if missing
  - Return `GovernanceError::ProposalNotCancellable` if status is `Executed`, `Failed`, or `Cancelled`
  - Return `GovernanceError::Unauthorized` if caller is neither the proposer nor `DAO_Admin`
  - Proposer may cancel only `Active` proposals; `DAO_Admin` may cancel `Active` or `Passed` proposals
  - Set status to `Cancelled`, transfer `MIN_PROPOSAL_DEPOSIT` back to proposer, emit `proposal_cancelled` event
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ]* 9.1 Write property test for cancellation invariants (Property 12)
    - **Property 12: Cancellation Invariants**
    - **Validates: Requirements 6.1, 6.2, 6.5**
    - Generate random cancellable proposals and authorized cancellers; assert `status == Cancelled`, proposer balance restored, and `proposal_cancelled` event emitted

- [ ] 10. Implement query functions
  - Implement `get_proposal(env, proposal_id) -> Proposal` — return `GovernanceError::ProposalNotFound` if missing
  - Implement `get_vote(env, proposal_id, voter) -> VoteRecord` — return `GovernanceError::VoteNotFound` if missing
  - Implement `get_proposal_count(env) -> u64`
  - Implement `has_voted(env, proposal_id, voter) -> bool`
  - None of these functions require authorization
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6_

- [ ] 11. Modify `TtlVaultContract` to support governance authorization
  - [ ] 11.1 Add `GovernanceContract` variant to `DataKey` enum in `contracts/ttl_vault/src/types.rs`
    - _Requirements: 9.2_

  - [ ] 11.2 Add `set_governance_contract(env, governance_contract)` and `get_governance_contract(env) -> Option<Address>` to `TtlVaultContract`
    - `set_governance_contract` requires admin auth; stores address in instance storage; extends instance TTL
    - _Requirements: 9.1, 9.2_

  - [ ] 11.3 Modify `upgrade` to require governance-only auth when governance is set
    - If `GovernanceContract` is set: call `gov.require_auth()` only (admin cannot upgrade)
    - If not set: fall back to `require_admin` (backwards compatible)
    - _Requirements: 9.4, 9.5_

  - [ ] 11.4 Modify `pause`, `unpause`, `set_min_check_in_interval`, `set_max_check_in_interval` to accept governance or admin auth
    - If `GovernanceContract` is set: accept call from either governance contract or admin
    - If not set: require admin only (backwards compatible)
    - _Requirements: 9.1, 9.3, 9.5_

  - [ ]* 11.5 Write property test for admin cannot upgrade when governance is set (Property 16)
    - **Property 16: Admin Cannot Upgrade When Governance Is Set**
    - **Validates: Requirements 9.4**
    - Generate random vault state with governance contract set; assert `upgrade` from admin returns auth error; same call from governance contract succeeds

  - [ ]* 11.6 Write unit tests for vault authorization changes
    - Test backwards compatibility: no governance contract set → admin calls succeed
    - Test governance set → admin `upgrade` rejected, governance `upgrade` accepted
    - Test governance set → admin `pause`/`unpause` still accepted
    - _Requirements: 9.1, 9.3, 9.4, 9.5_

- [ ] 12. Wire governance contract to vault contract in integration tests
  - Write an integration test that deploys both contracts, sets the governance contract address on the vault, creates an `UpgradeContract` proposal, votes to pass it, finalizes, waits past timelock, and executes — verifying the vault WASM is updated
  - Write an integration test for the full `PauseContract` proposal lifecycle
  - Write an integration test for `UpdateGovernanceConfig` proposal lifecycle
  - _Requirements: 5.4, 5.6, 5.7, 7.3, 9.1_

- [ ] 13. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests use the `proptest` crate with a minimum of 100 iterations per test
- Each property test must include a comment: `// Feature: dao-governance-protocol-upgrades, Property N: <property_text>`
- Cross-contract calls from `GovernanceContract` to `TtlVaultContract` use the Soroban SDK `contractclient!` macro or a hand-written client trait
- The `GovernanceContract` does not swallow vault contract errors; failed executions leave the proposal in `Passed` status for retry
