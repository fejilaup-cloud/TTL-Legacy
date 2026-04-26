# Requirements Document

## Introduction

This feature adds legal document anchoring to the TTL-Legacy vault contract. Vault owners can anchor legal documents — such as wills, trusts, or powers of attorney — to a vault by storing a cryptographic hash of the document on-chain. The document itself is stored off-chain, either referenced via an IPFS content identifier (CID) or via a Stellar transaction memo. The contract stores the hash and reference, enabling anyone to verify that a retrieved document matches what the owner anchored, without the contract ever holding the document content itself.

## Glossary

- **Vault**: A time-locked asset container managed by the `TtlVaultContract`, identified by a `vault_id`.
- **Owner**: The Stellar address that created and controls a vault.
- **Beneficiary**: The Stellar address (or one of several) designated to receive vault assets upon release.
- **Document_Anchor**: An on-chain record associating a vault with a legal document. Contains a `doc_hash`, a `reference`, and a `doc_type`.
- **Document_Hash**: A 32-byte SHA-256 digest of the legal document's canonical byte representation, stored as `BytesN<32>`.
- **Reference**: A UTF-8 string pointing to the off-chain document location. Either an IPFS CID (e.g. `ipfs://Qm...`) or a Stellar transaction hash (e.g. `stellar://TXHASH`). Maximum length: `MAX_REFERENCE_LEN` (128 bytes).
- **Doc_Type**: A short UTF-8 label classifying the document kind (e.g. `"will"`, `"trust"`, `"poa"`). Maximum length: `MAX_DOC_TYPE_LEN` (32 bytes).
- **Document_Store**: The on-chain storage layer for document anchors, implemented as new `DataKey` variants in the `TtlVaultContract`.
- **Document_Retrieval**: The contract function that returns a stored `Document_Anchor` to any caller.
- **MAX_ANCHORS_PER_VAULT**: The maximum number of document anchors that can be stored per vault, set to 20.
- **MAX_REFERENCE_LEN**: Maximum byte length of a `Reference` string, set to 128.
- **MAX_DOC_TYPE_LEN**: Maximum byte length of a `Doc_Type` string, set to 32.
- **IPFS**: InterPlanetary File System — a content-addressed distributed storage protocol. A CID uniquely identifies content by its hash.
- **Anchor_Id**: A per-vault monotonically incrementing `u32` identifier assigned to each `Document_Anchor` at creation time.

## Requirements

### Requirement 1: Anchor a Legal Document

**User Story:** As a vault owner, I want to anchor a legal document to my vault by storing its hash and an off-chain reference, so that beneficiaries and verifiers can confirm the document's authenticity after the vault is released.

#### Acceptance Criteria

1. WHEN an Owner calls `anchor_document` with a valid `vault_id`, a 32-byte `doc_hash`, a non-empty `reference` string, and a non-empty `doc_type` string, THE Document_Store SHALL persist a `Document_Anchor` record associated with that vault and assign it a unique `anchor_id`.
2. WHEN `anchor_document` is called, THE TtlVaultContract SHALL require authorization from the Owner of the specified vault.
3. WHEN `anchor_document` is called on a vault whose status is not `Locked`, THEN THE TtlVaultContract SHALL return `ContractError::AlreadyReleased`.
4. WHEN `anchor_document` is called while the contract is paused, THEN THE TtlVaultContract SHALL return `ContractError::Paused`.
5. WHEN `anchor_document` is called with a `reference` string whose byte length exceeds `MAX_REFERENCE_LEN` (128 bytes), THEN THE TtlVaultContract SHALL return `ContractError::ReferenceTooLong`.
6. WHEN `anchor_document` is called with a `doc_type` string whose byte length exceeds `MAX_DOC_TYPE_LEN` (32 bytes), THEN THE TtlVaultContract SHALL return `ContractError::DocTypeTooLong`.
7. WHEN `anchor_document` is called with an empty `reference` string, THEN THE TtlVaultContract SHALL return `ContractError::InvalidReference`.
8. WHEN `anchor_document` is called with an empty `doc_type` string, THEN THE TtlVaultContract SHALL return `ContractError::InvalidDocType`.
9. WHEN `anchor_document` is called and the vault already has `MAX_ANCHORS_PER_VAULT` (20) anchors stored, THEN THE TtlVaultContract SHALL return `ContractError::AnchorLimitReached`.
10. WHEN `anchor_document` succeeds, THE TtlVaultContract SHALL emit a `doc_anchored` event containing the `vault_id`, `anchor_id`, and `doc_hash` (but NOT the `reference` or document content).

### Requirement 2: Update a Document Anchor

**User Story:** As a vault owner, I want to update the reference or type of a previously anchored document, so that I can correct a stale IPFS link or reclassify a document while the vault is still active.

#### Acceptance Criteria

1. WHEN an Owner calls `update_anchor` with a valid `vault_id`, a valid `anchor_id`, and new `doc_hash`, `reference`, and `doc_type` values, THE Document_Store SHALL overwrite the existing `Document_Anchor` with the new values.
2. WHEN `update_anchor` is called, THE TtlVaultContract SHALL require authorization from the Owner of the specified vault.
3. WHEN `update_anchor` is called on a vault whose status is not `Locked`, THEN THE TtlVaultContract SHALL return `ContractError::AlreadyReleased`.
4. WHEN `update_anchor` is called with an `anchor_id` that does not exist for the given vault, THEN THE TtlVaultContract SHALL return `ContractError::AnchorNotFound`.
5. WHEN `update_anchor` succeeds, THE TtlVaultContract SHALL emit a `doc_updated` event containing the `vault_id`, `anchor_id`, and new `doc_hash`.

### Requirement 3: Remove a Document Anchor

**User Story:** As a vault owner, I want to remove a document anchor from my vault, so that I can retract a document I no longer want associated with the vault.

#### Acceptance Criteria

1. WHEN an Owner calls `remove_anchor` with a valid `vault_id` and `anchor_id`, THE Document_Store SHALL delete the `Document_Anchor` associated with that `(vault_id, anchor_id)` pair.
2. WHEN `remove_anchor` is called, THE TtlVaultContract SHALL require authorization from the Owner of the specified vault.
3. WHEN `remove_anchor` is called on a vault whose status is not `Locked`, THEN THE TtlVaultContract SHALL return `ContractError::AlreadyReleased`.
4. WHEN `remove_anchor` is called with an `anchor_id` that does not exist for the given vault, THEN THE TtlVaultContract SHALL return `ContractError::AnchorNotFound`.
5. WHEN `remove_anchor` succeeds, THE TtlVaultContract SHALL emit a `doc_removed` event containing the `vault_id` and `anchor_id`.

### Requirement 4: Retrieve a Document Anchor

**User Story:** As a beneficiary or verifier, I want to retrieve a document anchor by its vault and anchor ID, so that I can obtain the document hash and reference needed to fetch and verify the document.

#### Acceptance Criteria

1. WHEN a caller invokes `get_anchor` with a `vault_id` and `anchor_id`, THE Document_Retrieval SHALL return the stored `Document_Anchor` for that pair regardless of vault status.
2. WHEN `get_anchor` is called with a `vault_id` that does not exist, THEN THE TtlVaultContract SHALL return `ContractError::VaultNotFound`.
3. WHEN `get_anchor` is called with an `anchor_id` that does not exist for the given vault, THEN THE TtlVaultContract SHALL return `ContractError::AnchorNotFound`.
4. THE `get_anchor` function SHALL NOT require authorization — any caller may read an anchor given the `vault_id` and `anchor_id`.

### Requirement 5: List Document Anchors for a Vault

**User Story:** As a beneficiary or verifier, I want to list all document anchors associated with a vault, so that I can discover all legal documents the owner attached.

#### Acceptance Criteria

1. WHEN a caller invokes `list_anchors` with a `vault_id`, THE Document_Retrieval SHALL return a `Vec` of all `Document_Anchor` records stored for that vault, in ascending `anchor_id` order.
2. WHEN `list_anchors` is called on a vault with no anchors, THE Document_Retrieval SHALL return an empty `Vec`.
3. WHEN `list_anchors` is called with a `vault_id` that does not exist, THEN THE TtlVaultContract SHALL return `ContractError::VaultNotFound`.
4. THE `list_anchors` function SHALL NOT require authorization.

### Requirement 6: Verify a Document Against Its Anchor

**User Story:** As a verifier, I want to verify that a document's SHA-256 hash matches the stored anchor hash, so that I can confirm the document has not been tampered with since it was anchored.

#### Acceptance Criteria

1. WHEN a caller invokes `verify_document` with a `vault_id`, an `anchor_id`, and a 32-byte `candidate_hash`, THE TtlVaultContract SHALL return `true` if `candidate_hash` equals the stored `doc_hash` for that anchor, and `false` otherwise.
2. WHEN `verify_document` is called with a `vault_id` that does not exist, THEN THE TtlVaultContract SHALL return `ContractError::VaultNotFound`.
3. WHEN `verify_document` is called with an `anchor_id` that does not exist for the given vault, THEN THE TtlVaultContract SHALL return `ContractError::AnchorNotFound`.
4. THE `verify_document` function SHALL NOT require authorization.
5. FOR ALL valid `Document_Anchor` records, calling `verify_document` with the stored `doc_hash` SHALL return `true` (round-trip property).

### Requirement 7: Document Anchor Storage Lifecycle and TTL

**User Story:** As a contract operator, I want document anchor storage entries to follow the same TTL extension rules as vault data, so that anchors remain accessible as long as the vault data is live.

#### Acceptance Criteria

1. WHEN an anchor is stored or updated, THE Document_Store SHALL extend the persistent storage TTL of the anchor entry using the same `vault_ttl_ledgers` calculation applied to vault data.
2. WHEN `check_in` is called on a vault, THE TtlVaultContract SHALL extend the TTL of all anchor entries associated with that vault by the same amount applied to the vault entry itself.
3. WHEN a vault is cancelled via `cancel_vault`, THE Document_Store SHALL remove all anchor entries associated with that vault.
4. WHEN a vault transitions to `Released` status, THE Document_Store SHALL retain all anchor entries so beneficiaries and verifiers can retrieve them.

### Requirement 8: Document Hash Serialization Round-Trip

**User Story:** As a contract developer, I want document hashes and references to survive serialization and deserialization through Soroban storage without corruption, so that verifiers receive exactly the bytes the owner anchored.

#### Acceptance Criteria

1. THE Document_Store SHALL serialize `Document_Anchor` records to persistent storage using Soroban's native XDR encoding.
2. FOR ALL valid `Document_Anchor` records stored via `anchor_document`, retrieving the same record via `get_anchor` SHALL return a `doc_hash` that is byte-for-byte identical to the originally stored hash (round-trip property).
3. FOR ALL valid `Document_Anchor` records stored via `anchor_document`, retrieving the same record via `get_anchor` SHALL return a `reference` string that is character-for-character identical to the originally stored reference (round-trip property).

### Requirement 9: Legal Document Format Documentation

**User Story:** As a client developer, I want a documented convention for legal document references and types, so that I can build compatible tooling for anchoring and retrieving documents.

#### Acceptance Criteria

1. THE TtlVaultContract SHALL accept any UTF-8 `reference` string up to `MAX_REFERENCE_LEN` bytes, making the contract storage-backend-agnostic.
2. THE TtlVaultContract SHALL accept any UTF-8 `doc_type` string up to `MAX_DOC_TYPE_LEN` bytes, making the contract document-classification-agnostic.
3. WHERE a client stores documents on IPFS, THE client SHALL format the `reference` as `ipfs://{CID}` where `{CID}` is the base32 or base58 IPFS content identifier of the document.
4. WHERE a client anchors a document via a Stellar transaction memo, THE client SHALL format the `reference` as `stellar://{TRANSACTION_HASH}` where `{TRANSACTION_HASH}` is the hex-encoded Stellar transaction hash containing the memo.
5. THE TtlVaultContract SHALL store and return `doc_hash` as a raw `BytesN<32>` without interpreting its contents, making the contract hash-scheme-agnostic (though SHA-256 is the recommended scheme).
