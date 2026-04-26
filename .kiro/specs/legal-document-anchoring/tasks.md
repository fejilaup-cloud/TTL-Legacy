# Implementation Plan: Legal Document Anchoring

## Overview

Extend `TtlVaultContract` with six new entry points (`anchor_document`, `update_anchor`, `remove_anchor`, `get_anchor`, `list_anchors`, `verify_document`), two new `DataKey` variants, a new `DocumentAnchor` struct, six new `ContractError` codes, three new event topic constants, and lifecycle hooks in `check_in` and `cancel_vault`.

## Tasks

- [ ] 1. Add types, constants, and error codes
  - [ ] 1.1 Add `DocumentAnchor` struct and anchor constants to `types.rs`
    - Add `MAX_ANCHORS_PER_VAULT: u32 = 20`, `MAX_REFERENCE_LEN: u32 = 128`, `MAX_DOC_TYPE_LEN: u32 = 32` constants
    - Add `#[contracttype] DocumentAnchor` struct with fields: `anchor_id: u32`, `doc_hash: BytesN<32>`, `reference: String`, `doc_type: String`, `anchored_at: u64`
    - Add `DOC_ANCHORED_TOPIC`, `DOC_UPDATED_TOPIC`, `DOC_REMOVED_TOPIC` symbol constants using `symbol_short!`
    - _Requirements: 1.1, 1.10, 2.5, 3.5, 9.1, 9.2, 9.5_

  - [ ] 1.2 Add `AnchorData` and `AnchorCount` variants to `DataKey` enum in `types.rs`
    - Add `AnchorData(u64, u32)` (vault_id, anchor_id → `DocumentAnchor`) and `AnchorCount(u64)` (vault_id → `u32`) to the `DataKey` enum
    - _Requirements: 1.1, 7.1_

  - [ ] 1.3 Add new `ContractError` codes in `lib.rs`
    - Add `AnchorNotFound = 25`, `AnchorLimitReached = 26`, `ReferenceTooLong = 27`, `DocTypeTooLong = 28`, `InvalidReference = 29`, `InvalidDocType = 30`
    - _Requirements: 1.5, 1.6, 1.7, 1.8, 1.9, 2.4, 3.4, 4.3, 6.3_

  - [ ] 1.4 Update `use types::` import in `lib.rs` to include new symbols, types, and constants
    - Import `DocumentAnchor`, `DOC_ANCHORED_TOPIC`, `DOC_UPDATED_TOPIC`, `DOC_REMOVED_TOPIC`, `MAX_ANCHORS_PER_VAULT`, `MAX_REFERENCE_LEN`, `MAX_DOC_TYPE_LEN`
    - _Requirements: 1.1, 2.5, 3.5_

- [ ] 2. Implement `anchor_document`
  - [ ] 2.1 Implement `anchor_document` entry point in `lib.rs`
    - Assert not paused; load vault; require owner auth; assert vault status is `Locked`
    - Validate `reference` non-empty (`InvalidReference`), `<= MAX_REFERENCE_LEN` (`ReferenceTooLong`)
    - Validate `doc_type` non-empty (`InvalidDocType`), `<= MAX_DOC_TYPE_LEN` (`DocTypeTooLong`)
    - Load `AnchorCount(vault_id)` (default 0); assert `count < MAX_ANCHORS_PER_VAULT` (`AnchorLimitReached`)
    - Set `anchor_id = count`; build `DocumentAnchor { anchor_id, doc_hash, reference, doc_type, anchored_at: env.ledger().timestamp() }`
    - Save `AnchorData(vault_id, anchor_id)` and `AnchorCount(vault_id) = count + 1` with `extend_ttl` using `vault_ttl_ledgers(vault.check_in_interval)`
    - Emit `DOC_ANCHORED_TOPIC` event with `(vault_id, anchor_id, doc_hash)` — no reference in event
    - Return `anchor_id`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10, 7.1_

  - [ ]* 2.2 Write unit tests for `anchor_document`
    - `test_anchor_document_persists_anchor` — anchor then get returns same hash and reference
    - `test_anchor_document_requires_owner_auth` — non-owner call panics
    - `test_anchor_document_rejects_released_vault` — returns `AlreadyReleased`
    - `test_anchor_document_rejects_cancelled_vault` — returns `AlreadyReleased`
    - `test_anchor_document_rejects_empty_reference` — returns `InvalidReference`
    - `test_anchor_document_rejects_empty_doc_type` — returns `InvalidDocType`
    - `test_anchor_document_rejects_long_reference` — returns `ReferenceTooLong` at 129 bytes
    - `test_anchor_document_rejects_long_doc_type` — returns `DocTypeTooLong` at 33 bytes
    - `test_anchor_document_rejects_at_limit` — 20 anchors stored, 21st returns `AnchorLimitReached`
    - `test_anchor_document_rejects_when_paused` — returns `Paused`
    - `test_anchor_document_emits_event_without_reference` — event contains vault_id, anchor_id, doc_hash only
    - `test_anchor_document_assigns_sequential_ids` — first anchor gets id 0, second gets id 1
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10_

  - [ ]* 2.3 Write property test for anchor round-trip (Property 1)
    - **Property 1: Anchor Round-Trip**
    - **Validates: Requirements 1.1, 4.1, 8.2, 8.3, 9.1, 9.2, 9.5**

  - [ ]* 2.4 Write property test for owner auth requirement (Property 2)
    - **Property 2: Mutating Operations Require Owner Authorization**
    - **Validates: Requirements 1.2, 2.2, 3.2**

  - [ ]* 2.5 Write property test for non-Locked vault rejection (Property 3)
    - **Property 3: Mutations Rejected on Non-Locked Vaults**
    - **Validates: Requirements 1.3, 2.3, 3.3**

  - [ ]* 2.6 Write property test for paused contract rejection (Property 4)
    - **Property 4: Paused Contract Rejects All Mutating Anchor Operations**
    - **Validates: Requirements 1.4**

  - [ ]* 2.7 Write property test for reference length validation (Property 5)
    - **Property 5: Reference Length Validation**
    - **Validates: Requirements 1.5**

  - [ ]* 2.8 Write property test for doc type length validation (Property 6)
    - **Property 6: Doc Type Length Validation**
    - **Validates: Requirements 1.6**

  - [ ]* 2.9 Write property test for anchor limit enforcement (Property 7)
    - **Property 7: Anchor Limit Enforcement**
    - **Validates: Requirements 1.9**

- [ ] 3. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement `update_anchor` and `remove_anchor`
  - [ ] 4.1 Implement `update_anchor` entry point in `lib.rs`
    - Assert not paused; load vault; require owner auth; assert vault status is `Locked`
    - Apply same `reference` and `doc_type` length/empty validations as `anchor_document`
    - Load `AnchorData(vault_id, anchor_id)`; return `AnchorNotFound` if absent
    - Overwrite with new `doc_hash`, `reference`, `doc_type` (preserve `anchor_id` and `anchored_at`); extend TTL
    - Emit `DOC_UPDATED_TOPIC` event with `(vault_id, anchor_id, new_doc_hash)`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ] 4.2 Implement `remove_anchor` entry point in `lib.rs`
    - Assert not paused; load vault; require owner auth; assert vault status is `Locked`
    - Check `AnchorData(vault_id, anchor_id)` exists; return `AnchorNotFound` if absent
    - Remove `AnchorData(vault_id, anchor_id)` from persistent storage
    - Emit `DOC_REMOVED_TOPIC` event with `(vault_id, anchor_id)`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 4.3 Write unit tests for `update_anchor`
    - `test_update_anchor_overwrites_values` — get_anchor returns new values after update
    - `test_update_anchor_requires_owner_auth` — non-owner call panics
    - `test_update_anchor_rejects_released_vault` — returns `AlreadyReleased`
    - `test_update_anchor_rejects_missing_anchor` — returns `AnchorNotFound`
    - `test_update_anchor_emits_event` — emits `doc_upd` topic with new hash
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 4.4 Write unit tests for `remove_anchor`
    - `test_remove_anchor_deletes_entry` — get_anchor returns `AnchorNotFound` after remove
    - `test_remove_anchor_requires_owner_auth` — non-owner call panics
    - `test_remove_anchor_rejects_released_vault` — returns `AlreadyReleased`
    - `test_remove_anchor_rejects_missing_anchor` — returns `AnchorNotFound`
    - `test_remove_anchor_emits_event` — emits `doc_rem` topic
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ]* 4.5 Write property test for update overwrites anchor (Property 8)
    - **Property 8: Update Overwrites Anchor**
    - **Validates: Requirements 2.1, 2.5**

  - [ ]* 4.6 Write property test for AnchorNotFound on missing anchor (Property 9)
    - **Property 9: Operations on Non-Existent Anchor Return AnchorNotFound**
    - **Validates: Requirements 2.4, 3.4, 4.3, 6.3**

  - [ ]* 4.7 Write property test for remove deletes anchor (Property 10)
    - **Property 10: Remove Anchor Deletes Record**
    - **Validates: Requirements 3.1**

- [ ] 5. Implement `get_anchor`, `list_anchors`, and `verify_document`
  - [ ] 5.1 Implement `get_anchor` entry point in `lib.rs`
    - Assert vault exists via `try_load_vault`; return `VaultNotFound` if absent
    - Load `AnchorData(vault_id, anchor_id)`; return `AnchorNotFound` if absent
    - Return `DocumentAnchor` — no auth required, works for any vault status
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ] 5.2 Implement `list_anchors` entry point in `lib.rs`
    - Assert vault exists; return `VaultNotFound` if absent
    - Load `AnchorCount(vault_id)` (default 0); iterate `0..count`
    - For each `anchor_id`, if `AnchorData(vault_id, anchor_id)` exists, push to result `Vec`
    - Return `Vec<DocumentAnchor>` in ascending `anchor_id` order — no auth required
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [ ] 5.3 Implement `verify_document` entry point in `lib.rs`
    - Assert vault exists; return `VaultNotFound` if absent
    - Load `AnchorData(vault_id, anchor_id)`; return `AnchorNotFound` if absent
    - Return `candidate_hash == anchor.doc_hash` — no auth required
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ]* 5.4 Write unit tests for `get_anchor`, `list_anchors`, and `verify_document`
    - `test_get_anchor_returns_anchor_after_release` — anchor accessible after trigger_release
    - `test_get_anchor_rejects_nonexistent_vault` — returns `VaultNotFound`
    - `test_get_anchor_rejects_missing_anchor` — returns `AnchorNotFound`
    - `test_get_anchor_no_auth_required` — callable from any address
    - `test_list_anchors_returns_all_in_order` — 3 anchors returned in ascending id order
    - `test_list_anchors_empty_vault` — returns empty Vec
    - `test_list_anchors_rejects_nonexistent_vault` — returns `VaultNotFound`
    - `test_verify_document_returns_true_for_matching_hash` — exact match returns true
    - `test_verify_document_returns_false_for_wrong_hash` — different hash returns false
    - `test_verify_document_rejects_nonexistent_vault` — returns `VaultNotFound`
    - `test_verify_document_rejects_missing_anchor` — returns `AnchorNotFound`
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 5.1, 5.2, 5.3, 5.4, 6.1, 6.2, 6.3, 6.4_

  - [ ]* 5.5 Write property test for read operations requiring no auth (Property 11)
    - **Property 11: Read Operations Require No Authorization**
    - **Validates: Requirements 4.4, 5.4, 6.4**

  - [ ]* 5.6 Write property test for list_anchors ascending order (Property 12)
    - **Property 12: list_anchors Returns Anchors in Ascending Order**
    - **Validates: Requirements 5.1**

  - [ ]* 5.7 Write property test for verify_document round-trip (Property 13)
    - **Property 13: verify_document Round-Trip**
    - **Validates: Requirements 6.1, 6.5**

- [ ] 6. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Add lifecycle hooks to `check_in` and `cancel_vault`
  - [ ] 7.1 Extend `check_in` to refresh TTL of all anchor entries for the vault
    - After existing check_in logic, load `AnchorCount(vault_id)` (default 0)
    - For each `anchor_id` in `0..count`, if `AnchorData(vault_id, anchor_id)` exists, call `extend_ttl` with `vault_ttl_ledgers(vault.check_in_interval)`
    - Also extend TTL of `AnchorCount(vault_id)` if count > 0
    - _Requirements: 7.1, 7.2_

  - [ ] 7.2 Extend `cancel_vault` to remove all anchor entries for the vault
    - After marking vault `Cancelled`, load `AnchorCount(vault_id)` (default 0)
    - Remove each `AnchorData(vault_id, anchor_id)` for `anchor_id` in `0..count`
    - Remove `AnchorCount(vault_id)`
    - _Requirements: 7.3_

  - [ ]* 7.3 Write unit tests for lifecycle hooks
    - `test_check_in_extends_anchor_ttl` — anchor TTL extended after check_in
    - `test_cancel_vault_removes_all_anchors` — list_anchors returns empty after cancel
    - `test_release_retains_anchors` — anchors accessible after trigger_release
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

  - [ ]* 7.4 Write property test for cancel removing all anchors (Property 14)
    - **Property 14: Cancel Vault Removes All Anchors**
    - **Validates: Requirements 7.3**

  - [ ]* 7.5 Write property test for release retaining all anchors (Property 15)
    - **Property 15: Release Vault Retains All Anchors**
    - **Validates: Requirements 7.4**

  - [ ]* 7.6 Write property test for check_in extending anchor TTL (Property 16)
    - **Property 16: Anchor TTL Extended on check_in**
    - **Validates: Requirements 7.1, 7.2**

- [ ] 8. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Property tests use the `proptest` crate; add `proptest = "1"` to `[dev-dependencies]` in `contracts/ttl_vault/Cargo.toml` if not already present
- `AnchorCount` is a monotonically incrementing counter — removed anchors leave gaps in the ID space; `list_anchors` skips absent entries via `has()` check
- TTL for anchor entries mirrors vault TTL: `vault_ttl_ledgers(vault.check_in_interval)`
- Events carry `(vault_id, anchor_id, doc_hash)` for anchor/update, `(vault_id, anchor_id)` for remove — never the reference or document content
- `update_anchor` preserves the original `anchored_at` timestamp; only hash, reference, and doc_type are overwritten
