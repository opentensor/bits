# BIT-0019: Substrate Smart Contracts

- **BIT Number:** 0019
- **Title:** Substrate Smart Contracts
- **Author(s):** Ben Mason, Francisco Silva
- **Status:** Draft
- **Type:** Core
- **Created:** 24/09/2025

## Abstract

This BIT proposes the integration of Substrate's `pallet_contracts` into the Bittensor blockchain, enabling smart contracts written in Rust using the ink! language. This native Substrate smart contract functionality will provide seamless interaction between smart contracts and Bittensor's pallets, offering a more developer-friendly and user-friendly alternative to the existing EVM-based smart contracts.

## PR

https://github.com/opentensor/subtensor/pull/2059

## Motivation

The current EVM integration in Bittensor creates significant friction for both developers and users due to the fundamental differences between Ethereum's account-based model and Substrate's architecture. This mismatch complicates smart contract development, increases gas costs, and creates poor user experiences when interacting with Substrate-based wallets.

By implementing `pallet_contracts`, we enable:

1. **Native Substrate Compatibility**: Smart contracts can directly interact with Bittensor pallets without complex bridging or compatibility layers
2. **Rust/Ink! Development**: Developers can write smart contracts in Rust using the familiar ink! framework, leveraging existing Substrate tooling
3. **Improved Performance**: Native contracts execute more efficiently than EVM contracts on Substrate infrastructure
4. **Better Wallet Integration**: Seamless interaction with Substrate-based wallets like Polkadot.js, SubWallet, and others

## Getting Started

For general smart contract development on Subtensor, please refer to the official ink! documentation:

- [ink! Documentation](https://use.ink/docs/v5/)
- [ink! Getting Started Guide](https://use.ink/docs/v5/getting-started/setup)
- [ink! Examples](https://github.com/use-ink/ink-examples/tree/v5.x.x)

## Specification

1. **Pallet Integration**: Add `pallet_contracts` to the Bittensor runtime, configured to work with the existing pallet structure
2. **Whitelist Calls**: List the calls we want to make available to the smart contracts [reference](https://github.com/opentensor/subtensor/pull/2059/files#diff-d6f76ae401d0ed69fd1b08b7726270e3ad7983ab53491c90b55a36787c835418R41)

## Backwards Compatibility

This implementation is fully backwards compatible with existing Bittensor functionality:

1. **EVM Support**: Existing EVM smart contracts continue to function without modification
2. **Migration Path**: Provides a migration path for existing contracts that wish to move to native contracts
3. **Dual Environment**: Both EVM and native contracts can coexist and potentially interact
4. **No Breaking Changes**: No existing pallets or interfaces are modified

## Example Use Cases

### Collateral Contracts

Smart contracts that allow subnet owners to create staking mechanisms where miners can stake alpha tokens as collateral for good behavior. These contracts can:

- Hold alpha tokens as collateral
- Implement slashing mechanisms for malicious behavior
- Distribute rewards for positive contributions

### OTC Trading Desk

Decentralized over-the-counter trading contracts that enable:

- Direct peer-to-peer alpha token trading
- Miners to sell without impacting the pool
- Reduced slippage for large trades

### Native Bridge Contracts

Substrate-native bridge contracts for subnets with their own chains to bridge alpha tokens between subnet chains and Bittensor mainnet.

## Security Considerations

### Chain extensions

A chain extension is a way to extend `pallet_contracts` API by exposing parts of the runtime logic to smart contracts. Added functions should handle security (needs to be audited). In a case of a wrapper around an existing pallet (so that contract can call functions of this pallet) every pallet dispatchable should be implemented as a chain extension function, unit tested and be benchmarked (to determine the correct amount of weight).

### Runtime Calls

`call_runtime` is a function already present in `pallet_contracts` API that dispatches a `Call` passed as an argument. This way contracts can call pallets without having to go through chain extensions. There is no security issue and does not need to be audited (as it calls pallets directly), weight is also handled (as it uses weight from the pallet). No need to add tests either (pallets already have tests).
To activate/deactivate dispatchables accessible from a contract it should be added to `CallFilter`.

Smart contracts can to make runtime calls in which the caller is the contract itself. Similar to how EVM DEXs use ERC20 allowances to move tokens, users would have to grant delegate the contract as a staking proxy to allow it to move stake on their behalf. As with EVM, users have to analyse the contract they are interacting with to decide if it is exploitative or not.

### Design Decisions

1. **Conservative Limits**: Start with conservative contract execution limits to ensure network stability
2. **Selective API Exposure**: Carefully control which runtime functions are accessible to contracts for security

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
