# BIT-0009: Native collateral support

- **BIT Number:** 0009
- **Title:** Native collateral support
- **Author(s):** Rhef
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1309546649684672582
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 2025-06-29
- **Updated:** 2025-06-29
- **Requires:** -
- **Replaces:** -

## Abstract

A change in subtensor is being proposed to add a new storage map with a key of `(netuid, hotkey)` and a value of `collateral_in_tao`.
Miners will have an option to lock Tao and then they can also pay it out, unless the validators will vote (using a new system) to burn it.
Validators may choose to not send (paid) organic traffic to miners which refuse to provide a collateral.

## Motivation

Bittensor miners often try to cheat and the subnet owners have to spend time dealing with exploits,
 which slows down subnet development. If cheating is disincentivized, subnets development should accelerate.

## Specification

Known usecases and the features they require:

| Subnet Type                                 | partial burn      | burn after dereg | deposit before registration |
| ------------------------------------------- | ----------------- | ------------------- | --------------------------- |
| **Interactive compute** | ✅ | ?     | :x:                         |
| **Job compute**    | ✅ | ✅   | :x:                         |
| **Prediction**                   | ?   | :x:                 | ✅           |
| **Storage**                      | ✅ | ✅   | :x:                         |


### Interactive compute subnets

Think sn27 Neural Internet, sn49 Polaris, sn51 Celium:
- if a miner promises to rent a machine of a certain type on demand, he needs to come through, otherwise UX is bad and the service built on top of the subnet is not attractive
- validators can refuse to weight a miner and even dereg it
- the reputational damage to the subnet can exceed one tempo of incentive + reg fee
- if the subnet has no uid pressure, reg fee falls to zero
- the owner can use tricks to prevent market from setting the reg fee, but not for long
- reg fees don't allow the collateral size to grow with the amount of machines the miner promises to rent
- reg fees cannot be paid back to the miner, so they must be small
- reg fees are all the same for all miners - if subnet has jobs that don't pose risk, miners doing those jobs have to pay the same reg fee...
- in order to prevent collateral-induced uid pressure, the collateral system MUST support burning a part of the collateral


### Job compute subnets

Think sn4 Targon, sn12 ComputeHorde:
- a miner can do jobs that can be corrupt
- it takes time to verify whether the job was corrupted or not
- it takes time for one validator to convince others that the job was indeed corrupted
- a corrupted job has causes damage to the subnet reputation
- validators using this compute won't misdistribute weights, but their performance will suffer and their reputation may suffer too
- in order for this to work at all, the collateral system MUST support burning collateral _after_ the uid is deregistered


### Prediction subnets (sn57)
- impossible to evaluate a trading strategy on historical data
- high leverage extreme risk strategies are worthless
- if these can be submitted without any limits, miners will do that
- some of the strategies will be lucky
- immunity period insufficient with a 256/1024 uid limit
- a miner can submit a collateral and (timelocked?) trades before registering
- if a strategy will be viable, the collateral may be returned
- if the strategy fails, the collateral will be burned
- this increases the cost of the attack and makes it not viable anymore
- in order for the collateral system to properly support prediction subnets, there MUST be a way to pay in collateral _before_ uid registration
- should collaterals be returned or burned when the subnet is deregistered?
- evm key association might need a change from uid to hotkey
- evm keys can then be associated after paying collateral but before registering an uid
- we have a smart contract that is able to store data on chain cost-effectively


### Storage subnets
- a validator trusts a miner with data (fragments) which the miner should return after getting deregged
- if the miner doesn't return the data that the validators entrusted him with, the validator should regenarate it so it won't cause data loss
- regeneration is expensive
- if many miners run away with data at the same time, eventually data loss can occur
- if the subnet is going to hold a meaningful amount of data (petabytes), then the miners must be incentivized to return the data to the subnet once they are deregged
- the validators will burn the collateral of the miner a few days after he's deregged, unless he returns the data back to the subnet
- in order for the collateral system to properly support storage subnets, there MUST be a way to slash the miners after uid deregistration

## User interface

### CLI
```awk
btcli wallet collateral add --name coldkey-1 --hotkey hotkey-1 --netuid 123 --amount 10
btcli wallet collateral schedule_reclaim --name coldkey-1 --hotkey hotkey-1 --netuid 123 --amount 10
btcli wallet collateral status --name coldkey-1 --hotkey hotkey-1
btcli wallet collateral status --name coldkey-1 --hotkey hotkey-1 --netuid 123
```

### SDK
```py
subnet = Bittensor(network="test").subnet(netuid=123)
print(subnet.collaterals[hotkey])

reclaim_attempt = subnet.get_collateral_reclaim_attempts(hotkey=None)[0]  # hotkey=None means get reclaim attempts for all hotkeys
print(
    reclaim_attempt.hotkey,
    reclaim_attempt.amount,
    reclaim_attempt.expiry_block,
)

subnet.collateral_burn_vote(
    {
        hotkey: how_much_to_burn,
        other_hotkey: other_how_much_to_burn,
    },
)  # how_much_to_burn=1 means burn entire collateral
```

## Rationale

### Hotkey or uid
Whether we use `(netuid, uid)` or `(netuid, hotkey)` doesn't make a lot of difference from the implementation or storage standpoint, but as it can be seen in the table above, using hotkey allows for far more functionality (paying collateral before registration and burning collateral after deregistration). 

### Reclaimation
There are several ways this can be done:
- Reclaim only during uid / subnet deregistration (easiest)
- Reclaim immediately (bad for some types of subnets)
- Reclaim after a delay, say 60 tempos, where the reclaim doesn't actually go through if the validators vote to burn it before the delay expires

### Voting participation
Since weight copiers won't vote in the collateral system, we need to exclude their stake from the calculation and honest validators should vote `0` if they would prefer to not slash.

### Flashloans
Anti-MEV measures will foil flashloans

## Backwards Compatibility

Collaterals will be entirely optional.


## Security Considerations

The collateral system may be abused in a number of ways. The governor should have the ability to turn off
 the collateral system for a given netuid.
A governance policy of turning the collateral system off for subnets that abuse it should be added
 to make it clear for everyone what type of usage is allowed.

The collateral system is meant to be burned automatically when a miner egregiously breaches the rules of the subnet in such a way, that pulling weights and deregging him is not a sufficient deterrent.
If a subnet will be caught issuing burns manually or carelessly, their privilege to use the collateral should be revoked by the governor.


## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
