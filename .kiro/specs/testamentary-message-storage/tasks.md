# Implementation Plan: Testamentary Message Storage

## Overview

Extend `TtlVaultContract` with four new entry points (`store_message`, `delete_message`, `get_message`, `has_message`), two new `DataKey` variants, five new `ContractError` codes, three new event topic constants, and lifecycle hooks in `check_in` and `cancel_vault`.

## Tasks

- [ ] 1. Add types, constants, and error codes
  - [ ] 1.1 Add message constants and event topics to `types.rs`
    - Add `MAX_MESSAGE_SIZE: u32 = 4096` and `MAX_MESSAGES_PER_VAULT: u32 = 10` constants
    - Add `MSG_STORED_TOPIC`, `MSG_UPDATED_TOPIC`, `MSG_DELETED_TOPIC` symbol constants
    - _Requirements: 1.9, 2.2, 3.5, 5.1_

  - [ ] 1.2 Add `MessageData` and `MessageCount` variants to `DataKey` enum in `types.rs`
    - Add `MessageData(u64, Address)` and `MessageCount(u64)` to the `DataKey` enum
    - _Requirements: 5.1_

  - [ ] 1.3 Add new `ContractError` codes in `lib.rs`
    - Add `MessageTooLarge = 25`, `MessageLimitReached = 26`, `InvalidMessage = 27`, `MessageNotFound = 28`, `NotReleased = 29`
    - _Requirements: 1.4, 1.5, 1.6, 3.3, 4.2_

  - [ ] 1.4 Update `use types::` import in `lib.rs` to include new symbols and constants
    - Import `MSG_STORED_TOPIC`, `MSG_UPDATED_TOPIC`, `MSG_DELETED_TOPIC`, `MAX_MESSAGE_SIZE`, `MAX_MESSAGES_PER_VAULT`
    - _Requirements: 1.9, 2.2, 3.5_

- [ ] 2. Implement `store_message`
  - [ ] 2.1 Implement `store_message` entry point in `lib.rs`
    - Assert not paused; load vault; require owner auth; assert vault is `Locked`
    - Validate ciphertext non-empty (`InvalidMessage`) and `<= MAX_MESSAGE_SIZE` (`MessageTooLarge`)
    - Validate beneficiary is `vault.beneficiary` or in `vault.beneficiaries` (`InvalidBeneficiary`)
    - Check if key exists; if new, load count, assert `< MAX_MESSAGES_PER_VAULT`, increment and save `MessageCount`
    - Save `MessageData` with `extend_ttl` using `vault_ttl_ledgers(vault.check_in_interval)`
    - Emit `MSG_STORED_TOPIC` (new) or `MSG_UPDATED_TOPIC` (overwrite) with `(vault_id, beneficiary)`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 2.1, 2.2, 2.3, 5.1, 6.1, 6.2_

  - [ ]* 2.2 Write unit tests for `store_message`
    - `test_store_message_persists_ciphertext` — store then get (after release) returns same bytes
    - `test_store_message_requires_owner_auth` — non-owner call panics
    - `test_store_message_rejects_released_vault` — returns `AlreadyReleased`
    - `test_store_message_rejects_cancelled_vault` — returns `AlreadyReleased`
    - `test_store_message_rejects_empty_ciphertext` — returns `InvalidMessage`
    - `test_store_message_rejects_oversized_ciphertext` — returns `MessageTooLarge` at 4097 bytes
    - `test_store_message_rejects_at_limit` — 10 messages stored, 11th returns `MessageLimitReached`
    - `test_store_message_rejects_invalid_beneficiary` — non-beneficiary address returns `InvalidBeneficiary`
    - `test_store_message_rejects_when_paused` — returns `Paused`
    - `test_store_message_emits_stored_event` — first store emits `msg_store` topic
    - `test_store_message_overwrite_emits_updated_event` — second store emits `msg_upd` topic
    - `test_store_message_overwrite_does_not_increment_count` — count stays at 1 after overwrite
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 2.1, 2.2, 2.3_

  - [ ]* 2.3 Write property test for `store_message` round-trip (Property 1)
    - **Property 1: Message Round-Trip**
    - **Validates: Requirements 1.1, 4.1, 6.1, 6.2, 8.2**

  - [ ]* 2.4 Write property test for owner auth requirement (Property 2)
    - **Property 2: Mutating Operations Require Owner Authorization**
    - **Validates: Requirements 1.2, 3.4**

  - [ ]* 2.5 Write property test for non-Locked vault rejection (Property 3)
    - **Property 3: Message Mutations Rejected on Non-Locked Vaults**
    - **Validates: Requirements 1.3, 3.2**

  - [ ]* 2.6 Write property test for invalid beneficiary rejection (Property 4)
    - **Property 4: Invalid Beneficiary Rejected**
    - **Validates: Requirements 1.7**

  - [ ]* 2.7 Write property test for paused contract rejection (Property 5)
    - **Property 5: Paused Contract Rejects All Mutating Message Operations**
    - **Validates: Requirements 1.8**

  - [ ]* 2.8 Write property test for overwrite idempotence (Property 6)
    - **Property 6: Overwrite Does Not Increment Message Count**
    - **Validates: Requirements 2.1, 2.3**

- [ ] 3. Implement `delete_message`
  - [ ] 3.1 Implement `delete_message` entry point in `lib.rs`
    - Assert not paused; load vault; require owner auth; assert vault is `Locked`
    - Check key exists; if not, return `MessageNotFound`
    - Remove `MessageData` entry; decrement `MessageCount`
    - Emit `MSG_DELETED_TOPIC` with `(vault_id, beneficiary)`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 3.2 Write unit tests for `delete_message`
    - `test_delete_message_removes_entry` — `has_message` returns false after delete
    - `test_delete_message_rejects_released_vault` — returns `AlreadyReleased`
    - `test_delete_message_rejects_missing_message` — returns `MessageNotFound`
    - `test_delete_message_emits_event` — emits `msg_del` topic
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 3.3 Write property test for delete removes message (Property 7)
    - **Property 7: Delete Removes Message**
    - **Validates: Requirements 3.1**

- [ ] 4. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Implement `get_message` and `has_message`
  - [ ] 5.1 Implement `get_message` entry point in `lib.rs`
    - Load vault (returns `VaultNotFound` if missing); assert vault status is `Released` (else `NotReleased`)
    - Load `MessageData`; return `MessageNotFound` if absent; return ciphertext bytes
    - No auth required
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 6.1, 8.1, 8.2, 8.3_

  - [ ] 5.2 Implement `has_message` entry point in `lib.rs`
    - Return `env.storage().persistent().has(&DataKey::MessageData(vault_id, beneficiary))`
    - No auth required, no vault status check; return `false` for nonexistent vault
    - _Requirements: 7.1, 7.2, 7.3_

  - [ ]* 5.3 Write unit tests for `get_message` and `has_message`
    - `test_get_message_returns_ciphertext_after_release` — full lifecycle test
    - `test_get_message_rejects_locked_vault` — returns `NotReleased`
    - `test_get_message_rejects_cancelled_vault` — returns `NotReleased`
    - `test_get_message_rejects_missing_message` — returns `MessageNotFound`
    - `test_get_message_rejects_nonexistent_vault` — returns `VaultNotFound`
    - `test_has_message_returns_false_for_nonexistent_vault` — returns false, no panic
    - `test_has_message_no_auth_required` — callable from any address
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 7.1, 7.2, 7.3_

  - [ ]* 5.4 Write property test for get_message on non-Released vault (Property 8)
    - **Property 8: get_message on Non-Released Vault Returns NotReleased**
    - **Validates: Requirements 4.2**

  - [ ]* 5.5 Write property test for read operations requiring no auth (Property 9)
    - **Property 9: Read Operations Require No Authorization**
    - **Validates: Requirements 4.4, 7.2**

  - [ ]* 5.6 Write property test for has_message reflecting storage state (Property 12)
    - **Property 12: has_message Accurately Reflects Storage State**
    - **Validates: Requirements 7.1**

- [ ] 6. Add lifecycle hooks to `check_in` and `cancel_vault`
  - [ ] 6.1 Extend `check_in` to refresh TTL of all message entries for the vault
    - After existing check_in logic, iterate `[vault.beneficiary] + vault.beneficiaries`
    - For each address, if `MessageData(vault_id, addr)` exists, call `extend_ttl` with `vault_ttl_ledgers(vault.check_in_interval)`
    - Also extend TTL of `MessageCount(vault_id)` if it exists
    - _Requirements: 5.2_

  - [ ] 6.2 Extend `cancel_vault` to remove all message entries for the vault
    - After marking vault `Cancelled`, iterate `[vault.beneficiary] + vault.beneficiaries`
    - Remove each `MessageData(vault_id, addr)` entry; remove `MessageCount(vault_id)`
    - _Requirements: 5.3_

  - [ ]* 6.3 Write unit tests for lifecycle hooks
    - `test_check_in_extends_message_ttl` — message TTL extended after check_in
    - `test_cancel_vault_removes_all_messages` — all messages gone after cancel
    - `test_release_retains_messages` — messages accessible after trigger_release
    - _Requirements: 5.2, 5.3, 5.4_

  - [ ]* 6.4 Write property test for cancel removing all messages (Property 10)
    - **Property 10: Cancel Vault Removes All Messages**
    - **Validates: Requirements 5.3**

  - [ ]* 6.5 Write property test for release retaining all messages (Property 11)
    - **Property 11: Release Vault Retains All Messages**
    - **Validates: Requirements 5.4**

- [ ] 7. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Property tests use the `proptest` crate; add `proptest = "1"` to `[dev-dependencies]` in `contracts/ttl_vault/Cargo.toml` if not already present
- Ciphertext is stored as `soroban_sdk::Bytes`; no interpretation of contents by the contract
- TTL for message entries mirrors vault TTL: `vault_ttl_ledgers(vault.check_in_interval)`
- Events carry `(vault_id, beneficiary)` only — never the ciphertext
