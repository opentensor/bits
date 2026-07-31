# BIT-XXXX: Subnet Owner Right of First Refusal

- **BIT Number:** XXXX
- **Title:** Subnet Owner Right of First Refusal
- **Author(s):** Philip Maymin (@philipmaymin)
- **Discussions-to:** https://github.com/RaoFoundation/subtensor/issues/3024
- **Status:** Draft
- **Type:** Core
- **Created:** 2026-07-31
- **Updated:** 2026-07-31
- **Requires:** 0004

## Abstract

BIT-0004 reenabled subnet deregistration. When `SubnetLimit` is reached, a new registration
evicts the non-immune subnet with the lowest `SubnetMovingPrice`, ties broken by earliest
registration. The owner of that subnet gets no notice and no recourse. They cannot outbid the
newcomer and they cannot pay to stay.

This BIT adds one step to that rule. When a registration would evict a subnet, the chain opens a
window on it rather than pruning immediately. The window records the exact lock the challenger
committed, so the challenger sets the price and the owner does not. The owner has
`FIRST_REFUSAL_WINDOW` blocks to pay that same figure. Paying recycles the TAO and renews
immunity, and refunds the challenger while removing their queue entry. Not paying means eviction
on today's terms.

Owner-only, no bid parameter, no auction. Paying is the test of whether a subnet is worth
keeping, so nothing is protected for free and the protocol makes no new judgement about subnet
quality. Mainnet has been at the cap since before this was written, so every registration on
finney today takes the eviction path.

## Motivation

Every slot on finney is full. Registering a subnet means taking one from somebody, and the v440
notes say the price of entry should fall toward the cost of the registration transaction, so it
gets cheaper to do.

The owner finds out afterward. Every miner and validator on that subnet is deregistered.
`destroy_alpha_in_out_stakes` cashes out every alpha holder near the worst price the subnet ever
had, and a low price is exactly why it got picked. The netuid goes, so every explorer listing,
dashboard, API integration, Discord channel, website and validator config downstream has to move
with it.

An owner may value all of that far above the current lock cost. It makes no difference, because
there is no call they can make.

"Just register again" is the answer this design assumes, and it is the right answer for a miner.
A miner UID is a slot number. Lose it, register again, take whatever number is free, and nothing
that identifies you has moved.

A netuid is not a slot number. It is the name. It is what the community calls the subnet, what
every dashboard and explorer indexes, what validator configs point at, and what
`get_symbol_for_subnet` derives the token ticker from, since `SYMBOLS` is indexed by netuid.
Registering again gives you a different number, a different ticker, and an alpha token with no
relationship to the one your holders were just cashed out of. There is no migration path between
them: dissolution removes `SubnetAlphaIn`, `SubnetAlphaOut` and `SubnetProtocolAlpha` outright
rather than moving them.

The owner's lock does not come back either. Per BIT-0004 step 5, the refund applies only to
subnets registered before `NetworkRegistrationStartBlock`. New subnets receive no lock refund.

The selection input is not something an owner can act on. `SubnetMovingPrice` is an EMA of the
subnet's alpha price. `update_moving_prices` runs each block over the subnets currently receiving
emission, moving each toward spot at `SubnetMovingAlpha` scaled by `b/(b+h)` for `h` =
`EMAPriceHalvingBlocks`. A subnet outside that set has its EMA frozen, and membership there is
root-gated, not owner-gated. Either way an owner cannot set the number and cannot move it
quickly, because shifting the average takes sustained buying of that subnet's alpha. So someone
who has built nothing can take the slot of someone who has, and the incumbent has no move
available at any price.

Issue #1651, which specified this deregistration design, lists "maintain **root-only** access to
direct calls for now" among its goals. The "for now" is why this is being raised rather than
treated as settled.

## Specification

### Amended subnet pruning

BIT-0004 specifies pruning as three steps. This BIT inserts a fourth between selection and
dissolution:

- **Step 1:** Exclude subnets still within `NetworkImmunityPeriod`
- **Step 2:** Among the rest, find the subnet with the lowest moving alpha price
- **Step 3:** If multiple share the same price, pick the one with the earliest registration
  timestamp
- **Step 4 (new):** If the selected subnet has no open window, open one recording the
  challenger's committed lock and the current block, and queue the challenger rather than
  dissolving. If the selected subnet has a window that has lapsed unanswered, clear it and
  dissolve as BIT-0004 specifies.

Subnets with an open, unlapsed window are skipped by selection, in the same manner as immune
subnets.

### Window state

`SubnetRefusalWindow: netuid -> (lock_amount, opened_at_block, challenger_lock_id)`.

`FIRST_REFUSAL_WINDOW` is 7,200 blocks. A window is open while
`current_block < opened_at_block + FIRST_REFUSAL_WINDOW`.

### `exercise_first_refusal(netuid)`

Callable only by `SubnetOwner` of `netuid`, and only while that subnet's window is open. Charges
the owner exactly `lock_amount` as recorded when the window opened, not the current quote.

On success:

1. The payment is recycled, following the same path as a registration lock.
2. `NetworkRegisteredAt` is set to the current block, which renews `NetworkImmunityPeriod`.
3. The challenger is refunded in full and their entry is removed from
   `NetworkRegistrationQueue`.
4. `SubnetRefusalWindow` for that netuid is cleared.

Step 3 is the part worth stating explicitly: a refused offer does not keep a place in line at a
price that was just beaten.

### `cancel_network_registration(lock_id)`

Callable by the challenger who created `lock_id`. Refunds their lock and removes the queue entry.

This exists because window expiry is lazy. A challenger whose window lapses with no further
registration arriving would otherwise sit with TAO locked and nothing block-driven to release it.

### Expiry

Expiry is lazy rather than scheduled. A lapsed window is cleared by the next registration to
reach the limit, and that registration is the one that prunes.

## Rationale

**A standing renewal flag**, where the owner pre-commits to auto-renew and the challenger's
transaction collects. No deadline, no second call, much less code. That is the version written
first, and it does not work.

Collecting inside the challenger's transaction means the chain has to know the charge will clear.
That depends on the owner's balance, and balances are public. The flag becomes a published price
for making a specific owner spend money: watch a flagged owner's balance, wait for
`get_network_lock_cost` to decay past it, then register. They pay, and you never wanted the slot
at all. A window puts the decision after the offer instead, which is where a right of first
refusal belongs. Nothing about the owner sits on chain beforehand, so there is no number to walk
down to, and they can fund during the window rather than holding TAO against a challenge that may
never come.

**An auction or bidding parameter.** Rejected: it prices the slot twice, adds a knob that needs a
defensible starting value, and turns eviction into a negotiation the protocol has to arbitrate.

**Raising `SubnetLimit` instead.** This moves the date, not the problem. `sudo_set_subnet_limit`
is root-only with no ceiling, so 128 could be 256 tomorrow, and the day 256 fills the owner is
back here. The eviction rule is what is broken.

**A conviction or stake gate on the right.** Rejected. Paying is already the test. A gate on top
of it would decide which subnets deserve to be asked, which is the judgement this proposal is
trying to keep the protocol out of.

**Letting the owner buy immunity whenever they want**, rather than only under challenge.
Rejected, though not on price. `get_network_lock_cost` quotes the same number to everyone in a
given block, so an owner buying at will pays exactly what a registrant would pay then. What
changes is whether anyone had to want the slot. Under challenge, the payment is what somebody
actually committed for your subnet. Buy-anytime, it is a function of how long it has been since
the last registration.

That is what breaks, and it compounds. Immunity takes a subnet out of the eviction pool, and once
that pool is empty `get_network_to_prune` returns `None` and the registration fails with
`SubnetLimitReached`. A failed registration never reaches `set_network_last_lock`, so the quote
keeps decaying while the network is shut. `get_lock_reduction_interval` scales the stored 115,200
by the block emission, currently 0.5 TAO, giving an effective interval of 57,600 blocks and a
fall to the 1 TAO floor about 16 days after the last successful registration. So owners who had
all bought immunity would find each renewal cheaper than the last, with nobody able to get in and
reprice it, and the 128 slots would belong permanently to whoever held them the day it started.

Gating to under-challenge inverts that. Buying requires a challenge, a challenge requires a
registration that opened a window, and that registration ratchets the price up. The pool can only
close where demand actually landed, and each closure makes the next one more expensive.

**Doing nothing.** The status quo is not neutral. Eviction falls on the lowest EMA alpha price,
and a subnet sits low while it is being built rather than traded. So the builder pays the lock,
builds, gets evicted, and pays the lock again, while a subnet with a high alpha price never faces
the question at any price.

Under this proposal that same builder pays exactly what the challenger committed, and gets
another full `NetworkImmunityPeriod`. Not a premium and not a penalty: the same figure somebody
else just put up for the same slot. It is a real cost and it lands hardest on whoever has least.
What changes is that it becomes a price they can choose to pay. Today the mechanism never asks
whether anyone wants the slot, it takes it from whoever the market has priced lowest, and that is
often whoever is still building rather than trading.

## Backwards Compatibility

No existing extrinsic changes signature or semantics. Two extrinsics are added, and one storage
map. Both new calls are permissioned and neither is reachable by an account that could not
already act in that role.

Behaviour at the cap changes, and that change is the proposal: a registration arriving at
`SubnetLimit` against a subnet with no open window now queues instead of pruning in the same
transaction. A registrant who expected eviction to complete immediately waits up to
`FIRST_REFUSAL_WINDOW` blocks, or is refunded if the owner matches. `cancel_network_registration`
exists so that wait is never open-ended without an exit.

Two costs, both measured rather than reasoned about:

- **Queue depth.** An unanswered cycle leaves one extra entry in `NetworkRegistrationQueue`,
  because a lapsed window is cleared by the next registration to arrive, and that registration is
  the one that prunes and then queues behind the original challenger. Depth climbs by one per
  unanswered cycle, bounded by the evictable pool. `NetworkRegistrationQueue` is a plain `Vec`
  decoded on every registration, so the cost is real and not cosmetic.
- **Weights.** Filling the queue in the benchmark fixture raises `register_network` proof size by
  about 861 KB, which is 3.6x. Same storage annotations, larger values.

**Leased subnets cannot use this.** `SubnetOwner` is the derived lease coldkey and nobody can
sign for it, and putting the call in the lease beneficiary proxy would let a beneficiary spend
crowdloan funds. A perpetual lease has no `end_block`, so it can never terminate and can never
answer a challenge. This is a gap, stated rather than papered over.

## Reference Implementation

https://github.com/RaoFoundation/subtensor/pull/3023

22 files against `main`, with benchmarks and tests. The author is not attached to this particular
implementation. If the mechanism is right and the implementation is wrong, that is a good outcome
and it will be rewritten.

It depends on a separate benchmark correction: `register_network`'s benchmark measures an empty
subnet map, so the weight is understated on a chain that is at the cap. That correction stands
alone and should land regardless of what is decided here.

## Security Considerations

**An owner can challenge themselves.** `do_exercise_first_refusal` checks that the caller owns
the subnet and never compares them to the challenger, and a matched challenger is refunded in
full. So an owner who is already the prune target can register from a second coldkey, open the
window on themselves, take the refund back, and match at whatever the quote was. After a full
decay that quote is the floor. The gate costs them timing and the rate limit, not the price.

This residual looks acceptable. The owner who can run it is by definition the subnet the scan was
about to evict, so the case where the trick works is the case the feature exists to serve. Read
the other way, the fix is a floor on what an owner may match, not another parameter on the
challenge.

**Griefing an owner into spending.** Addressed by design rather than left open. The window puts
the decision after the offer, so there is no on-chain flag advertising who will pay and no
balance to walk a decaying quote down to. A challenger who opens a window against an owner they
expect to match pays the lock up front and gets it back only if the owner does match, so the
attack costs them the wait and gains them nothing.

**Denial of the pool.** Windows do not accumulate protection. A subnet with an open window is
skipped by selection, but only until its window lapses, and a lapsed window is cleared by the
next registration to reach the limit. Repeated protection therefore requires repeated payment at
a price that ratchets upward with each successful registration.

**Locked funds.** A challenger's TAO is held while a window runs.
`cancel_network_registration` is the release valve, and it is callable by the challenger alone.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
