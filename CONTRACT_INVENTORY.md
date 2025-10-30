# StakeWise Smart Contracts Inventory

## Overview
This document provides an inventory of all smart contracts in the StakeWise protocol, including their purpose, upgradeability status, and key security considerations.

## Contracts Inventory

| Contract Name | File Path | Type | Upgradeable | Purpose | Critical Functions |
|--------------|-----------|------|-------------|---------|-------------------|
| **Pool** | `contracts/collectors/Pool.sol` | Core | ✅ Yes | Accumulates ETH deposits, mints tokens, registers validators | `addDeposit()`, `registerValidator()`, `activate()`, `setActivatedValidators()` |
| **PoolEscrow** | `contracts/collectors/PoolEscrow.sol` | Escrow | ❌ No | Receives ETH withdrawals from ETH2 validators | `withdraw()`, `receive()` |
| **Solos** | `contracts/collectors/Solos.sol` | Core | ❌ No | Standalone validators with custom withdrawal credentials | `addDeposit()`, `registerValidator()`, `cancelDeposit()` |
| **StakedEthToken** | `contracts/tokens/StakedEthToken.sol` | Token | ✅ Yes | ERC20 token representing staked ETH deposits | `mint()`, `toggleRewards()`, `_transfer()` |
| **RewardEthToken** | `contracts/tokens/RewardEthToken.sol` | Token | ✅ Yes | ERC20 token representing staking rewards | `updateTotalRewards()`, `claim()`, `setRewardsDisabled()` |
| **StakeWiseToken** | `contracts/tokens/StakeWiseToken.sol` | Token | ✅ Yes | Governance token (SWISE) | `initialize()`, `_transfer()` |
| **Validators** | `contracts/Validators.sol` | Registry | ✅ Yes | Tracks registered validator public keys | `register()`, `isOperator()` |
| **Oracles** | `contracts/Oracles.sol` | Oracle | ✅ Yes | Manages oracle voting for rewards and merkle roots | `voteForRewards()`, `voteForMerkleRoot()` |
| **MerkleDistributor** | `contracts/merkles/MerkleDistributor.sol` | Distribution | ✅ Yes | Distributes rewards via Merkle proofs | `claim()`, `setMerkleRoot()`, `distribute()` |
| **MerkleDrop** | `contracts/merkles/MerkleDrop.sol` | Distribution | ❌ No | One-time token drop via Merkle proofs | `claim()`, `stop()` |
| **VestingEscrow** | `contracts/vestings/VestingEscrow.sol` | Vesting | ✅ Yes | Time-locked token vesting with cliff | `claim()`, `stop()` |
| **VestingEscrowFactory** | `contracts/vestings/VestingEscrowFactory.sol` | Factory | ✅ Yes | Creates vesting escrow instances | `deployEscrow()` |
| **OwnablePausable** | `contracts/presets/OwnablePausable.sol` | Base | ❌ No | Access control + pausable functionality | `pause()`, `addAdmin()`, `removeAdmin()` |
| **OwnablePausableUpgradeable** | `contracts/presets/OwnablePausableUpgradeable.sol` | Base | ✅ Yes | Upgradeable access control + pausable | `pause()`, `addAdmin()`, `removeAdmin()` |

## Key Dependencies

- **OpenZeppelin Contracts**: v3.4.1 (upgradeable versions)
- **Solidity Version**: 0.7.5
- **Validator Deposit**: 32 ETH (constant)

## Access Control Roles

- **DEFAULT_ADMIN_ROLE**: Full administrative control
- **PAUSER_ROLE**: Can pause/unpause contracts
- **ORACLE_ROLE**: Can vote on rewards and merkle roots
- **OPERATOR_ROLE**: Can register validators

## Critical Security Considerations

1. **Upgradeable Contracts**: Many contracts use OpenZeppelin's upgradeable pattern - ensure proper initialization and storage layout compatibility
2. **Oracle Dependencies**: Reward updates depend on oracle consensus (2/3 majority)
3. **Validator Registration**: Only operators can register validators, but must be validated
4. **Reentrancy**: Some contracts use ReentrancyGuard, others rely on checks-effects-interactions pattern
5. **Access Control**: Admin roles can be added/removed - centralized control risk
6. **ETH Handling**: Direct ETH transfers and validator deposits require careful validation

## Financial Flow

1. Users deposit ETH → `Pool.addDeposit()`
2. ETH is minted as `StakedEthToken` (or locked until validator activation)
3. Validators registered via `Pool.registerValidator()` or `Solos.registerValidator()`
4. Rewards updated via `Oracles.voteForRewards()` → `RewardEthToken.updateTotalRewards()`
5. Users claim rewards via `MerkleDistributor.claim()` or automatic accrual
