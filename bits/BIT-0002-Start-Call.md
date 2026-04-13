# BIT-0005: dTAO Subnet Start Call Function

- **BIT Number:** 0002
- **Title:** dTAO Subnet Start Call
- **Author(s):** CapriciousSage, Rhef, Vune
- **Status:** Final
- **Type:** Core
- **Created:** 28/02/2025
- **Updated:** 05/03/2025

## Abstract

Introduce a "Start Call" function to all new subnets and change the default registration toggle to `False`. Once a new subnet is registered, UID registrations would be disabled by default and subnet epoch/emissions do not start (pool locked, inflation paused) until the Subnet Owner executes both the start call function (which unlocks the pool, starts inflation, and - optionally - enables registrations).

## Motivation

Pre-dtao, new subnets experienced a 7-day immunity period, which allowed a subnet owner to verify the discord channel, publish their repo, onboard valis and miners, and 'get things rolling' - all without pressure of deregistration and without any real money on the table.

Meanwhile, dTAO creates a free-for-all 'rush' that incentivizes bogus validator or mining code to try and establish early vtrust and child hotkey delegations, undermining new subnets right out of the gate. This not only muddies the early token supply (causing undue market damage) but also undermines the subnet owner's attempts to launch the subnet correctly.

By allowing subnet owners to control when the subnet "starts", Subnet owners get a chance to establish themselves without being pushed around in their own network, and network participants get time to evaluate "do I want to mine, validate or buy this token" (minimising the chances of getting rekt on a rugnet/ponzi). It also subnet ensures registration costs remain at steady prices (maximising burn for the ecosystem) by making opportunity registrations more appealing.

## Specification

When new subnets are registered, they are effectively dormant. No tao/dtao added and no trading possible, no dtao Token Emissions, and no UID registrations.

When the subnet owner is ready to initialise the subnet, they execute the "start call" function (similar to Parachain), which unlocks the LP (allowing trading), and starts epoch/emissions, with the option to also enables UID registrations. We also propose that "start call" cannot be triggered for at least 50,400 blocks (7 days) after the subnet has been registered to minimise the chances of haphazard launches or unfounded retail FOMO.  

NOTE: Subnet owners should be able to enable UID registrations prior to/without executing the "start call" to give validators and miners an opportunity to register and deploy "real code" before the wheels start turning, however there would be a function to optionally enable both things at once depending on the flag set.

## Rationale

Every single new subnet launched since dTAO has seen it's dtao token exploited by validators and miners running fake/arbitrary code (not supplied by the subnet owner). As network participants are also attempting to 'fomo' into these new subnets based on rumours of what they might be, this has allowed the exploiters to use retail as exit liquidity (or gain an unfair advantage in the new subnet prior to its practical-launch with a proper repo).

## Backwards Compatibility

While not a backwards compatibility issue specifically, as some subnets were registered before this function was introduced, it may be worth considering a one-time "stop call" for subnets that have registered but are not yet ready to be live. This would these subnet owners the one-time option to put their not-ready-to-launch subnets back on ice until they're ready to go (disabling emissions, stopping inflation, disabling UID registrations). However, as they've already been trading on the market, locking the LP would be unreasonable.

## Reference Implementation (Optional)

N/A

## Security Considerations

May encourage subnet squatting (people buying subnets when the price drops low with the intention of selling them on the secondary market later via key swaps). It is subjective as to whether this is a good or bad thing.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).

