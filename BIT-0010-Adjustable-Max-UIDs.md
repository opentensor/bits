# BIT-0010: Adjustable Max UIDs

- **BIT Number:** 0010
- **Title:** Adjustable Max UIDs
- **Author(s):** Namoray, "Maciej Kula"
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1389781506444230656
- **Status:** Draft
- **Type:** Core
- **Created:** 02/07/2025
- **Updated:** 02/07/2025
- **Requires:** -
- **Replaces:** -

## Abstract

This BIT proposes allowing subnet owners to reduce the maximum number of UIDs on their subnet below the current fixed limit of 256. Subnet owners would be able to set their maximum UIDs down to a minimum threshold of 64, with changes taking effect immediately and excess UIDs being deregistered based on lowest emissions first.

## Motivation

Every subnet currently operates with a fixed maximum of 256 UIDs regardless of actual demand. No subnet also currently has close to that many unique entities mining, yet validators are still required to evaluate up to 256 miners every epoch. For some subnets, forming opinions on this many miners per epoch is not feasible.

## Specification

This proposal would make the existing `max_uids` hyperparameter modifiable by subnet owners, allowing them to reduce it from the current fixed value of 256 down to a minimum threshold between 32-64 UIDs, with 64 being the preferred lower bound.

When a subnet owner reduces the `max_uids` value, the change would take effect immediately in the next epoch. Any UIDs beyond the new maximum would be automatically deregistered in order of emissions, with lowest-performing UIDs removed first. All UIDs would be eligible for deregistration, not just zero-emission UIDs.

## Rationale

TBD

## Backwards Compatibility

This change maintains full backwards compatibility. Existing subnets will continue operating with their current 256 UID maximum unless explicitly modified by subnet owners.

## Reference Implementation (Optional)

TBD (to be developed following community consensus on technical approach)

## Security Considerations

TBD

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
