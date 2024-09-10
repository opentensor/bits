# BIT-0002: Improve emission drain timing

- **BIT Number:** 0002
- **Title:** Improve emission drain timing
- **Author(s):** µ <maarten@coldint.io>
- **Discussions-to:** ?
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 20240910
- **Updated:** 20240910
- **Requires:** N/A
- **Replaces:** N/A

## Abstract

Since September 9th 2024 emissions are aggregated in the runtime state in a
hotkey:amount map, in order to be periodically added ("drained") to the coldkey
balance.  The period is currently 7200 blocks (~24 hours) and the offset within
this period is fixed for each hotkey. This BIT proposed to add features that
add some flexibility to the timing of these drain events. One approach is to
have a call that triggers draining the pending emission on a hotkey. Another
approach is to drain the pending emission on a hotkey once the amount reaches a
particular threshold value.

## Motivation

The existing protocol specification is rigid in its scheduling of drain events,
giving wallet owners no control over the timing of the drain event.

There are several reasons why wallet owners would like to have faster (or
immediate) access to their funds:
- they might want to respond to evolving market conditions
- they might want to ensure they actually receive emissions
- they might want to synchronize emissions from various keys for (tax)
  accounting purposes, to eachother or to a particular time of day

Also this will reduce the amount of confusion and Q&A resulting from this
change in emission timing.

Two approaches are suggested, that are not mutually exclusive:

1) let the wallet owner trigger the drain event using an extrinsic

2) automatically trigger the drain event as soon as a value threshold is reached

## Specification

### 1) let the wallet owner trigger the drain event using an extrinsic

An additional extrinsic could be implemented to trigger the function

````
    pallets/subtensor/src/coinbase/run_coinbase.rs:drain_hotkey_emission()
````

either directly, or indirectly by altering

````
    pallets/subtensor/src/coinbase/run_coinbase.rs:should_drain_hotkey()
````

so that it can read a flag that can be set using an extrinsic that requests
draining the pending emissions.

Of course, whatever its final form will be, this extrinsic should carry a fee
and/or a rate limit to prevent abuse.

### 2) automatically trigger the drain event as soon as a value threshold is reached

Alter this line:

````
    pallets/subtensor/src/coinbase/run_coinbase.rs:drain_hotkey_emission()

        if Self::should_drain_hotkey(&hotkey, current_block, emission_tempo)
````

by adding `|| hotkey_emission > threshold` with `threshold` some value, e.g. 1 Tao.

## Rationale

The rationale of this change is that it restores how wallet owners can
immediately access their new funds, while keeping the advantages of
emission aggregation regarding chain bloat reduction.

No alternative designs were considered. The impact is negligible.

## Backwards Compatibility

This BIT does not introduce backward incompatibilities.

## Reference Implementation (Optional)

N/A

## Security Considerations

The security of Bittensor is improved by this BIT, by improving clarity for end
users and minimizing the amount of uncertainty and related attack surface for
scammers.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
