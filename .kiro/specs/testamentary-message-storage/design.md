# Design Document: Testamentary Message Storage

## Overview

This feature extends the `TtlVaultContract` to allow vault owners to store encrypted messages for beneficiaries. Messages are opaque byte blobs stored on-chain and gated by vault release status — only retrievable after `trigger_release` transitions the vault to `Released`. Encryption and decryption are entirely off-chain; the contract is scheme-agnostic.

The design adds three new entry points (`store_message`, `delete_message`, `get_message`, `has_message`), two new `DataKey` variants, three new `ContractError` codes, and lifecycle hooks into `check_in` and `cancel_vault`.

## Architecture

```mermaid
sequenceDiagram
    participant Owner
    participant Contract as TtlVaultContract
    participant Storage as Persistent Storage

    Owner->>Contract: store_message(vault_id, beneficiary, ciphertext)
    Contract->>Contract: require_auth(owner), validate vault status, size, count
    Contract->>Storage: set(MessageData(vault_id, beneficiary), ciphertext)
    Contract->>Storage: extend_ttl(MessageCount(vault_id), ...)
    Contract-->>Owner: emit message_stored / message_updated event

    Note over Owner,Storage: Vault remains Locked; owner can check in

    Owner->>Contract: check_in(vault_id, caller)
    Contract->>Storage: extend_ttl(all MessageData entries for vault)

    Note over Owner,Storage: Owner stops checking in; vault expires

    participant Anyone
    Anyone->>Contract: trigger_release(vault_id)
    Contract->>Storage: vault.status = Released (messages retained)

    participant Beneficiary
    Beneficiary->>Contract: get_message(vault_id, beneficiary)
    Contract->>Storage: get(MessageData(vault_id, beneficiary))
    Contract-->>Beneficiary: ciphertext bytes
```

### Storage Layout

Two new `DataKey` variants are added:

| Key | Type | Description |
|-----|------|-------------|
| `MessageData(vault_id: u64, beneficiary: Address)` | `Bytes` | The ciphertext blob for a specific (vault, beneficiary) pair |
| `MessageCount(vault_id: u64)` | `u32` | Number of distinct beneficiary messages stored for a vault |

Both use **persistent storage** with TTL derived from `vault_ttl_ledgers(check_in_interval)`, matching the vault entry's TTL policy.

## Components and Interfaces

### New Contract Entry Points

```rust
/// Stores or overwrites an encrypted message for a beneficiary.
/// Requires owner auth. Vault must be Locked.
pub fn store_message(
    env: Env,
    vault_id: u64,
    beneficiary: Address,
    ciphertext: Bytes,
) -> Result<(), ContractError>

/// Deletes a stored message for a beneficiary.
/// Requires owner auth. Vault must be Locked.
pub fn delete_message(
    env: Env,
    vault_id: u64,
    beneficiary: Address,
) -> Result<(), ContractError>

/// Returns the stored ciphertext. No auth required. Vault must be Released.
pub fn get_message(
    env: Env,
    vault_id: u64,
    beneficiary: Address,
) -> Result<Bytes, ContractError>

/// Returns true if a message exists for the pair. No auth, no status check.
pub fn has_message(
    env: Env,
    vault_id: u64,
    beneficiary: Address,
) -> bool
```

### Modified Entry Points

- `check_in` — after saving the vault, iterates over all beneficiaries with stored messages and calls `extend_ttl` on each `MessageData` key.
- `cancel_vault` — after marking the vault `Cancelled`, removes all `MessageData` entries and the `MessageCount` entry for the vault.

### New Event Topics

Added to `types.rs`:

```rust
pub const MSG_STORED_TOPIC: Symbol  = symbol_short!("msg_store");
pub const MSG_UPDATED_TOPIC: Symbol = symbol_short!("msg_upd");
pub const MSG_DELETED_TOPIC: Symbol = symbol_short!("msg_del");
```

Events carry `(vault_id, beneficiary)` — never the ciphertext.

### New Error Codes

Added to `ContractError`:

```rust
MessageTooLarge     = 25,  // ciphertext > MAX_MESSAGE_SIZE
MessageLimitReached = 26,  // vault already has MAX_MESSAGES_PER_VAULT messages
InvalidMessage      = 27,  // empty ciphertext
MessageNotFound     = 28,  // no message for (vault_id, beneficiary)
NotReleased         = 29,  // vault is not Released (for get_message)
```

## Data Models

### Constants

```rust
/// Maximum byte length of a single stored ciphertext.
pub const MAX_MESSAGE_SIZE: u32 = 4096;

/// Maximum number of distinct beneficiary messages per vault.
pub const MAX_MESSAGES_PER_VAULT: u32 = 10;
```

### Updated `DataKey` Enum

```rust
#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    // ... existing variants ...
    MessageData(u64, Address),   // (vault_id, beneficiary) → Bytes
    MessageCount(u64),           // vault_id → u32
}
```

### Storage Access Pattern

`store_message` logic:

```
1. assert_not_paused
2. load vault; require owner auth
3. assert vault.status == Locked
4. assert ciphertext.len() > 0 (else InvalidMessage)
5. assert ciphertext.len() <= MAX_MESSAGE_SIZE (else MessageTooLarge)
6. assert beneficiary is in vault.beneficiary or vault.beneficiaries (else InvalidBeneficiary)
7. let key = DataKey::MessageData(vault_id, beneficiary)
8. let is_new = !env.storage().persistent().has(&key)
9. if is_new:
     count = load MessageCount(vault_id), default 0
     assert count < MAX_MESSAGES_PER_VAULT (else MessageLimitReached)
     save MessageCount(vault_id, count + 1)
10. save MessageData(vault_id, beneficiary) = ciphertext, extend TTL
11. emit msg_stored (is_new) or msg_updated (!is_new)
```

`check_in` TTL extension addition:

```
After existing check_in logic:
  for each beneficiary in [vault.beneficiary] + vault.beneficiaries:
    let key = DataKey::MessageData(vault_id, beneficiary.address)
    if env.storage().persistent().has(&key):
      env.storage().persistent().extend_ttl(&key, VAULT_TTL_THRESHOLD, ttl)
  extend_ttl(MessageCount(vault_id), ...)
```

`cancel_vault` cleanup addition:

```
After marking vault Cancelled:
  for each beneficiary in [vault.beneficiary] + vault.beneficiaries:
    let key = DataKey::MessageData(vault_id, beneficiary.address)
    env.storage().persistent().remove(&key)
  env.storage().persistent().remove(&DataKey::MessageCount(vault_id))
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Message Round-Trip

*For any* locked vault, registered beneficiary, and non-empty ciphertext of at most `MAX_MESSAGE_SIZE` bytes, storing the ciphertext via `store_message` and then retrieving it via `get_message` (after release) shall return a byte-for-byte identical value.

**Validates: Requirements 1.1, 4.1, 6.1, 6.2, 8.2**

### Property 2: Mutating Operations Require Owner Authorization

*For any* vault and any caller that is not the vault owner, calling `store_message` or `delete_message` shall fail with an authorization error.

**Validates: Requirements 1.2, 3.4**

### Property 3: Message Mutations Rejected on Non-Locked Vaults

*For any* vault whose status is `Released` or `Cancelled`, calling `store_message` or `delete_message` shall return `ContractError::AlreadyReleased`.

**Validates: Requirements 1.3, 3.2**

### Property 4: Invalid Beneficiary Rejected

*For any* vault and any address that is not registered as a beneficiary of that vault, calling `store_message` shall return `ContractError::InvalidBeneficiary`.

**Validates: Requirements 1.7**

### Property 5: Paused Contract Rejects All Mutating Message Operations

*For any* vault and any caller, when the contract is paused, calling `store_message` or `delete_message` shall return `ContractError::Paused`.

**Validates: Requirements 1.8**

### Property 6: Overwrite Does Not Increment Message Count

*For any* locked vault with an existing message for a beneficiary, calling `store_message` again for the same `(vault_id, beneficiary)` pair shall not increase the stored message count, and `get_message` (after release) shall return the new ciphertext.

**Validates: Requirements 2.1, 2.3**

### Property 7: Delete Removes Message

*For any* locked vault with a stored message for a beneficiary, after calling `delete_message`, `has_message` shall return `false` for that `(vault_id, beneficiary)` pair.

**Validates: Requirements 3.1**

### Property 8: get_message on Non-Released Vault Returns NotReleased

*For any* vault whose status is `Locked` or `Cancelled`, calling `get_message` shall return `ContractError::NotReleased`.

**Validates: Requirements 4.2**

### Property 9: Read Operations Require No Authorization

*For any* vault, any caller (including addresses unrelated to the vault), and any vault status, calling `get_message` (on a released vault) or `has_message` shall succeed without requiring authorization from any specific address.

**Validates: Requirements 4.4, 7.2**

### Property 10: Cancel Vault Removes All Messages

*For any* vault with one or more stored messages, after calling `cancel_vault`, `has_message` shall return `false` for every beneficiary that previously had a message stored.

**Validates: Requirements 5.3**

### Property 11: Release Vault Retains All Messages

*For any* vault with one or more stored messages, after `trigger_release` transitions the vault to `Released`, `get_message` shall return the original ciphertext for every beneficiary that had a message stored.

**Validates: Requirements 5.4**

### Property 12: has_message Accurately Reflects Storage State

*For any* vault and beneficiary, `has_message` shall return `true` if and only if a message has been stored via `store_message` and not subsequently deleted via `delete_message`.

**Validates: Requirements 7.1**

## Error Handling

| Scenario | Error |
|----------|-------|
| Contract paused | `ContractError::Paused` |
| Vault does not exist | `ContractError::VaultNotFound` |
| Caller is not vault owner (for mutations) | `ContractError::NotOwner` (via `require_auth`) |
| Vault is not `Locked` (for store/delete) | `ContractError::AlreadyReleased` |
| Vault is not `Released` (for get_message) | `ContractError::NotReleased` |
| Ciphertext is empty | `ContractError::InvalidMessage` |
| Ciphertext exceeds 4096 bytes | `ContractError::MessageTooLarge` |
| Vault already has 10 messages | `ContractError::MessageLimitReached` |
| No message for (vault_id, beneficiary) | `ContractError::MessageNotFound` |
| Beneficiary not registered on vault | `ContractError::InvalidBeneficiary` |

All errors use `panic_with_error!` or `return Err(...)` consistent with existing contract patterns. No partial state mutations occur before validation is complete.

## Testing Strategy

### Unit Tests

Unit tests cover specific examples, integration points, and all error conditions:

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
- `test_delete_message_removes_entry` — `has_message` returns false after delete
- `test_delete_message_rejects_released_vault` — returns `AlreadyReleased`
- `test_delete_message_rejects_missing_message` — returns `MessageNotFound`
- `test_delete_message_emits_event` — emits `msg_del` topic
- `test_get_message_returns_ciphertext_after_release` — full lifecycle test
- `test_get_message_rejects_locked_vault` — returns `NotReleased`
- `test_get_message_rejects_cancelled_vault` — returns `NotReleased`
- `test_get_message_rejects_missing_message` — returns `MessageNotFound`
- `test_get_message_rejects_nonexistent_vault` — returns `VaultNotFound`
- `test_has_message_returns_false_for_nonexistent_vault` — returns false, no panic
- `test_has_message_no_auth_required` — callable from any address
- `test_check_in_extends_message_ttl` — message TTL extended after check_in
- `test_cancel_vault_removes_all_messages` — all messages gone after cancel
- `test_release_retains_messages` — messages accessible after trigger_release

### Property-Based Tests

Property tests use the `proptest` crate (already available in the Rust ecosystem for Soroban test environments). Each test runs a minimum of 100 iterations.

**Property test library**: `proptest` (via `proptest = "1"` in `[dev-dependencies]`)

Each test is tagged with a comment in the format:
`// Feature: testamentary-message-storage, Property N: <property_text>`

| Property | Test Name | Generator Strategy |
|----------|-----------|-------------------|
| P1: Round-trip | `prop_message_round_trip` | Random `vault_id`, random `Bytes` 1–4096 len |
| P2: Owner auth required | `prop_store_delete_require_owner_auth` | Random non-owner address |
| P3: Non-Locked rejection | `prop_mutations_rejected_on_non_locked` | Vault in Released or Cancelled state |
| P4: Invalid beneficiary | `prop_invalid_beneficiary_rejected` | Random address not in vault beneficiary set |
| P5: Paused rejection | `prop_paused_rejects_mutations` | Any vault, any caller |
| P6: Overwrite idempotence | `prop_overwrite_does_not_increment_count` | Two random ciphertexts for same pair |
| P7: Delete removes message | `prop_delete_removes_message` | Random stored message, then delete |
| P8: get_message on non-Released | `prop_get_message_non_released_fails` | Locked or Cancelled vault |
| P9: No auth for reads | `prop_reads_require_no_auth` | Random caller address on released vault |
| P10: Cancel removes messages | `prop_cancel_removes_all_messages` | Vault with 1–10 messages |
| P11: Release retains messages | `prop_release_retains_messages` | Vault with 1–10 messages |
| P12: has_message reflects state | `prop_has_message_reflects_state` | Random store/delete sequence |
