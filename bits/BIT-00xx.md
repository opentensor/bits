# BIT-00xx: Track validator voting power

- **BIT Number:** 00??
- **Title:** Track voting power of validators in each subnet that needs it
- **Author(s):** Rhef
- **Discussions-to:** https://forum.bittensor.church/t/add-tracking-stakeweight-over-time/34
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 2025-10-22
- **Updated:** 2025-10-22
- **Requires:** -
- **Replaces:** -

## Abstract

Some smart contracts, such as collateral and treasury, need a way to count votes of validators in order to make a decision for the subnet.
The only metrics currently available are:
- current stake
- stakeweight (stake snapshot saved at the last epoch)

and both can be manipulated with loans and flashloans, where an attacker could hijack the treasury or burn collaterals of his competitors.

The proposed solution is for the chain to provide a realiable metric that the votes could be weighted by. This could then be used by smart contracts as well as on-chain logic.


## Motivation

Up until recently smart contract calls could be sandwitched between a stake and unstake operation, allowing flashloans of extreme magnitude to safely enable the attacker to cast a high weight vote.
Currently in Bittensor simple flashloans wouldn't work, though wrapped equivalents might still be possible. Normal (collateralized) loans are still a threat - even if the attack will last for more than one block, it could be possible (especially after we move to PoS) to execute it safely without risk of MEV.

Unless the contract operator would call it very frequently to sample the stake of all validators over a long period of time, smart contracts do not have reliable means of assuming vote weights at key moments such as deciding whether to burn a collateral or transfer funds from treasury.
Frequently calling the smart contract to sample stake would be prohibitively expensive.

## Proposal

A set of new hyperparameters is proposed to be added to indicate which subnets should have their validator stakeweight long term average (? or median or ema or something) tracked (long term = say two weeks). It would only be tracked for up to N top validators per subnet with at least K (alpha? tao?) worth of stake, to reduce the chain load.

In practice this would need to be enabled on very few subnets - only on those which use smart contracts to make decisions in the name of the subnet, such as Collateral or Capacitor.
The new “smart contract voting power” metric would allow the contracts to weighting the votes.


## Specification

There are several ideas on how this could be implemented, and a discussion needs to take place so that we can figure out which one of those solutions is viable.

Assumptions:
- max_netuid = 1024 (currently 128)
- max_validators = 64 (should be probably decreased to 32 because no subnet in history ever had more than 24 and slots 25-64 are only usable during a sybil attack)
- subsubnets version = 1 (v2 might need to store the metric separately for each mechanism)
- sample count = two weeks = 14 * 7200 = 100800


### storage and compute cost

| method name   | storage cost (worst case) | compute cost |
| ------------- | ------------- | ------------- |
| stake EMA     | 263 KB | O(max_netuid * max_validators) every block |
| min/average   |  20 MB | O(641) per epoch after basic optimizations |
| median        | 6.6 GB | O(64 000 000) per epoch (naiive)           |
| median (sampling) | 6-60 MB | O(64 000)-O(640 000) per epoch, it seems that it can be optimized much further at a cost of slight inaccuracy |

### algorithm candidates

#### stake EMA

Very storage efficient, but can be skewed by a loan or a flashloan so a subnet which has a cabal anywhere near 50% of stake would be at very high risk.

#### average

Could use compression to reduce storage further. Averages are vulnerable to a few blocks of a stake spike.

#### min

Very hard to cheat. Can be manipulated by whales moving stake between root validators to weaken them.

#### median

Potential storage efficiency issues, but very fair results.

### random sampling

If we can select a handful of blocks now and then based on drand, block crafter random secret, last block hash etc. then we can potentially reduce the storage and compute cost by 2-4 orders of magnitude.


## Rationale

TODO: fill after the choice is made


## Backwards Compatibility

N/A


## Reference Implementation (Optional)

TODO: fill after the choice is made


## Security Considerations

TODO: fill after the choice is made


## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
