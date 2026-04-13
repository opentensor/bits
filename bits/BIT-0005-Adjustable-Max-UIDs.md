# BIT-0005: Adjustable Max UIDs

- **BIT Number:** 0005
- **Title:** Adjustable Max UIDs
- **Author(s):** Namoray
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1389781506444230656
- **Status:** Final
- **Type:** Core
- **Created:** 2025-07-02
- **Updated:** 2026-04-13
- **Requires:** -
- **Replaces:** -

## Abstract

This BIT proposes allowing subnet owners to adjust the maximum number of UIDs on their subnet below the current fixed limit of 256. Subnet owners would be able to set their maximum UIDs down to a minimum threshold of 64, provided the new value does not fall below the subnet's current UID count.

## Motivation

Every subnet currently operates with a fixed maximum of 256 UIDs regardless of actual demand. No subnet also currently has close to that many unique entities mining, yet validators are still required to evaluate up to 256 miners every epoch. For some subnets, forming opinions on this many miners per epoch is not feasible.

## Specification

This proposal makes the existing max_allowed_uids hyperparameter modifiable by subnet owners (and root), via the sudo_set_max_allowed_uids extrinsic in the admin-utils pallet. The value can be set within the following constraints:

- The new value must be greater than or equal to the subnet's `min_allowed_uids` (default: 64).
- The new value must be greater than or equal to the subnet's current UID count (`subnetwork_n`). Reducing below the current count is not permitted.
- The new value must be less than or equal to the global `DefaultMaxAllowedUids` (256).
- The new value multiplied by the subnet's current mechanism count must not exceed `DefaultMaxAllowedUids`, to prevent chain bloat.
- The extrinsic is callable by the subnet owner or root.

## Rationale

The constraints ensure that adjusting the maximum UIDs cannot disrupt existing registered miners (no auto-deregistration), cannot exceed global bounds, and accounts for multi-mechanism subnets where the effective UID footprint is multiplied by the mechanism count.

## Backwards Compatibility

This change maintains full backwards compatibility. Existing subnets will continue operating with their current 256 UID maximum unless explicitly modified by subnet owners.

## Reference Implementation (Optional)

Implemented in `pallets/admin-utils/src/lib.rs` via `sudo_set_max_allowed_uids` (call_index 15), with supporting storage in `pallets/subtensor/src/lib.rs` (MaxAllowedUids storage map) and setter in `pallets/subtensor/src/utils/misc.rs`.

## Security Considerations

- The `max_allowed_uids` cannot be set below the current number of registered UIDs, preventing involuntary deregistration of miners.
- The mechanism count multiplier check prevents chain state bloat on multi-mechanism subnets.
- Access is restricted to subnet owners and root.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
