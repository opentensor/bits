# BIT-0009: Track Validator Voting Power

- **BIT Number:** 0009
- **Title:** Track Voting Power of Validators in each Subnet That Needs It
- **Author(s):** Rhef
- **Discussions-to:** https://forum.bittensor.church/t/add-tracking-stakeweight-over-time/34
- **Status:** Final
- **Type:** Subtensor
- **Created:** 2025-10-22
- **Updated:** 2026-04-13
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

Up until recently smart contract calls could be sandwitched between a stake and unstake operation, allowing flashloans of extreme magnitude to safely enable the attacker to cast a high weight vote. Currently in Bittensor simple flashloans wouldn't work, though wrapped equivalents might still be possible. Normal (collateralized) loans are still a threat - even if the attack will last for more than one block, it could be possible (especially after we move to PoS) to execute it safely without risk of MEV.

Unless the contract operator would call it very frequently to sample the stake of all validators over a long period of time, smart contracts do not have reliable means of assuming vote weights at key moments such as deciding whether to burn a collateral or transfer funds from treasury. Frequently calling the smart contract to sample stake would be prohibitively expensive.

## Proposal

A per-subnet `VotingPowerTrackingEnabled` flag controls which subnets have their validator stake EMA tracked. When enabled, an exponential moving average of each validator's stake is updated every epoch, providing a manipulation-resistant voting power metric. Validators with permits are tracked; miners are excluded. Entries are removed when hotkeys deregister or decay below the stake threshold.

In practice this would need to be enabled on very few subnets - only on those which use smart contracts to make decisions in the name of the subnet, such as Collateral or Capacitor. The new "Smart Contract Voting Power" metric would allow the contracts to weighting the votes.


## Specification

The stake EMA algorithm was chosen for its storage efficiency and per-epoch compute cost.

### Storage

- `VotingPower<T>`: `DoubleMap<NetUid, AccountId> -> u64` — EMA of stake per validator per subnet
- `VotingPowerTrackingEnabled<T>`: `Map<NetUid> -> bool` — per-subnet opt-in flag (default: false)
- `VotingPowerDisableAtBlock<T>`: `Map<NetUid> -> u64` — block at which tracking will be disabled after a 14-day grace period
- `VotingPowerEmaAlpha<T>`: `Map<NetUid> -> u64` — EMA alpha with 18-decimal precision (`10^18 = 1.0`), settable by root/sudo

### Defaults

- EMA alpha: `0.00357 * 10^18` — gives a 2-week e-folding time constant at 361 blocks/tempo
  - After ~2 weeks: EMA reaches 63.2% of a step change
  - After ~4 weeks: 86.5%
  - After ~6 weeks: 95%
- **Grace period**: 100,800 blocks (14 days) — when tracking is disabled, it continues for this period before clearing data

### EMA Calculation

Updated every epoch for each validator with a permit:

```
new_ema = alpha * current_stake + (1 - alpha) * previous_ema
```

Intermediate calculations use u128 to avoid overflow.

### Extrinsics
- `enable_voting_power_tracking(netuid)` — subnet owner or root enables tracking
- `disable_voting_power_tracking(netuid)` — schedules disable with 14-day grace period
- `set_voting_power_ema_alpha(netuid, alpha)` — root/sudo sets the EMA alpha

### Smart Contract Access

Voting power is accessible to smart contracts via an EVM precompile, allowing contracts to query VotingPower for any (netuid, hotkey) pair.

### Additional Behaviors
- **Deregistration cleanup**: voting power is removed for hotkeys no longer registered on the subnet
- **Hotkey swap**: voting power transfers from old to new hotkey during swaps
- **Threshold removal**: if a validator's EMA decays below the stake threshold (having previously been above it), the entry is removed

## Rationale

The stake EMA was chosen over min, average, median, and random sampling approaches because:

- **Storage efficient**: 263 KB worst case vs 20 MB–6.6 GB for alternatives
- **Compute efficient**: O(validators) per epoch, no per-block cost
- **Configurable responsiveness**: the alpha parameter lets root tune the tradeoff between manipulation resistance (low alpha, slow response) and responsiveness (high alpha, fast response)
- **14-day grace period on disable**: prevents subnet owners from suddenly removing voting power data that smart contracts depend on


## Backwards Compatibility

N/A — opt-in per subnet, no changes to existing behavior.


## Reference Implementation (Optional)

- Storage: `pallets/subtensor/src/lib.rs` (`VotingPower`, `VotingPowerTrackingEnabled`, `VotingPowerDisableAtBlock`, `VotingPowerEmaAlpha`)
- Logic: `pallets/subtensor/src/utils/voting_power.rs`
- Precompile: `precompiles/src/voting_power.rs`
- Tests: `pallets/subtensor/src/tests/voting_power.rs`, `contract-tests/test/votingPower.precompile.test.ts`


## Security Considerations

- EMA can be skewed by sustained stake manipulation (loans held across many epochs), but the low default alpha (2-week time constant) makes short-term attacks impractical.
- The 14-day grace period on disable prevents a subnet owner from abruptly removing voting power data that smart contracts rely on for governance decisions.
- Only root/sudo can set the EMA alpha, preventing subnet owners from weakening the metric by setting alpha too high.


## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
