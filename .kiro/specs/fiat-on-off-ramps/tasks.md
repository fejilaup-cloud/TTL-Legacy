# Implementation Plan: Fiat On/Off-Ramps

## Overview

Implement the off-chain Ramp Registry and Anchor HTTP Client in TypeScript. The vault contract (Rust/Soroban) requires no changes — existing `whitelist_token`, `deposit`, and `withdraw` entry points handle all on-chain settlement. All SEP-6/24/31 protocol logic lives in the client layer.

## Tasks

- [ ] 1. Define core types and constants
  - Create `src/ramps/types.ts` with `AnchorProvider`, `RampSession`, `RampDirection`, `RampStatus`, `StellarAsset`, `SepProtocol` interfaces and type aliases
  - Create `src/ramps/constants.ts` with `MAX_MEMO_LEN = 28`, `POLL_MIN_INTERVAL_MS = 5_000`, `POLL_MAX_INTERVAL_MS = 60_000`, `MAX_RETRY_ATTEMPTS = 3`, `SESSION_RETENTION_DAYS = 90`
  - Define all valid `RampStatus` transition pairs as a constant lookup table
  - _Requirements: 2.1, 5.1, 6.5, 9.1_

- [ ] 2. Implement RampSession state machine
  - [ ] 2.1 Implement `isValidTransition(from: RampStatus, to: RampStatus): boolean` in `src/ramps/transitions.ts`
    - Encode the valid transition table from the design: `pending → kyc_required → processing → completed`, `pending → failed`, `completed → disputed → completed | failed`, `failed → disputed → completed | failed`
    - _Requirements: 2.4, 3.3, 3.4, 5.6, 5.7_

  - [ ]* 2.2 Write property test for status transition validity
    - **Property 1: No invalid transition is accepted** — for all `(from, to)` pairs not in the valid set, `isValidTransition` returns `false`
    - **Property 2: All valid transitions are accepted** — every pair in the design's transition table returns `true`
    - **Validates: Requirements 2.4, 3.3, 5.6, 5.7**

- [ ] 3. Implement RampSession serialization
  - [ ] 3.1 Implement `serialize(session: RampSession): string` and `deserialize(json: string): RampSession` in `src/ramps/serialization.ts`
    - All fields must be present in output; `null` fields serialized as JSON `null` (not omitted)
    - `vaultId` serialized as JSON string to avoid 64-bit precision loss
    - Numeric timestamps serialized as JSON numbers (integers)
    - Throw a descriptive error on schema mismatch — do not silently discard
    - _Requirements: 9.1, 9.4_

  - [ ]* 3.2 Write property test for serialization round-trip
    - **Property 3: Round-trip identity** — `deserialize(serialize(session))` produces a session with field values identical to the original for all valid `RampSession` inputs
    - **Property 4: Serialization idempotence** — `serialize(deserialize(serialize(session)))` equals `serialize(session)` for all valid inputs
    - **Validates: Requirements 9.2, 9.3**

- [ ] 4. Implement Anchor HTTP Client
  - [ ] 4.1 Implement `AnchorHttpClient` class in `src/ramps/anchorHttpClient.ts`
    - `fetchToml(domain: string): Promise<StellarToml>` — fetch and parse `/.well-known/stellar.toml`
    - `sep6Deposit`, `sep6Withdraw`, `sep6TransactionStatus` — call SEP-6 REST endpoints
    - `sep24InteractiveUrl` — return interactive URL for popup/iframe launch
    - `sep24TransactionStatus` — poll SEP-24 transaction status
    - `sep31Send` — call SEP-31 `/send` endpoint
    - _Requirements: 1.4, 1.5, 2.2, 2.3, 5.2, 5.3_

  - [ ] 4.2 Add retry logic with exponential backoff to `AnchorHttpClient`
    - Retry up to `MAX_RETRY_ATTEMPTS` (3) on network timeouts or HTTP 5xx responses
    - Backoff delays: 1 s, 2 s, 4 s
    - After exhausting retries, throw an error that the caller can map to `Failed` status
    - _Requirements: 7.2_

  - [ ]* 4.3 Write unit tests for AnchorHttpClient retry logic
    - Test that exactly 3 attempts are made before failure on persistent 5xx
    - Test that a success on the 2nd attempt does not trigger a 3rd
    - Test that non-retryable errors (4xx) are not retried
    - _Requirements: 7.2_

- [ ] 5. Implement RampRegistry — provider management
  - [ ] 5.1 Implement `addProvider(sepTomlUrl: string): Promise<AnchorProvider>` in `src/ramps/rampRegistry.ts`
    - Fetch and validate SEP-1 TOML via `AnchorHttpClient.fetchToml`
    - Reject if provider does not support at least one of SEP-6, SEP-24, or SEP-31
    - Store provider config keyed by derived `id` (domain)
    - _Requirements: 1.3, 1.4, 1.5_

  - [ ] 5.2 Implement `getProvider`, `listProviders` on `RampRegistry`
    - _Requirements: 1.3_

  - [ ]* 5.3 Write unit tests for provider validation
    - Test rejection when TOML fetch fails
    - Test rejection when TOML lists no supported SEPs
    - Test successful storage when at least one SEP is supported
    - _Requirements: 1.4, 1.5_

- [ ] 6. Implement RampRegistry — session lifecycle
  - [ ] 6.1 Implement `initiateDeposit(params: DepositParams): Promise<RampSession>` on `RampRegistry`
    - Create `RampSession` with `direction = 'deposit'`, `status = 'pending'`, UUID v4 `sessionId`
    - Set `memo` to `sessionId` truncated to `MAX_MEMO_LEN` bytes
    - For SEP-24: call `sep24InteractiveUrl` and return URL for client to open
    - For SEP-6: call `sep6Deposit` with Owner address, asset code, and memo
    - On anchor error response: set `status = 'failed'`, store error message
    - _Requirements: 2.1, 2.2, 2.3, 2.6_

  - [ ] 6.2 Implement `initiateWithdrawal(params: WithdrawalParams): Promise<RampSession>` on `RampRegistry`
    - Create `RampSession` with `direction = 'withdrawal'`, `status = 'pending'`, UUID v4 `sessionId`
    - For SEP-24: call `sep24InteractiveUrl` for withdrawal
    - For SEP-6: call `sep6Withdraw` to obtain anchor receiving address and memo
    - _Requirements: 5.1, 5.2, 5.3_

  - [ ] 6.3 Implement `getSession`, `listSessions` on `RampRegistry`
    - _Requirements: 6.5, 8.3_

  - [ ]* 6.4 Write unit tests for session initiation
    - Test that `sessionId` is unique across multiple calls
    - Test that memo is truncated to 28 bytes
    - Test that anchor error response sets `status = 'failed'`
    - _Requirements: 2.1, 2.3, 2.6_

- [ ] 7. Implement RampRegistry — status polling
  - [ ] 7.1 Implement `pollSession(sessionId: string): Promise<RampSession>` on `RampRegistry`
    - Call anchor's SEP-6 or SEP-24 transaction status endpoint
    - Map anchor status strings to `RampStatus` values
    - On `completed`: call `markCompleted`; on `error`/`expired`: call `markFailed`
    - On `kyc_customer_info_needed`: set `status = 'kyc_required'`, store KYC URL
    - Validate transition via `isValidTransition` before applying
    - _Requirements: 2.4, 2.5, 3.3, 5.6, 6.2, 6.3, 6.4_

  - [ ] 7.2 Implement polling loop with interval bounds in `src/ramps/poller.ts`
    - Poll at intervals between `POLL_MIN_INTERVAL_MS` (5 s) and `POLL_MAX_INTERVAL_MS` (60 s)
    - Stop when session reaches terminal status (`completed` or `failed`)
    - _Requirements: 2.5, 6.2_

  - [ ]* 7.3 Write unit tests for polling interval enforcement
    - Test that polling never fires faster than 5 s
    - Test that polling stops on terminal status
    - _Requirements: 2.5, 6.2_

- [ ] 8. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement RampRegistry — completion, failure, and dispute management
  - [ ] 9.1 Implement `markCompleted(sessionId, txHash, externalRef)` on `RampRegistry`
    - Set `status = 'completed'`, record `stellarTxHash`, `externalRef`, `completedAt` timestamp
    - _Requirements: 4.3, 5.7, 6.1_

  - [ ] 9.2 Implement `markFailed(sessionId, reason)` on `RampRegistry`
    - Set `status = 'failed'`, store failure reason in `errorMessage`
    - Emit a client-side `ramp:failed` event with `sessionId`, `direction`, and reason
    - _Requirements: 6.4, 7.1, 7.5_

  - [ ] 9.3 Implement `raiseDispute(sessionId, description)` on `RampRegistry`
    - Validate transition to `'disputed'` via `isValidTransition`
    - Set `status = 'disputed'`, record `disputeDescription` and `disputeRaisedAt`
    - Return anchor support contact from provider config
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ] 9.4 Implement `resolveDispute(sessionId, resolution, note)` on `RampRegistry`
    - Validate `resolution` is `'completed'` or `'failed'`
    - Validate transition via `isValidTransition`
    - Set status and record `resolutionNote`
    - _Requirements: 8.4_

  - [ ]* 9.5 Write unit tests for dispute lifecycle
    - Test `raiseDispute` on a `completed` session transitions to `disputed`
    - Test `resolveDispute` transitions `disputed → completed` and `disputed → failed`
    - Test that `raiseDispute` on a `pending` session is rejected
    - _Requirements: 8.1, 8.4_

- [ ] 10. Implement session persistence
  - [ ] 10.1 Implement `serialize` / `deserialize` integration in `RampRegistry` storage layer
    - Persist sessions to client-side storage (e.g. `localStorage` or a JSON file adapter)
    - Use the `serialize`/`deserialize` functions from task 3.1
    - Surface descriptive error on schema mismatch; do not silently discard records
    - _Requirements: 9.1, 9.4_

  - [ ] 10.2 Implement session retention enforcement
    - On load, filter out sessions older than `SESSION_RETENTION_DAYS` (90 days) only if in a non-terminal status; retain terminal sessions for the full 90 days
    - _Requirements: 6.5_

  - [ ]* 10.3 Write unit tests for persistence
    - Test that a session survives a serialize → store → load → deserialize round-trip
    - Test that a schema-mismatched record throws a descriptive error
    - _Requirements: 9.1, 9.4_

- [ ] 11. Wire components together
  - [ ] 11.1 Export a top-level `createRampRegistry(storage, httpClient)` factory in `src/ramps/index.ts`
    - Compose `RampRegistry`, `AnchorHttpClient`, serialization, and poller
    - Expose the full `RampRegistry` interface as the public API
    - _Requirements: 1.3, 2.1, 5.1_

  - [ ]* 11.2 Write integration tests for deposit and withdrawal flows
    - Test full deposit flow: `addProvider` → `initiateDeposit` → poll to `completed` → `markCompleted`
    - Test full withdrawal flow: `initiateWithdrawal` → poll → `markCompleted`
    - Test KYC branch: poll returns `kyc_required` → status updates → resumes to `processing`
    - _Requirements: 2.1–2.6, 3.1–3.6, 4.1–4.5, 5.1–5.7_

- [ ] 12. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- The vault contract (Rust/Soroban) is not modified — `whitelist_token`, `deposit`, and `withdraw` are used as-is
- All SEP protocol logic is TypeScript, client-side only
- Property tests validate universal correctness properties; unit tests cover specific examples and edge cases
- Checkpoints ensure incremental validation before moving to the next phase
