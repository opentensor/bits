# BIT-0006: Subnet Lending

- **BIT Number:** 0006
- **Title:** Subnet Lending
- **Author(s):** [Name(s) and contact information]
- **Discussions-to:** [Github discussion](https://github.com/opentensor/bits/discussions/16)
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 24/03/2025
- **Updated:** 25/03/2025
- **Requires:** NA
- **Replaces:** NA

## Abstract

The high upfront cost of creating a subnet can deter many users, stifling innovation on Bit Tensor by favoring those with deep pockets or strong networks. To address this, we propose a lending mechanism where a pool of token holders funds the initial subnet creation fee, recouping their investment over time through a share of the subnet owner’s emissions, as agreed upon for a set period.

## Motivation

Bit Tensor’s growth hinges on an ecosystem that fuels innovation and invites diverse talent to experiment with subnets. Yet, the steep upfront costs of launching a subnet can deter skilled individuals and teams—precisely the people who could drive thriving, market-ready subnets aligned with Bit Tensor’s vision. By limiting subnet creation to the well-funded or well-connected, we’re sidelining brilliant minds who could spark breakthroughs and rally communities around them. A native subnet lending solution would break this barrier, empowering more innovators to build, while letting the community back promising projects, accelerating Bit Tensor’s evolution.

## Specification

A Substrate pallet will manage the subnet lending process. A potential subnet creator can submit an initial lending proposal, restricted to one active proposal per AccountId. To deter spam, a small deposit fee—much lower than the subnet creation cost—is required, alongside an expiration date for the proposal. Optionally, a global cap on active proposals could be enforced.

Lenders can express interest in one or more proposals by locking a TAO balance. These funds remain locked until the proposal is fully funded or expires. If a proposal expires without reaching the required amount, the locked TAO is returned to the lenders.

When a proposal’s locked TAO matches the subnet creation cost (dynamically determined by the registration fee at the current block, or capped by a maximum bid), the pallet executes the following:

1. Calculates each lender’s participation share for the proposal based on their contribution.
2. Retrieves the current subnet registration cost from the subtensor pallet.
3. If the maximum bid is below the actual cost, the pallet delays subnet creation until the registration cost decreases to or below the bid amount.
4. If the total locked TAO exceeds the registration cost, the excess is refunded proportionally to each lender.

Upon successful funding, the pallet creates a chain-managed LendingPoolAccount. This account:

1. Generates an on-chain hotkey (TODO: feasible?) linked to the LendingPoolAccount.
2. Registers the subnet, setting LendingPoolAccount as the owner and the pallet-generated hotkey for subnet operations.
3. Delegates operational control of this hotkey to the subnet creator’s AccountId via some delegation mechanism. (TODO: feasible?)
4. Receives TAO emissions into LendingPoolAccount at the end of each tempo.
5. Distributes these emissions to lenders (and potentially the operator) per the proposal’s shares, up to the repayment limit (TODO: could include interest?).

The pallet retains authority over the hotkey hence the subnet. If the operator misbehaves, the pallet could revoke delegation and reassign it to a new operator via governance (TODO: additional complexity?) ensuring lender protection without upfront collateral. 

Post-repayment, subnet ownership transfer to the operator’s hotkey.

## Rationale

TODO

This section describes why particular design decisions were made. It should address alternative
designs and why they were not chosen, and should discuss the potential impact of the proposal
on the existing system.

## Backwards Compatibility

TODO

All BITs that introduce backward incompatibilities must include a section describing these
incompatibilities and their severity. The BIT must explain how the author proposes to deal with
these incompatibilities. BIT submissions without a sufficient backward compatibility treatise
may be rejected outright.

## Reference Implementation (Optional)

TODO

If applicable, include a link to a reference implementation that demonstrates the feasibility
of the proposal. This implementation may be partial or fully complete.

## Security Considerations

WIP

- The actual registration logic of a subnet requires a hotkey, and for this to works, we would need to have a way to have automated control of a hotkey on chain or eventually another way to register a subnet without hotkey but linked to a lending account or a more generic abstraction.

- In the case of deregistration, lenders needs to be aware that their funds will be lost.

All BITs should consider the security implications of their proposals. This section should
address potential attack vectors, vulnerabilities, and how the proposed changes affect the
overall security of the Bittensor protocol.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).