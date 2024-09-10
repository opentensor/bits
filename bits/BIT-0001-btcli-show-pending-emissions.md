# BIT-0001: Let btcli show pending emissions

- **BIT Number:** 0001
- **Title:** Let btcli show pending emissions
- **Author(s):** µ <maarten@coldint.io>
- **Discussions-to:** ?
- **Status:** Draft
- **Type:** Interface
- **Created:** 20240910
- **Updated:** 20240910
- **Requires:** N/A
- **Replaces:** N/A

## Abstract

Since September 9th 2024 emissions are aggregated in the runtime state in a
hotkey:amount map, in order to be periodically added to the coldkey balance.
This "PendingdHotkeyEmission" value can be queried using substrate, but this
seems a bit convoluted for non-developers. This BIT proposes to add a feature
to btcli to query the PendingdHotkeyEmission as part of the wallet balance
query.

## Motivation

The existing protocol specification and related tools requires an ordinary
user, that wants to know e.g. whether his mining activities are indeed paying
off, within a shorter timeframe than 24 hours, to learn about interacting with
the substrate layer using other tools than btcli.

This is a bit much to ask of an ordinary user, and it opens up a very real
possibility that other, possibly malicious, actors will fill the gap with third
party tools.

Also this will greatly reduce the amount of confusion and Q&A resulting from
this change in emission timing.


## Specification

Something along these lines should be put in btcli wherever the balance of a
coldkey is fetched:

```
def get_coldkey_pendinghotkeyemission(substrate,coldkey):
    e = 0
    for hotkey in substrate.query('SubtensorModule','OwnedHotkeys',[coldkey]).value:
        e += substrate.query('SubtensorModule','PendingdHotkeyEmission',[hotkey]).value
    return e/1e9
```

Then the resulting value should be output along with the balance; either the
grand total or the value per hotkey, depending on context.

## Rationale

No alternative designs were considered. The impact is negligible.

## Backwards Compatibility

The output of btcli should now contain a new field alongside 'balance' and
'staked' amounts.

This may break code that depends on the exact output of btcli. This is
intentional. Code should not depend on the exact output of btcli. Rather it
should process structured output (perhaps btcli already has a --json flag for
this, and if not, it should get one).

## Reference Implementation (Optional)

N/A

## Security Considerations

The security of Bittensor is improved by this BIT, by improving clarity for end
users and minimizing the amount of uncertainty and related attack surface for
scammers.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
