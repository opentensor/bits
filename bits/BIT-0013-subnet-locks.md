# BIT-0013: Stake locks

- **BIT Number:** 0013
- **Title:** Stake Locks
- **Author(s):** Greg Zaitsev
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-07-23
- **Updated:** 2025-07-23

**Note:** This BIT in fact tables the (PR #1860)[https://github.com/opentensor/subtensor/pull/1860].

## 🔍 Abstract

This BIT proposes introduction of stake locks, where lock conviction will determine subnet ownership.

## 🔧 Motivation

There are some low quality subnets in the ecosystem, we can call them abandoned subnets. They are not actively managed by their owners and (1) are the source of attacks on the bittensor network because they can be occupied/taken over by malignant neurons, and (2) waste bittensor compute resource. One way to address the issue is allowing the subnet to be taken over by staking and locking a significant amount of TAO.

## 🧪 Specification

- Change subnet owner based on subnet stake lock. Locks are linear with 100% being locked at the start block and 0% on the end block. Any staker can lock their stake. Approximately once per month (7200 * 30 blocks) the lock EMAs are recalculated and, if stake lock EMA is the highest for some account, it becomes the owner of the subnet.
- If stake is locked, then only unlocked portion of it can be unstaked at a given block.
- In order to prevent subnet sniping, the lock EMA is applied, which gives subnet owner enough warning to act.
- Conviction should also increase with higher lock duration (not implemented in reference impl).

### State maps

Additional state maps to handle the stake locks are:

```rust
/// ======================
/// ==== Stake locks =====
/// ======================
#[pallet::storage]
/// --- ITEM ( lock_interval_blocks ) | Stake lock EMA half-life factor
pub type LockIntervalBlocks<T: Config> =
    StorageValue<_, u64, ValueQuery, DefaultLockIntervalBlocks<T>>;

#[pallet::storage]
/// --- NMAP ( netuid, hot, cold ) --> stake_lock | Returns the stake_lock struct for netuid, hot and cold triplet.
pub type Locks<T: Config> = StorageNMap<
    _,
    (
        NMapKey<Blake2_128Concat, NetUid>, // subnet
        NMapKey<Identity, T::AccountId>,   // hot
        NMapKey<Identity, T::AccountId>,   // cold
    ),
    StakeLock,
    ValueQuery,
>;

#[pallet::storage]
/// --- NMAP ( netuid, hot, cold ) --> stake conviction ema | Returns the stake conviction EMA for netuid, hot and cold triplet.
pub type ConvictionEma<T: Config> = StorageNMap<
    _,
    (
        NMapKey<Identity, NetUid>,               // subnet
        NMapKey<Blake2_128Concat, T::AccountId>, // hot
        NMapKey<Blake2_128Concat, T::AccountId>, // cold
    ),
    AlphaCurrency,
    ValueQuery,
>;
```

### Extrinsic to lock stake

```rust
pub fn lock_stake(
    origin: OriginFor<T>,
    hotkey: T::AccountId,
    netuid: NetUid,
    duration: u64,
    alpha_locked: AlphaCurrency,
) -> DispatchResult {
    Self::do_lock(origin, hotkey, netuid, duration, alpha_locked)
}
```

### Extrinsic implementation

```rust
use super::*;
use safe_math::*;
use substrate_fixed::types::{I96F32, U64F64};
use subtensor_runtime_common::NetUid;

#[freeze_struct("f92b0bb7408af4d8")]
#[derive(
    Clone, Copy, Decode, Default, Encode, Eq, MaxEncodedLen, PartialEq, RuntimeDebug, TypeInfo,
)]
pub struct StakeLock {
    pub alpha_locked: AlphaCurrency,
    pub start_block: u64,
    pub end_block: u64,
}

impl<T: Config> Pallet<T> {
    /// Sets the lock interval in blocks.
    ///
    /// This function updates the minimum duration for which stakes can be locked.
    ///
    /// # Arguments
    ///
    /// * `new_interval` - The new lock interval in blocks.
    ///
    /// # Events
    ///
    /// Emits a `LockIntervalSet` event with the new interval value.
    pub fn set_lock_interval_blocks(new_interval: u64) {
        // Update the lock interval storage
        LockIntervalBlocks::<T>::put(new_interval);

        // Emit an event for the new lock interval
        Self::deposit_event(Event::LockIntervalSet { new_interval });
    }

    /// Gets the current lock interval in blocks.
    ///
    /// This function retrieves the current value of the lock interval.
    ///
    /// # Returns
    ///
    /// * `u64` - The current lock interval in blocks.
    pub fn get_lock_interval_blocks() -> u64 {
        LockIntervalBlocks::<T>::get()
    }

    /// Calculates the conviction score for a specific hotkey and coldkey pair on a given subnet.
    ///
    /// This function retrieves the locked stake amount from the `Locks` storage and calculates
    /// the conviction score based on the locked amount and the lock duration.
    ///
    /// # Arguments
    ///
    /// * `hotkey` - The hotkey account ID.
    /// * `coldkey` - The coldkey account ID.
    /// * `netuid` - The subnet ID.
    ///
    /// # Returns
    ///
    /// * `AlphaCurrency` - The conviction score calculated from the locked stake.
    pub fn get_conviction_for_hotkey_and_coldkey_on_subnet(
        hotkey: &T::AccountId,
        coldkey: &T::AccountId,
        netuid: NetUid,
    ) -> AlphaCurrency {
        let stake_lock = Locks::<T>::get((netuid, hotkey.clone(), coldkey.clone()));

        Self::calculate_conviction(&stake_lock, Self::get_current_block_as_u64())
    }

    /// Locks a specified amount of stake for a given duration on a subnet.
    ///
    /// This function allows a user to lock their stake, increasing their conviction score.
    /// The locked stake cannot be withdrawn until the lock period expires, and the new lock
    /// must not decrease the current conviction score.
    ///
    /// # Arguments
    ///
    /// * `origin` - The origin of the call, must be signed by the coldkey.
    /// * `hotkey` - The hotkey associated with the stake to be locked.
    /// * `netuid` - The ID of the subnet where the stake is locked.
    /// * `duration` - The duration (in blocks) for which the stake will be locked.
    /// * `alpha_locked` - The amount of stake to be locked.
    ///
    /// # Returns
    ///
    /// * `DispatchResult` - The result of the lock operation.
    ///
    /// # Errors
    ///
    /// * `SubnetNotExists` - If the specified subnet does not exist.
    /// * `HotKeyAccountNotExists` - If the hotkey account does not exist.
    /// * `HotKeyNotRegisteredInSubNet` - If the hotkey is not registered on the specified subnet.
    /// * `NotEnoughStakeToWithdraw` - If the user doesn't have enough stake to lock, or if the new lock would decrease the current conviction.
    ///
    /// # Events
    ///
    /// * `LockIncreased` - Emitted when the lock is successfully increased.
    ///
    /// # TODO
    ///
    /// * Consider implementing a maximum lock duration to prevent excessively long locks.
    /// * Implement a mechanism to partially unlock stakes as the lock period progresses.
    /// * Add more granular error handling for different failure scenarios.
    pub fn do_lock(
        origin: T::RuntimeOrigin,
        hotkey: T::AccountId,
        netuid: NetUid,
        duration: u64,
        alpha_locked: AlphaCurrency,
    ) -> dispatch::DispatchResult {
        // Step 1: Validate inputs and check conditions
        // Ensure the origin is valid.
        let coldkey = ensure_signed(origin)?;

        // Ensure that the subnet exists.
        ensure!(Self::if_subnet_exist(netuid), Error::<T>::SubnetNotExists);

        // Ensure that the hotkey account exists.
        ensure!(
            Self::hotkey_account_exists(&hotkey),
            Error::<T>::HotKeyAccountNotExists
        );

        // Ensure the the lock is above zero.
        ensure!(
            alpha_locked > 0.into(),
            Error::<T>::NotEnoughStakeToWithdraw
        );

        // Ensure the lock duration is at least the minimum
        ensure!(
            duration >= DefaultMinLockDuration::<T>::get(),
            Error::<T>::DurationTooShort
        );

        // Get the lockers current stake.
        let current_alpha_stake =
            Self::get_stake_for_hotkey_and_coldkey_on_subnet(&hotkey, &coldkey, netuid);

        // Ensure that the caller has enough stake to lock.
        ensure!(
            alpha_locked <= current_alpha_stake,
            Error::<T>::NotEnoughStakeToWithdraw
        );

        // Step 2: Calculate and compare convictions
        // Get the current block.
        let current_block = Self::get_current_block_as_u64();
        let new_end_block = current_block.saturating_add(duration);

        // Check that we are not decreasing the current conviction.
        if Locks::<T>::contains_key((netuid, hotkey.clone(), coldkey.clone())) {
            // Get the current lock.
            let stake_lock = Locks::<T>::get((netuid, &hotkey, &coldkey));

            // Calculate the current conviction.
            let current_conviction = Self::calculate_conviction(&stake_lock, current_block);

            // Calculate the new conviction.
            let new_conviction = Self::calculate_conviction(
                &StakeLock {
                    alpha_locked,
                    start_block: current_block,
                    end_block: new_end_block,
                },
                current_block,
            );

            // Ensure the new lock does not decrease the current conviction
            ensure!(
                new_conviction >= current_conviction,
                Error::<T>::NotEnoughStakeToWithdraw
            );
        }

        // Step 3: Set the new lock
        Locks::<T>::insert(
            (netuid, hotkey.clone(), coldkey.clone()),
            StakeLock {
                alpha_locked,
                start_block: current_block,
                end_block: current_block.saturating_add(duration),
            },
        );

        // Step 4: Emit event and return
        // Lock increased event.
        log::info!(
            "LockIncreased( coldkey:{:?}, hotkey:{:?}, netuid:{:?}, alpha_locked:{:?} )",
            coldkey.clone(),
            hotkey.clone(),
            netuid,
            alpha_locked
        );
        Self::deposit_event(Event::LockIncreased {
            coldkey: coldkey.clone(),
            hotkey: hotkey.clone(),
            netuid,
            alpha_locked,
        });

        // Ok and return.
        Ok(())
    }

    /// Updates the stake lock EMAs and owners of all subnets periodically.
    ///
    /// This function checks if it's time to update subnet owners based on the current block number
    /// and a predefined update interval. If the condition is met, it iterates through all subnet
    /// network IDs and calls the `update_subnet_owner` function for each subnet.
    ///
    /// # Details
    /// - The update interval is set to 7200 * 15 blocks (approximately 15 days, assuming 7200 blocks per day).
    /// - The update is triggered every two intervals (30 days) when the current block number is divisible by twice the update interval.
    ///
    /// # Effects
    /// - When the update condition is met, it calls `update_subnet_owner` for each subnet,
    ///   potentially changing the owner of each subnet based on conviction scores.
    pub fn update_stake_locks(current_block: u64) {
        let update_interval = 216_000; // Approx 30 days.
        if current_block.checked_rem(update_interval).unwrap_or(1) == 0 {
            for netuid in Self::get_all_subnet_netuids() {
                Self::update_subnet_owner(netuid, update_interval);
            }
        }
    }

    /// Calculates the exponentially moving average (EMA) of conviction for a given (hotkey, coldkey) pair in a subnet.
    ///
    /// # Arguments
    ///
    /// * `netuid` - The identifier of the subnet.
    /// * `update_period` - The number of blocks since the last update. Used to compute the smoothing factor.
    /// * `conviction` - The current conviction value to blend into the EMA.
    /// * `hotkey` - The hotkey account associated with the stake lock.
    /// * `coldkey` - The coldkey account associated with the stake lock.
    ///
    /// # Returns
    ///
    /// Returns the updated conviction EMA as a `u64`.
    ///
    /// # Description
    ///
    /// This function uses the formula:
    ///
    /// ```text
    /// new_ema = old_ema * (1 - alpha) + conviction * alpha
    /// where alpha = update_period / lock_interval
    /// ```
    ///
    /// - If `alpha` (smoothing factor) exceeds `1.0`, it is capped at `1.0`.
    /// - `ConvictionEma` is retrieved from storage for the key `(netuid, hotkey, coldkey)`.
    /// - Floating point arithmetic is performed using `U64F64` fixed-point type to maintain precision.
    ///
    /// # Notes
    ///
    /// - This function is pure: it does not mutate any storage.
    /// - Use the result to update `ConvictionEma` if necessary.
    /// - Zero LockIntervalBLocks will result in constant EMA
    ///
    pub fn get_conviction_ema(
        netuid: NetUid,
        update_period: u64,
        conviction: AlphaCurrency,
        hotkey: &T::AccountId,
        coldkey: &T::AccountId,
    ) -> AlphaCurrency {
        let one = U64F64::saturating_from_num(1.0);
        let zero = U64F64::saturating_from_num(1.0);
        let lock_interval_blocks = U64F64::saturating_from_num(Self::get_lock_interval_blocks());
        let mut smoothing_factor =
            U64F64::saturating_from_num(update_period).safe_div_or(lock_interval_blocks, zero);
        if smoothing_factor > one {
            smoothing_factor = one;
        }

        let old_ema =
            U64F64::saturating_from_num(ConvictionEma::<T>::get((netuid, hotkey, coldkey)));

        AlphaCurrency::from(
            old_ema
                .saturating_mul(one.saturating_sub(smoothing_factor))
                .saturating_add(
                    smoothing_factor.saturating_mul(U64F64::saturating_from_num(conviction)),
                )
                .saturating_to_num::<u64>(),
        )
    }

    /// Determines the subnet owner based on the highest conviction score.
    ///
    /// This function calculates the conviction score for each hotkey in the subnet,
    /// considering the lock amount and duration. The hotkey with the highest total
    /// conviction score becomes the subnet owner.
    ///
    /// # Arguments
    /// * `netuid` - The network ID of the subnet
    /// * `update_period` - How frequently this call is made (for EMA calculation)
    ///
    /// # Effects
    /// * Updates the SubnetOwner storage item with the coldkey of the highest conviction hotkey
    pub fn update_subnet_owner(netuid: NetUid, update_period: u64) {
        let current_block = Self::get_current_block_as_u64();

        // Get the updated current owner's conviction first
        let owner_coldkey = SubnetOwner::<T>::get(netuid);
        let owner_hotkey = SubnetOwnerHotkey::<T>::get(netuid);
        let owner_lock = Locks::<T>::get((netuid, owner_hotkey.clone(), owner_coldkey.clone()));
        let updated_owner_conviction = Self::calculate_conviction(&owner_lock, current_block);
        let mut updated_owner_conviction_ema = Self::get_conviction_ema(
            netuid,
            update_period,
            updated_owner_conviction,
            &owner_hotkey,
            &owner_coldkey,
        );
        let mut new_owner_coldkey = owner_coldkey.clone();
        let mut new_owner_hotkey = owner_hotkey.clone();
        let mut owner_updated = false;
        let mut total_conviction = AlphaCurrency::from(0);

        for ((hotkey, coldkey), stake_lock) in Locks::<T>::iter_prefix((netuid,)) {
            // Update EMAs. The update value depends on the update_period so that even if we change how
            // frequently we update EMAs, the EMA curve doesn't change (except getting less or more
            // accurate)

            let new_conviction = Self::calculate_conviction(&stake_lock, current_block);
            let new_ema =
                Self::get_conviction_ema(netuid, update_period, new_conviction, &hotkey, &coldkey);
            ConvictionEma::<T>::insert((netuid, hotkey.clone(), coldkey.clone()), new_ema);
            total_conviction = total_conviction.saturating_add(new_conviction);

            // In case of a tie, lower value coldkey wins
            if (new_ema > updated_owner_conviction_ema)
                || (new_ema == updated_owner_conviction_ema && coldkey < new_owner_coldkey)
            {
                new_owner_coldkey = coldkey;
                new_owner_hotkey = hotkey;
                updated_owner_conviction_ema = new_ema;
                owner_updated = true;
            }
        }

        // Implement a minimum conviction threshold for becoming a subnet owner
        let min_conviction_threshold = AlphaCurrency::from(1000); // TODO: adjust as needed
        if total_conviction < min_conviction_threshold {
            owner_updated = false;
        }

        // Set the subnet owner to the coldkey of the hotkey with highest conviction
        if owner_updated {
            SubnetOwner::<T>::insert(netuid, new_owner_coldkey.clone());
            SubnetOwnerHotkey::<T>::insert(netuid, new_owner_hotkey.clone());
        }

        // Update subnet locked
        SubnetLocked::<T>::insert(netuid, total_conviction);
    }

    /// Calculates the conviction score for a locked stake.
    ///
    /// This function computes a conviction score based on the amount of locked stake, the time
    /// this lock existed (since start_block) and the remaining lock duration. The score increases
    /// with both the lock amount and duration, but with diminishing returns for longer lock
    /// periods.
    ///
    /// # Arguments
    ///
    /// * `lock_amount` - The amount of stake locked, as a u64.
    /// * `start_block` - The block number when the lock was set, as a u64.
    /// * `end_block` - The block number when the lock expires, as a u64.
    /// * `current_block` - The current block number, as a u64.
    ///
    /// # Returns
    ///
    /// * A u64 representing the calculated conviction score.
    ///
    /// # Formula
    ///
    /// The conviction score is linear of blocks, starting with 100% locked at start_block and going
    /// down to 0% locked at the end_block:
    ///
    ///   conviction = alpha_locked * (ax + b),
    ///
    /// where a = 1 / (start_block - end_block)
    ///       b = end_block / (end_block - start_block)
    ///       x is current block
    ///
    pub fn calculate_conviction(lock: &StakeLock, current_block: u64) -> AlphaCurrency {
        // Handle corner cases first (with 100% precision)
        if current_block < lock.start_block {
            return 0.into();
        } else if current_block == lock.start_block {
            return lock.alpha_locked;
        } else if current_block >= lock.end_block {
            return 0.into();
        }

        // Handle the cases between start and end
        let lock_duration =
            I96F32::saturating_from_num(lock.end_block.saturating_sub(lock.start_block));
        let minus_one = I96F32::saturating_from_num(-1);
        let a = minus_one.safe_div(lock_duration);
        let b = I96F32::saturating_from_num(lock.end_block).safe_div(lock_duration);
        let x = I96F32::saturating_from_num(current_block);
        let locked_alpha_fixed = I96F32::saturating_from_num(lock.alpha_locked);
        let conviction_score =
            locked_alpha_fixed.saturating_mul(a.saturating_mul(x).saturating_add(b));

        AlphaCurrency::from(conviction_score.saturating_to_num::<u64>())
    }

    pub fn check_locks_on_stake_reduction(
        hotkey: &T::AccountId,
        coldkey: &T::AccountId,
        netuid: NetUid,
        alpha_unstaked: AlphaCurrency,
    ) -> dispatch::DispatchResult {
        if Locks::<T>::contains_key((netuid, &hotkey, &coldkey)) {
            let total_stake =
                Self::get_stake_for_hotkey_and_coldkey_on_subnet(hotkey, coldkey, netuid);
            let current_block = Self::get_current_block_as_u64();
            // Retrieve the lock information for the given netuid, hotkey, and coldkey
            let stake_lock = Locks::<T>::get((netuid, hotkey.clone(), coldkey.clone()));
            let conviction = Self::calculate_conviction(&stake_lock, current_block);

            let stake_after_unstake = total_stake.saturating_sub(alpha_unstaked);
            // Ensure the requested unstake amount is not more than what's allowed
            ensure!(
                stake_after_unstake >= conviction,
                Error::<T>::NotEnoughStakeToWithdraw
            );
            // If conviction is 0, remove the lock
            if conviction == 0.into() {
                Locks::<T>::remove((netuid, hotkey.clone(), coldkey.clone()));
            }
        }

        Ok(())
    }
}
```

### Events

```rust
/// The new stake lock interval half-life factor was set
LockIntervalSet {
    /// New interval value (in blocks)
    new_interval: u64,
},

/// Stake lock has increased
LockIncreased {
    /// The owner coldkey of the stake
    coldkey: T::AccountId,
    /// The hotkey the stake is made to
    hotkey: T::AccountId,
    /// Subnet ID
    netuid: NetUid,
    /// Amount of alpha locked
    alpha_locked: AlphaCurrency,
},
```

### Additional check in remove_stake and all other extrinsics where stake is reduced

```rust
// Check stake locks
Self::check_locks_on_stake_reduction(
    origin_hotkey,
    origin_coldkey,
    origin_netuid,
    alpha_amount,
)?;
```

### Handling locks in swap coldkey

```rust
// 3. Swap Stake.
...

let stake_lock_old = Locks::<T>::get((netuid, &hotkey, old_coldkey));

...

// Merge locks
let stake_lock_new = Locks::<T>::get((netuid, &hotkey, new_coldkey));
let stake_lock_merged = StakeLock {
    alpha_locked: stake_lock_old
        .alpha_locked
        .saturating_add(stake_lock_new.alpha_locked),
    start_block: stake_lock_old.start_block.max(stake_lock_new.start_block),
    end_block: stake_lock_old.end_block.max(stake_lock_new.end_block),
};
Locks::<T>::insert((netuid, &hotkey, old_coldkey), stake_lock_merged);

// Remove old stake lock
Locks::<T>::remove((netuid, &hotkey, old_coldkey));
```

## 📘 Reference Implementation

The feature was once implemented and tabled:
https://github.com/opentensor/subtensor/pull/1860

## Test Plan

### Lock Stake: Success Cases
- [ ] `test_do_lock_success`  
  Lock a portion of staked tokens.
- [ ] `test_do_lock_hotkey_not_registered`  
  Lock stake from a different coldkey than original registration.
- [ ] `test_do_lock_max_duration`  
  Lock stake with maximum allowed duration.
- [ ] `test_do_lock_multiple_times`  
  Lock multiple times, updating the locked amount and duration.
- [ ] `test_do_lock_different_subnets`  
  Lock stake independently on different subnets.
- [ ] `test_do_lock_increase_conviction`  
  Increase locked stake amount and duration.

### Lock Stake: Failure Cases
- [ ] `test_do_lock_subnet_does_not_exist`  
  Lock on non-existent subnet.
- [ ] `test_do_lock_hotkey_does_not_exist`  
  Lock from non-existent hotkey.
- [ ] `test_do_lock_zero_amount`  
  Lock with zero amount.
- [ ] `test_do_lock_insufficient_stake`  
  Lock more than available stake.
- [ ] `test_do_lock_decrease_conviction`  
  Decrease conviction by reducing locked amount or duration.
- [ ] `test_do_lock_too_short`  
  Lock duration is too short.

### Remove Stake: Lock Enforcement
- [ ] `test_remove_stake_fully_locked`  
  Cannot remove fully locked stake.
- [ ] `test_remove_stake_partially_locked`  
  Remove only unlocked portion if partially locked.
- [ ] `test_remove_stake_after_lock_expiry`  
  Full stake removal after lock expiry.
- [ ] `test_remove_stake_multiple_locks`  
  Prevent removal of locked stake exceeding unlockable amount.
- [ ] `test_remove_stake_conviction_calculation`  
  Validate conviction calculation before removal.
- [ ] `test_remove_stake_partial_lock_removal`  
  Partial stake removal preserves lock.
- [ ] `test_remove_stake_full_lock_removal`  
  Full removal after lock expiry clears lock.
- [ ] `test_remove_stake_across_subnets`  
  Remove stake separately across multiple subnets.

### Conviction Calculation
- [ ] `test_calculate_conviction_zero_lock_amount`  
  Conviction is 0 when locked amount is 0.
- [ ] `test_calculate_conviction_zero_duration`  
  Conviction is 0 when duration is 0.
- [ ] `test_calculate_conviction_max_lock_amount`  
  Conviction scales with lock amount.
- [ ] `test_calculate_conviction_max_duration`  
  Conviction scales with lock duration.
- [ ] `test_calculate_conviction_overflow_check`  
  Overflow-safe calculation.
- [ ] `test_calculate_conviction_precision_small_values`  
  High precision maintained for small values.
- [ ] `test_calculate_conviction_precision_large_values`  
  Large values are preserved with precision.
- [ ] `test_calculate_conviction_rounding`  
  Conviction increases with longer duration.
- [ ] `test_calculate_conviction_expired_lock`  
  Conviction drops to zero after lock expiry.
- [ ] `test_calculate_conviction_lock_interval_boundary`  
  Conviction boundary at lock interval.
- [ ] `test_calculate_conviction_consistency`  
  Conviction increases consistently with lock amount or duration.

### Conviction EMA Calculation
- [ ] `test_conviction_ema_basic`  
  EMA starts from 0.
- [ ] `test_conviction_ema_existing_ema`  
  EMA with existing value converges.
- [ ] `test_conviction_ema_zero_update`  
  EMA doesn't change with update period = 0.
- [ ] `test_conviction_ema_zero_lockint`  
  EMA equals conviction when lock interval is 0.
- [ ] `test_conviction_ema_large_update`  
  EMA capped at conviction when update period > lock interval.
- [ ] `test_conviction_ema_conviction_0`  
  EMA decays when conviction = 0.
- [ ] `test_conviction_ema_conviction_max`  
  EMA increases when conviction = max.

### Subnet Ownership and Lock Updates
- [ ] `test_update_subnet_owner_no_locks`  
  No lock → no subnet owner.
- [ ] `test_update_subnet_owner_single_lock`  
  Single lock → sets owner and updates locked value.
- [ ] `test_update_subnet_owner_multiple_locks`  
  Multiple locks → pick owner with highest conviction.
- [ ] `test_update_subnet_owner_tie_breaking`  
  Tie-breaking when convictions are equal.
- [ ] `test_update_subnet_owner_below_threshold`  
  Conviction below threshold → no owner set.
- [ ] `test_update_subnet_owner_ownership_change`  
  Conviction changes over time updates owner/locked value.
- [ ] `test_update_subnet_owner_storage_updates`  
  Verify correct storage updates across block intervals.
- [ ] `test_update_subnet_owner_conviction_calculation`  
  Conviction calculated properly across different stake locks.
- [ ] `test_update_subnet_owner_different_subnets`  
  Subnet updates handle multiple subnets independently.
- [ ] `test_update_subnet_owner_large_subnet`  
  Scalability test with large subnet (1500 locks).
- [ ] `test_locks_are_updated_in_block_step`  
  Subnet owner auto-updates at block step.


## 💬 Discussion

- Consider support of the current owner (or any owner-to-be) by other keys, i.e. allow others to join their locks

## © Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
