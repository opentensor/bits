# BIT-0005: Recycle UID

- **BIT Number:** 0005
- **Title:** Recycle or Burn Subnet Owner UID Incentives
- **Author(s):** Maciej Kula, Vune
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1392648789956890636
- **Status:** Final
- **Type:** Core
- **Created:** 2025-07-18
- **Updated:** 2026-04-13
- **Requires:** None

## Abstract

This BIT proposes changing the subnet owner UID implementation from a fixed "burn UID" to a configurable "recycle or burn" per subnet. Currently, mining incentives allocated to the subnet owner hotkey (and associated hotkeys) are skipped during distribution. This proposal adds a per-subnet hyperparameter allowing subnet owners to choose whether these skipped incentives are recycled (by reducing `SubnetAlphaOut`) or burned (no-op, effectively removing them from circulation).

## Motivation

The current implementation burns ALPHA tokens when the subnet's consensus (through validator weights) decides they should not be distributed immediately. This is inconsistent with how the protocol handled similar situations pre-DTAO, where validators could assign weights to root and TAO would be recycled for later distribution.

When validators collectively decide certain incentives should not be distributed, this represents a consensus decision about "timing" rather than a desire to permanently remove tokens from circulation. The tokens should remain available for future distribution when conditions change.

## Specification

A per-subnet storage map `RecycleOrBurn` is added with a `RecycleOrBurnEnum` value (Burn or Recycle), defaulting to Burn.

Subnet owners can set this via `sudo_set_recycle_or_burn` in the admin-utils pallet.

During incentive distribution, when a hotkey is identified as the subnet owner hotkey or an associated hotkey, the incentive is handled according to the subnet's setting:

```rust
// Check if we should recycle or burn the incentive
match RecycleOrBurn::<T>::try_get(netuid) {
    Ok(RecycleOrBurnEnum::Recycle) => {
        Self::recycle_subnet_alpha(netuid, incentive);
    }
    Ok(RecycleOrBurnEnum::Burn) | Err(_) => {
        Self::burn_subnet_alpha(netuid, incentive);
    }
}
```

Where:
- `recycle_subnet_alpha` reduces `SubnetAlphaOut`, making the ALPHA available for future distribution
- `burn_subnet_alpha` is currently a no-op (the alpha is simply not distributed)

## Rationale

The change aligns with the existing recycling pattern and maintains consistency with pre-DTAO TAO recycling on root. By reducing `SubnetAlphaOut`, the recycled ALPHA becomes available for future distribution rather than being permanently removed. Making it a per-subnet choice gives subnet owners flexibility to select the behavior that best fits their tokenomics.

## Backwards Compatibility

This BIT introduces no backward incompatibilities. The default behavior remains Burn, preserving existing behavior for all subnets unless explicitly changed by the subnet owner.

## Reference Implementation

- **Storage**: `RecycleOrBurn<T>` in `pallets/subtensor/src/lib.rs`
- **Distribution logic**: `distribute_dividends_and_incentives` in `pallets/subtensor/src/coinbase/run_coinbase.rs`
- **Recycle helper**: `recycle_subnet_alpha` in `pallets/subtensor/src/staking/helpers.rs`
- **Setter**: `sudo_set_recycle_or_burn` in `pallets/admin-utils/src/lib.rs`
- **User extrinsics**: `recycle_alpha` and `burn_alpha` in `pallets/subtensor/src/staking/recycle_alpha.rs`

## Security Considerations

None

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
