# Requirements Document

## Introduction

This feature adds testamentary message storage to the TTL-Legacy vault contract. Vault owners can attach encrypted messages intended for beneficiaries. These messages are stored on-chain (as opaque byte blobs) and become retrievable only after the vault is released — i.e., when the owner's check-in TTL expires and `trigger_release` is called. Encryption and decryption happen off-chain; the contract stores and gates access to the ciphertext.

## Glossary

- **Vault**: A time-locked asset container managed by the `TtlVaultContract`, identified by a `vault_id`.
- **Owner**: The Stellar address that created and controls a vault.
- **Beneficiary**: The Stellar address (or one of several) designated to receive vault assets upon release.
- **Message**: An encrypted byte blob stored by the Owner for a specific Beneficiary, associated with a vault.
- **Ciphertext**: The encrypted form of a message. The contract never sees or processes plaintext.
- **Message_Store**: The on-chain storage layer for vault messages, implemented as a new `DataKey` variant in the `TtlVaultContract`.
- **Message_Retrieval**: The contract function that returns stored ciphertext to an authorized caller after vault release.
- **Release**: The state transition of a vault from `Locked` to `Released`, triggered by `trigger_release` when the vault has expired.
- **Encryption_Scheme**: The off-chain cryptographic protocol used by clients to encrypt and decrypt messages. The contract is scheme-agnostic.
- **Nonce**: A unique value included in the ciphertext envelope to prevent replay attacks; managed off-chain.
- **MAX_MESSAGE_SIZE**: The maximum byte length of a single stored ciphertext, set to 4096 bytes.
- **MAX_MESSAGES_PER_VAULT**: The maximum number of messages that can be stored per vault, set to 10.

## Requirements

### Requirement 1: Store Encrypted Message

**User Story:** As a vault owner, I want to store an encrypted message for a beneficiary, so that I can leave instructions or personal notes that are revealed only when the vault is released.

#### Acceptance Criteria

1. WHEN an Owner calls `store_message` with a valid `vault_id`, a `beneficiary` address, and a non-empty `ciphertext` byte blob, THE Message_Store SHALL persist the ciphertext associated with that `(vault_id, beneficiary)` pair.
2. WHEN `store_message` is called, THE TtlVaultContract SHALL require authorization from the Owner of the specified vault.
3. WHEN `store_message` is called on a vault whose status is not `Locked`, THEN THE TtlVaultContract SHALL return `ContractError::AlreadyReleased`.
4. WHEN `store_message` is called with a `ciphertext` whose byte length exceeds `MAX_MESSAGE_SIZE` (4096 bytes), THEN THE TtlVaultContract SHALL return `ContractError::MessageTooLarge`.
5. WHEN `store_message` is called and the vault already has `MAX_MESSAGES_PER_VAULT` (10) messages stored, THEN THE TtlVaultContract SHALL return `ContractError::MessageLimitReached`.
6. WHEN `store_message` is called with an empty `ciphertext` (zero bytes), THEN THE TtlVaultContract SHALL return `ContractError::InvalidMessage`.
7. WHEN `store_message` is called with a `beneficiary` address that is not registered as a beneficiary of the specified vault, THEN THE TtlVaultContract SHALL return `ContractError::InvalidBeneficiary`.
8. WHEN `store_message` is called while the contract is paused, THEN THE TtlVaultContract SHALL return `ContractError::Paused`.
9. WHEN `store_message` succeeds, THE TtlVaultContract SHALL emit a `message_stored` event containing the `vault_id` and `beneficiary` address (but NOT the ciphertext).

### Requirement 2: Update Encrypted Message

**User Story:** As a vault owner, I want to replace a previously stored message for a beneficiary, so that I can revise my instructions while the vault is still active.

#### Acceptance Criteria

1. WHEN an Owner calls `store_message` for a `(vault_id, beneficiary)` pair that already has a stored message, THE Message_Store SHALL overwrite the existing ciphertext with the new one.
2. WHEN a message is overwritten, THE TtlVaultContract SHALL emit a `message_updated` event containing the `vault_id` and `beneficiary` address.
3. WHILE a vault is in `Locked` status, THE TtlVaultContract SHALL allow the Owner to call `store_message` any number of times for the same beneficiary without incrementing the stored message count.

### Requirement 3: Delete Encrypted Message

**User Story:** As a vault owner, I want to delete a stored message for a beneficiary, so that I can remove instructions I no longer want revealed.

#### Acceptance Criteria

1. WHEN an Owner calls `delete_message` with a valid `vault_id` and `beneficiary` address, THE Message_Store SHALL remove the ciphertext associated with that `(vault_id, beneficiary)` pair.
2. WHEN `delete_message` is called on a vault whose status is not `Locked`, THEN THE TtlVaultContract SHALL return `ContractError::AlreadyReleased`.
3. WHEN `delete_message` is called for a `(vault_id, beneficiary)` pair with no stored message, THEN THE TtlVaultContract SHALL return `ContractError::MessageNotFound`.
4. WHEN `delete_message` is called, THE TtlVaultContract SHALL require authorization from the Owner of the specified vault.
5. WHEN `delete_message` succeeds, THE TtlVaultContract SHALL emit a `message_deleted` event containing the `vault_id` and `beneficiary` address.

### Requirement 4: Retrieve Encrypted Message After Release

**User Story:** As a beneficiary, I want to retrieve the encrypted message left for me after the vault is released, so that I can decrypt and read the owner's final instructions.

#### Acceptance Criteria

1. WHEN a caller invokes `get_message` with a `vault_id` and a `beneficiary` address, and the vault status is `Released`, THE Message_Retrieval SHALL return the stored ciphertext for that `(vault_id, beneficiary)` pair.
2. WHEN `get_message` is called on a vault whose status is `Locked` or `Cancelled`, THEN THE TtlVaultContract SHALL return `ContractError::NotReleased`.
3. WHEN `get_message` is called for a `(vault_id, beneficiary)` pair with no stored message, THEN THE TtlVaultContract SHALL return `ContractError::MessageNotFound`.
4. THE Message_Retrieval SHALL NOT require authorization — any caller may read a released vault's message given the `vault_id` and `beneficiary` address.
5. WHEN `get_message` is called with a `vault_id` that does not exist, THEN THE TtlVaultContract SHALL return `ContractError::VaultNotFound`.

### Requirement 5: Message Storage Lifecycle and TTL

**User Story:** As a contract operator, I want message storage entries to follow the same TTL extension rules as vault data, so that messages remain accessible as long as the vault data is live.

#### Acceptance Criteria

1. WHEN a message is stored or updated via `store_message`, THE Message_Store SHALL extend the persistent storage TTL of the message entry using the same `vault_ttl_ledgers` calculation applied to vault data.
2. WHEN `check_in` is called on a vault, THE TtlVaultContract SHALL extend the TTL of all message entries associated with that vault by the same amount applied to the vault entry itself.
3. WHEN a vault is cancelled via `cancel_vault`, THE Message_Store SHALL remove all message entries associated with that vault.
4. WHEN a vault transitions to `Released` status, THE Message_Store SHALL retain all message entries so beneficiaries can retrieve them.

### Requirement 6: Message Encryption Scheme Documentation

**User Story:** As a client developer, I want a documented encryption scheme for vault messages, so that I can implement compatible encrypt/decrypt logic in the frontend or SDK.

#### Acceptance Criteria

1. THE TtlVaultContract SHALL store and return ciphertext as a raw `BytesN`-compatible byte blob without interpreting its contents.
2. THE TtlVaultContract SHALL accept ciphertext of any structure up to `MAX_MESSAGE_SIZE` bytes, making the contract encryption-scheme-agnostic.
3. WHERE a client implements message encryption, THE client SHALL be able to use any asymmetric or symmetric scheme whose output fits within `MAX_MESSAGE_SIZE` bytes.

### Requirement 7: Message Existence Check

**User Story:** As a client developer, I want to check whether a message exists for a given vault and beneficiary without retrieving the full ciphertext, so that I can build efficient UIs.

#### Acceptance Criteria

1. WHEN a caller invokes `has_message` with a `vault_id` and `beneficiary` address, THE TtlVaultContract SHALL return `true` if a message is stored for that pair, and `false` otherwise.
2. THE `has_message` function SHALL NOT require authorization and SHALL NOT check vault release status.
3. WHEN `has_message` is called with a `vault_id` that does not exist, THEN THE TtlVaultContract SHALL return `false`.

### Requirement 8: Message Serialization Round-Trip

**User Story:** As a contract developer, I want message ciphertext to survive serialization and deserialization through Soroban storage without corruption, so that beneficiaries receive exactly the bytes the owner stored.

#### Acceptance Criteria

1. THE Message_Store SHALL serialize ciphertext to persistent storage using Soroban's native XDR encoding.
2. FOR ALL valid ciphertext byte blobs stored via `store_message`, retrieving the same blob via `get_message` SHALL return a byte-for-byte identical value (round-trip property).
3. WHEN a ciphertext blob is stored and then retrieved, THE Message_Retrieval SHALL return a value whose byte length equals the byte length of the originally stored blob.
