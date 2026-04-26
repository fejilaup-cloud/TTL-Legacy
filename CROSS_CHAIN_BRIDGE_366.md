# Cross-Chain Bridge Support - Issue #366

## Overview
This document outlines the cross-chain bridge architecture for TTL-Legacy, enabling vaults to manage assets on multiple blockchains.

## Architecture

### 1. Bridge Integration Design

The cross-chain bridge support is implemented through:

- **Bridge Registry**: Maintains mapping of supported chains and their bridge contracts
- **Asset Mapping**: Maps assets across chains (e.g., USDC on Ethereum → USDC on Stellar)
- **Cross-Chain Vault**: Extended vault structure to support multi-chain assets
- **Bridge Operations**: Deposit/withdraw operations that interact with bridge contracts

### 2. Data Structures

#### Bridge Configuration
```rust
pub struct BridgeConfig {
    pub chain_id: u32,
    pub bridge_address: Address,
    pub supported_assets: Vec<Address>,
    pub is_active: bool,
}

pub struct AssetMapping {
    pub source_chain: u32,
    pub source_asset: Address,
    pub target_chain: u32,
    pub target_asset: Address,
    pub exchange_rate: u64, // Fixed-point rate
}
```

#### Cross-Chain Vault Extension
```rust
pub struct CrossChainVault {
    pub base_vault: Vault,
    pub chain_id: u32,
    pub bridge_config: BridgeConfig,
    pub asset_mappings: Vec<AssetMapping>,
}
```

### 3. Implementation Strategy

#### Phase 1: Bridge Registry (Current)
- Admin can register supported chains
- Admin can add bridge contracts
- Admin can configure asset mappings
- View functions to query bridge configuration

#### Phase 2: Cross-Chain Deposits
- Deposit assets from other chains
- Bridge contract handles asset transfer
- Vault balance updated on Stellar
- Event emitted for bridge indexing

#### Phase 3: Cross-Chain Releases
- Release funds to beneficiary on target chain
- Bridge contract handles cross-chain transfer
- Atomic settlement with bridge
- Event emitted for bridge confirmation

#### Phase 4: Multi-Chain Vaults
- Single vault managing assets across multiple chains
- Consolidated balance view
- Cross-chain release logic

### 4. API Functions

#### Admin Functions
```rust
pub fn register_bridge(env: Env, chain_id: u32, bridge_address: Address) -> Result<(), ContractError>
pub fn add_asset_mapping(env: Env, mapping: AssetMapping) -> Result<(), ContractError>
pub fn deactivate_bridge(env: Env, chain_id: u32) -> Result<(), ContractError>
pub fn get_bridge_config(env: Env, chain_id: u32) -> Option<BridgeConfig>
```

#### Vault Operations
```rust
pub fn create_cross_chain_vault(
    env: Env,
    owner: Address,
    beneficiary: Address,
    check_in_interval: u64,
    chain_id: u32,
) -> Result<u64, ContractError>

pub fn deposit_from_bridge(
    env: Env,
    vault_id: u64,
    source_chain: u32,
    amount: i128,
    bridge_tx_hash: String,
) -> Result<(), ContractError>

pub fn release_to_bridge(
    env: Env,
    vault_id: u64,
    target_chain: u32,
    beneficiary_address: String,
) -> Result<(), ContractError>
```

### 5. Security Considerations

#### Bridge Trust Model
- Bridges are trusted intermediaries
- Admin controls bridge registration
- Asset mappings validated before use
- Bridge transactions require proof (tx hash)

#### Cross-Chain Atomicity
- Deposits: Bridge → Stellar (one-way)
- Releases: Stellar → Bridge (one-way)
- No atomic cross-chain transactions
- Retry logic for failed transfers

#### Asset Validation
- Only whitelisted assets can be bridged
- Exchange rates validated before conversion
- Balance checks prevent over-withdrawal
- Slippage protection for conversions

### 6. Implementation Status

#### Completed
- ✅ Bridge registry structure
- ✅ Asset mapping configuration
- ✅ Admin functions for bridge management
- ✅ View functions for bridge queries

#### Planned (Future Phases)
- ⏳ Cross-chain deposit implementation
- ⏳ Cross-chain release implementation
- ⏳ Multi-chain vault support
- ⏳ Bridge event indexing
- ⏳ Atomic settlement logic

### 7. Testing Strategy

#### Unit Tests
- Bridge registration validation
- Asset mapping validation
- Exchange rate calculations
- Authorization checks

#### Integration Tests
- Bridge contract interaction
- Cross-chain deposit flow
- Cross-chain release flow
- Error handling and recovery

#### Security Tests
- Bridge address validation
- Asset whitelist enforcement
- Exchange rate manipulation prevention
- Unauthorized bridge access prevention

### 8. Deployment Considerations

#### Testnet Deployment
1. Deploy bridge contracts on testnet
2. Register bridges in TTL-Legacy
3. Configure asset mappings
4. Test cross-chain operations

#### Mainnet Deployment
1. Audit bridge contracts
2. Establish bridge partnerships
3. Configure production asset mappings
4. Enable cross-chain operations

### 9. Future Enhancements

#### Multi-Bridge Support
- Support multiple bridges per chain
- Bridge selection logic
- Fallback bridge handling

#### Liquidity Pools
- Integrate with DEX liquidity
- Automated market maker support
- Slippage optimization

#### Governance
- DAO-controlled bridge registry
- Community-voted asset mappings
- Bridge fee distribution

## Conclusion

The cross-chain bridge architecture provides a foundation for TTL-Legacy to manage assets across multiple blockchains. The phased implementation approach allows for incremental feature rollout while maintaining security and stability.

---
**Status**: Architecture Defined
**Priority**: Low
**Estimated Implementation**: 6+ hours
