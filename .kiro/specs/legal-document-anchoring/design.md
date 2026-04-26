# Design Document: Legal Document Anchoring

## Overview

This feature extends the `TtlVaultContract` to allow vault owners to anchor legal documents — wills, trusts, powers of attorney — to a vault by storing a cryptographic hash and an off-chain reference on-chain. The contract never holds document content; it stores only a `BytesN<32>` SHA-256 digest and a UTF-8 reference string (IPFS CID or Stellar transaction hash). Anyone can later retrieve the anchor and independently verify a document against the stored hash.

The design adds five new entry points (`anchor_document`, `update_anchor`, `remove_anchor`, `get_anchor`, `list_anchors`, `verify_document`), two new `DataKey` variants, four new `ContractError` codes, two new event topics, and lifecycle hooks into `check_in` and `cancel_vault`.

## Architecture

```mermaid
sequenceDiagram
    participant Owner
    participant Contract as TtlVaultContract
    participant Storage as Persistent Storage

    Owner->>Contract: anchor_document(vault_id, doc_hash, reference, doc_type)
    Contract->>Contract: require_auth(owner), validate vault status, lengths, count
    Contract->>Storage: set(AnchorData(vault_id, anchor_id), DocumentAnchor)
    Contract->>Storage: set(AnchorCount(vault_id), count + 1)
    Contract->>Storage: extend_ttl(AnchorData, vault_ttl_ledgers(...))
    Contract-->>Owner: emit doc_anchored(vault_id, anchor_id, doc_hash)

    Note over Owner,Storage: Vault remains Locked; owner can check in

    Owner->>Contract: check_in(vault_id, caller)
    Contract->>Storage: extend_ttl(all AnchorData entries for vault)

    Note over Owner,Storage: Owner stops checking in; vault expires

    participant Anyone
    Anyone->>Contract: trigger_release(vault_id)
    Contract->>Storage: vault.status = Released (anchors retained)

    participant Verifier
    Verifier->>Contract: get_anchor(vault_id, anchor_id)
    Contract->>Storage: get(AnchorData(vault_id, anchor_id))
    Contract-->>Verifier: DocumentAnchor { doc_hash, reference, doc_type }

    Verifier->>Contract: verify_document(vault_id, anchor_id, candidate_hash)
    Contract-->>Verifier: true / false
```

### Storage Layout

Two new `DataKey` variants are added:

| Key | Type | Description |
|-----|------|-------------|
| `AnchorData(vault_id: u64, anchor_id: u32)` | `DocumentAnchor` | The anchor record for a specific (vault, anchor) pair |
| `AnchorCount(vault_id: u64)` | `u32` | Monotonically incrementing counter; also tracks total live anchors |

Both use **persistent storage** with TTL derived from `vault_ttl_ledgers(check_in_interval)`, matching the vault entry's TTL policy.

## Components and Interfaces

### New Contract Entry Points

```rust
/// Anchors a legal document to a vault by storing its hash and off-chain reference.
/// Requires owner auth. Vault must be Locked.
pub fn anchor_document(
    env: Env,
    vault_id: u64,
    doc_hash: BytesN<32>,
    reference: String,
    doc_type: String,
) -> Result<u32, ContractError>

/// Overwrites an existing anchor's hash, reference, and doc_type.
/// Requires owner auth. Vault must be Locked.
pub fn update_anchor(
    env: Env,
    vault_id: u64,
    anchor_id: u32,
    doc_hash: BytesN<32>,
    reference: String,
    doc_type: String,
) -> Result<(), ContractError>

/// Removes an anchor from a vault.
/// Requires owner auth. Vault must be Locked.
pub fn remove_anchor(
    env: Env,
    vault_id: u64,
    anchor_id: u32,
) -> Result<(), ContractError>

/// Returns the stored DocumentAnchor. No auth required. Works for any vault status.
pub fn get_anchor(
    env: Env,
    vault_id: u64,
    anchor_id: u32,
) -> Result<DocumentAnchor, ContractError>

/// Returns all DocumentAnchors for a vault in ascending anchor_id order.
/// No auth required.
pub fn list_anchors(
    env: Env,
    vault_id: u64,
) -> Result<Vec<DocumentAnchor>, ContractError>

/// Returns true if candidate_hash matches the stored doc_hash for the anchor.
/// No auth required.
pub fn verify_document(
    env: Env,
    vault_id: u64,
    anchor_id: u32,
    candidate_hash: BytesN<32>,
) -> Result<bool, ContractError>
```

### Modified Entry Points

- `check_in` — after saving the vault, iterates over all anchor IDs (0..AnchorCount) and calls `extend_ttl` on each live `AnchorData` key.
- `cancel_vault` — after marking the vault `Cancelled`, removes all `AnchorData` entries and the `AnchorCount` entry for the vault.

### New Event Topics

Added to `types.rs`:

```rust
pub const DOC_ANCHORED_TOPIC: Symbol = symbol_short!("doc_anch");
pub const DOC_UPDATED_TOPIC:  Symbol = symbol_short!("doc_upd");
pub const DOC_REMOVED_TOPIC:  Symbol = symbol_short!("doc_rem");
```

Events carry `(vault_id, anchor_id, doc_hash)` for anchor/update, and `(vault_id, anchor_id)` for remove — never the reference or document content.

### New Error Codes

Added to `ContractError`:

```rust
AnchorNotFound     = 25,  // no anchor for (vault_id, anchor_id)
AnchorLimitReached = 26,  // vault already has MAX_ANCHORS_PER_VAULT anchors
ReferenceTooLong   = 27,  // reference byte length > MAX_REFERENCE_LEN
DocTypeTooLong     = 28,  // doc_type byte length > MAX_DOC_TYPE_LEN
InvalidReference   = 29,  // empty reference string
InvalidDocType     = 30,  // empty doc_type string
```

## Data Models

### Constants

```rust
/// Maximum number of document anchors per vault.
pub const MAX_ANCHORS_PER_VAULT: u32 = 20;

/// Maximum byte length of a reference string (IPFS CID or Stellar tx hash).
pub const MAX_REFERENCE_LEN: u32 = 128;

/// Maximum byte length of a doc_type string.
pub const MAX_DOC_TYPE_LEN: u32 = 32;
```

### New `DocumentAnchor` Struct

```rust
#[contracttype]
#[derive(Clone)]
pub struct DocumentAnchor {
    /// Per-vault monotonically incrementing identifier.
    pub anchor_id: u32,
    /// SHA-256 digest of the document's canonical byte representation.
    pub doc_hash: BytesN<32>,
    /// Off-chain location: ipfs://{CID} or stellar://{TX_HASH}.
    pub reference: String,
    /// Document classification label, e.g. "will", "trust", "poa".
    pub doc_type: String,
    /// Unix timestamp when this anchor was created.
    pub anchored_at: u64,
}
```

### Updated `DataKey` Enum

```rust
#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    // ... existing variants ...
    AnchorData(u64, u32),   // (vault_id, anchor_id) → DocumentAnchor
    AnchorCount(u64),       // vault_id → u32 (next anchor_id / total count)
}
```

### Storage Access Patterns

**`anchor_document` logic:**

```
1. assert_not_paused
2. load vault; require owner auth
3. assert vault.status == Locked
4. assert reference.len() > 0 (else InvalidReference)
5. assert reference.len() <= MAX_REFERENCE_LEN (else ReferenceTooLong)
6. assert doc_type.len() > 0 (else InvalidDocType)
7. assert doc_type.len() <= MAX_DOC_TYPE_LEN (else DocTypeTooLong)
8. count = load AnchorCount(vault_id), default 0
9. assert count < MAX_ANCHORS_PER_VAULT (else AnchorLimitReached)
10. anchor_id = count  (0-indexed; count becomes next_id)
11. anchor = DocumentAnchor { anchor_id, doc_hash, reference, doc_type, anchored_at: now }
12. save AnchorData(vault_id, anchor_id) = anchor, extend TTL
13. save AnchorCount(vault_id) = count + 1, extend TTL
14. emit doc_anchored(vault_id, anchor_id, doc_hash)
15. return anchor_id
```

**`check_in` TTL extension addition:**

```
After existing check_in logic:
  let count = load AnchorCount(vault_id), default 0
  for anchor_id in 0..count:
    let key = DataKey::AnchorData(vault_id, anchor_id)
    if env.storage().persistent().has(&key):
      env.storage().persistent().extend_ttl(&key, VAULT_TTL_THRESHOLD, ttl)
  if count > 0:
    extend_ttl(AnchorCount(vault_id), ...)
```

**`cancel_vault` cleanup addition:**

```
After marking vault Cancelled:
  let count = load AnchorCount(vault_id), default 0
  for anchor_id in 0..count:
    env.storage().persistent().remove(&DataKey::AnchorData(vault_id, anchor_id))
  env.storage().persistent().remove(&DataKey::AnchorCount(vault_id))
```

**`list_anchors` logic:**

```
1. assert vault exists (else VaultNotFound)
2. count = load AnchorCount(vault_id), default 0
3. result = Vec::new()
4. for anchor_id in 0..count:
     if env.storage().persistent().has(&AnchorData(vault_id, anchor_id)):
       result.push(load AnchorData(vault_id, anchor_id))
5. return result  // ascending anchor_id order by construction
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Anchor Round-Trip

*For any* locked vault, any 32-byte hash, any non-empty reference string up to `MAX_REFERENCE_LEN` bytes, and any non-empty doc_type string up to `MAX_DOC_TYPE_LEN` bytes, calling `anchor_document` followed by `get_anchor` with the returned `anchor_id` shall return a `DocumentAnchor` whose `doc_hash` is byte-for-byte identical to the input hash and whose `reference` is character-for-character identical to the input reference.

**Validates: Requirements 1.1, 4.1, 8.2, 8.3, 9.1, 9.2, 9.5**

### Property 2: Mutating Operations Require Owner Authorization

*For any* vault and any caller that is not the vault owner, calling `anchor_document`, `update_anchor`, or `remove_anchor` shall fail with an authorization error.

**Validates: Requirements 1.2, 2.2, 3.2**

### Property 3: Mutations Rejected on Non-Locked Vaults

*For any* vault whose status is `Released` or `Cancelled`, calling `anchor_document`, `update_anchor`, or `remove_anchor` shall return `ContractError::AlreadyReleased`.

**Validates: Requirements 1.3, 2.3, 3.3**

### Property 4: Paused Contract Rejects All Mutating Anchor Operations

*For any* vault and any caller, when the contract is paused, calling `anchor_document`, `update_anchor`, or `remove_anchor` shall return `ContractError::Paused`.

**Validates: Requirements 1.4**

### Property 5: Reference Length Validation

*For any* reference string whose byte length exceeds `MAX_REFERENCE_LEN` (128), calling `anchor_document` or `update_anchor` shall return `ContractError::ReferenceTooLong`.

**Validates: Requirements 1.5**

### Property 6: Doc Type Length Validation

*For any* doc_type string whose byte length exceeds `MAX_DOC_TYPE_LEN` (32), calling `anchor_document` or `update_anchor` shall return `ContractError::DocTypeTooLong`.

**Validates: Requirements 1.6**

### Property 7: Anchor Limit Enforcement

*For any* vault that already holds `MAX_ANCHORS_PER_VAULT` (20) anchors, calling `anchor_document` shall return `ContractError::AnchorLimitReached`.

**Validates: Requirements 1.9**

### Property 8: Update Overwrites Anchor

*For any* locked vault with an existing anchor, calling `update_anchor` with new values followed by `get_anchor` shall return the new `doc_hash` and `reference`, not the original values.

**Validates: Requirements 2.1, 2.5**

### Property 9: Operations on Non-Existent Anchor Return AnchorNotFound

*For any* vault and any `anchor_id` that has not been created (or has been removed), calling `update_anchor`, `remove_anchor`, `get_anchor`, or `verify_document` shall return `ContractError::AnchorNotFound`.

**Validates: Requirements 2.4, 3.4, 4.3, 6.3**

### Property 10: Remove Anchor Deletes Record

*For any* locked vault with an existing anchor, after calling `remove_anchor`, calling `get_anchor` with the same `anchor_id` shall return `ContractError::AnchorNotFound`.

**Validates: Requirements 3.1**

### Property 11: Read Operations Require No Authorization

*For any* vault, any caller (including addresses unrelated to the vault), and any vault status, calling `get_anchor`, `list_anchors`, and `verify_document` shall succeed without requiring authorization from any specific address.

**Validates: Requirements 4.4, 5.4, 6.4**

### Property 12: list_anchors Returns Anchors in Ascending Order

*For any* vault with N anchors added in any order, `list_anchors` shall return exactly those anchors in ascending `anchor_id` order.

**Validates: Requirements 5.1**

### Property 13: verify_document Round-Trip

*For any* locked vault and any anchored `doc_hash`, calling `verify_document` with the same `doc_hash` shall return `true`, and calling it with any different 32-byte value shall return `false`.

**Validates: Requirements 6.1, 6.5**

### Property 14: Cancel Vault Removes All Anchors

*For any* vault with one or more stored anchors, after calling `cancel_vault`, `list_anchors` shall return an empty `Vec` (or `VaultNotFound` if the vault record is also removed).

**Validates: Requirements 7.3**

### Property 15: Release Vault Retains All Anchors

*For any* vault with one or more stored anchors, after `trigger_release` transitions the vault to `Released`, `get_anchor` shall return the original `DocumentAnchor` for every previously anchored `anchor_id`.

**Validates: Requirements 7.4**

### Property 16: Anchor TTL Extended on check_in

*For any* vault with stored anchors, after calling `check_in`, the persistent storage TTL of every `AnchorData` entry for that vault shall be at least as large as it was before the check-in.

**Validates: Requirements 7.1, 7.2**

## Error Handling

| Scenario | Error |
|----------|-------|
| Contract paused | `ContractError::Paused` |
| Vault does not exist | `ContractError::VaultNotFound` |
| Caller is not vault owner (for mutations) | `ContractError::NotOwner` (via `require_auth`) |
| Vault is not `Locked` (for anchor/update/remove) | `ContractError::AlreadyReleased` |
| `reference` is empty | `ContractError::InvalidReference` |
| `reference` exceeds 128 bytes | `ContractError::ReferenceTooLong` |
| `doc_type` is empty | `ContractError::InvalidDocType` |
| `doc_type` exceeds 32 bytes | `ContractError::DocTypeTooLong` |
| Vault already has 20 anchors | `ContractError::AnchorLimitReached` |
| `anchor_id` does not exist for vault | `ContractError::AnchorNotFound` |

All errors use `panic_with_error!` or `return Err(...)` consistent with existing contract patterns. All validations run before any state mutation.

## Testing Strategy

### Unit Tests

Unit tests cover specific examples, integration points, and all error conditions:

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
- `test_update_anchor_overwrites_values` — get_anchor returns new values after update
- `test_update_anchor_requires_owner_auth` — non-owner call panics
- `test_update_anchor_rejects_released_vault` — returns `AlreadyReleased`
- `test_update_anchor_rejects_missing_anchor` — returns `AnchorNotFound`
- `test_update_anchor_emits_event` — emits `doc_upd` topic with new hash
- `test_remove_anchor_deletes_entry` — get_anchor returns `AnchorNotFound` after remove
- `test_remove_anchor_requires_owner_auth` — non-owner call panics
- `test_remove_anchor_rejects_released_vault` — returns `AlreadyReleased`
- `test_remove_anchor_rejects_missing_anchor` — returns `AnchorNotFound`
- `test_remove_anchor_emits_event` — emits `doc_rem` topic
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
- `test_check_in_extends_anchor_ttl` — anchor TTL extended after check_in
- `test_cancel_vault_removes_all_anchors` — list_anchors returns empty after cancel
- `test_release_retains_anchors` — anchors accessible after trigger_release

### Property-Based Tests

Property tests use the `proptest` crate (`proptest = "1"` in `[dev-dependencies]`). Each test runs a minimum of 100 iterations.

Each test is tagged with a comment in the format:
`// Feature: legal-document-anchoring, Property N: <property_text>`

| Property | Test Name | Generator Strategy |
|----------|-----------|-------------------|
| P1: Round-trip | `prop_anchor_round_trip` | Random `BytesN<32>`, random valid reference (1–128 bytes), random valid doc_type (1–32 bytes) |
| P2: Owner auth required | `prop_mutations_require_owner_auth` | Random non-owner address, random valid anchor inputs |
| P3: Non-Locked rejection | `prop_mutations_rejected_on_non_locked` | Vault in Released or Cancelled state |
| P4: Paused rejection | `prop_paused_rejects_mutations` | Any vault, any caller |
| P5: Reference length | `prop_long_reference_rejected` | Random strings of length 129–256 bytes |
| P6: Doc type length | `prop_long_doc_type_rejected` | Random strings of length 33–64 bytes |
| P7: Anchor limit | `prop_anchor_limit_enforced` | Fill vault to 20 anchors, attempt 21st |
| P8: Update overwrites | `prop_update_overwrites_anchor` | Two random anchor payloads for same anchor_id |
| P9: AnchorNotFound | `prop_missing_anchor_returns_not_found` | Random anchor_id not in [0..count) |
| P10: Remove deletes | `prop_remove_deletes_anchor` | Random anchor, then remove, then get |
| P11: No auth for reads | `prop_reads_require_no_auth` | Random caller address, any vault status |
| P12: list_anchors order | `prop_list_anchors_ascending_order` | Add N (1–20) anchors, verify order |
| P13: verify_document round-trip | `prop_verify_document_round_trip` | Random hash anchored, verify with same and different hash |
| P14: Cancel removes anchors | `prop_cancel_removes_all_anchors` | Vault with 1–20 anchors |
| P15: Release retains anchors | `prop_release_retains_anchors` | Vault with 1–20 anchors |
| P16: check_in extends TTL | `prop_check_in_extends_anchor_ttl` | Vault with 1–20 anchors, then check_in |
