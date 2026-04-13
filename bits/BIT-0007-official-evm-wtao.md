# BIT-0007: Official wTAO Contract on Bittensor EVM

- **BIT Number:** 0007
- **Title:** Official wTAO Contract on Bittensor EVM
- **Author(s):** Captain Quint - discord @quintessenial
- **Discussions-to:** N/A
- **Status:** Draft
- **Type:** Subtensor | Meta | Informational
- **Created:** 2025/05/07
- **Updated:** 2025/05/07
- **Requires:** N/A
- **Replaces:** N/A

## Abstract

Bittensor needs an official and endorsed ERC-20 compatible wTAO contract for use across all applications built within the Bittensor EVM.
By design, an EVM does not allow an EOA (Externally Owned Account) to grant another address direct access to the native token of a network.
This is often handled on other networks by creating an ERC-20 compatible wrapper contract for the native token.

**Note:** This proposal has *nothing* to do with the wTAO contract deployed on Ethereum nor bridging between any network. This is scoped soley for use within the Bittensor EVM, where 1 wTAO == 1 TAO.

## Motivation

wTAO is a fundamental building block required by developers building on the Bittensor EVM. [WETH](https://etherscan.io/address/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2), the universally accepted wrapped ethereum token was deployed in Dec 2017 and today contains over 2.5M ETH (>$4B USD) and has exceeded 20M transactions.
It is listed across most, if not all, wallets and accepted in nearly all scenarios where native ETH is.
It's existance has made the development of dapps on Ethereum much easier and streamlined.
Ethereum is not the only chain to have adopted a wrapped native token standard on their network.
Nearly all EVM based L1s have an active and universal contract deployed. wPOL, wBNB, wAVAX, even wSOL etc.. use this today.
Not having a wTAO contract ready to use for builders is a blocker and is holding back development on the Bittensor EVM.

According to [taostats](https://evm.taostats.io/tokens?q=wtao) there are currently 6 different wTAO deployed contracts (based soley on the name containing characters "wtao").
This is a sign of already existing fragmentation.
Today, each developer building in the EVM will need to deploy their own wTAO contract which will be only compatible with their application.
Multiple incompatible implementations will lead to confusion and chain bloat needlessly switching between wTAO versions, as well as potentially malicious implementations being deployed.

OTF should deploy and endorse one wTAO contract to become the accepted standard. The docs should reference the official contract address, including instructions on adding the token to common wallets.

## Specification & Reference Implementation

Based on [WETH9](https://etherscan.io/address/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2#code#L1) implementation, a reference implementation of WTAO has been completed as part of [taonado-cash](https://github.com/taonado/taonado-cash/blob/da6f4c4a551a072e4ffcc3223e85a8f4587e7147/contracts/WTAO.sol)
This contains minimal modifications from the original WETH9 implementation. The original contract was compiled with Solidity `v0.4.19+commit.c4cbbb05` which compared to the modern `^0.8.24` compiler, there are better ways of expressing a few of the original statements. There have also been a few naming changes, obviously WETH -> WTAO.

## Rationale

Basing the design off a proven standard is the obvious route forward.
Many applications such as DeFi primitives, liquidity pools etc.. are built under the assumption of a universal token standard which this proposal brings forward.
This BIT does not propose to re-invent the wheel, but merely install an existing one to the Bittensor car.

## Backwards Compatibility

Since this BIT does not replace or improve any existing offical wTAO contract, it is net-new.
Existing applications which use one of the already deployed wTAO contracts may need to be updated, here is a [reference implementation](https://github.com/taonado/taonado-cash/blob/da6f4c4a551a072e4ffcc3223e85a8f4587e7147/contracts/weights.sol#L27).
Any TAO deposited to these existing wTAO implementations can be easily unwrapped and re-wrapped with the official contract.

## Security Considerations

Not having an official address creates a burden on every single project which needs a wTAO token. It creates an additional step and diligence from the developer to deploy, update, and maintain. Multiple wTAO addresses are confusing and could lead to issues such as users mistakenly sending wTAO1 instead of wTAO2, resulting in an irrecoverable loss of funds.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
