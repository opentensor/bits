# BIT-0011: Recycle UID

- **BIT Number:** 0011
- **Title:** Recycle Subnet Owner UID Incentives Instead of Burning
- **Author(s):** Maciej Kula
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1392648789956890636
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-07-18
- **Updated:** 2025-07-18
- **Requires:** None

## Abstract

This BIT proposes changing the subnet owner UID implementation from a "burn UID" to a "recycle UID". Currently, mining incentives allocated to the subnet owner hotkey are burned by being skipped during distribution while still being counted in emission total. This proposal would instead recycle these incentives by reducing the `SubnetAlphaOut` variable, making the ALPHA available for future distribution similar to how TAO was recycled pre-DTAO.

## Motivation

The current implementation burns ALPHA tokens when the subnet's consensus (through validator weights) decides they should not be distributed immediately. This is inconsistent with how the protocol handled similar situations pre-DTAO, where validators could assign weights to root and TAO would be recycled for later distribution.

When validators collectively decide certain incentives should not be distributed, this represents a consensus decision about "timing" rather than a desire to permanently remove tokens from circulation. The tokens should remain available for future distribution when conditions change.

## Specification

**Current Implementation:**
```rust
// Distribute mining incentives.
for (hotkey, incentive) in incentives {
    if let Ok(owner_hotkey) = SubnetOwnerHotkey::::try_get(netuid) {
        if hotkey == owner_hotkey {
            continue; // Skip/burn miner-emission for SN owner hotkey.
        }
    }
    // ... other hotkeys
}
```

**Proposed Implementation:**
```rust
// Distribute mining incentives.
for (hotkey, incentive) in incentives {
    if let Ok(owner_hotkey) = SubnetOwnerHotkey::::try_get(netuid) {
        if hotkey == owner_hotkey {
            // Recycle the incentive instead of burning
            SubnetAlphaOut::::mutate(netuid, |total| {
                *total = total.saturating_sub(incentive);
            });
            continue;
        }
    }
    // ... other hotkeys
}
```

## Rationale

The change aligns with the existing recycling pattern used in `do_recycle_alpha` and maintains consistency with pre-DTAO TAO recycling on root. By reducing `SubnetAlphaOut`, the recycled ALPHA becomes available for future distribution rather than being permanently destroyed.

## Backwards Compatibility

This BIT introduces no backward incompatibilities. The change modifies only distribution logic.

## Reference Implementation

TBD

## Security Considerations

TBD

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
