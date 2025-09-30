# BIT-XXXX: Synchronized TAO–ALPHA Halvings

- BIT Number: XXXX
- Title: Synchronized TAO–ALPHA Halvings in Dynamic TAO
- Author(s): Maciej Kula
- Discussions-to: TBD
- Status: Draft
- Type: Core
- Created: 2025-09-29
- Updated: 2025-09-29
- Requires: None
- Replaces: None

### Abstract
This BIT proposes synchronizing all subnet ALPHA emissions to the global TAO halving schedule, replacing per-subnet ALPHA halvings that introduce timing-driven cohort bias in Dynamic TAO. Under synchronization, ALPHA-halving intervals stop compressing (getting shorter), liquidity impact depends on sell fraction rather than cohort timing, liquidation haircuts disappear, and the root proportion declines uniformly, resulting in fairer and more predictable behavior across cohorts.

### Motivation
The current design uses global TAO halvings but per-subnet ALPHA halvings, so otherwise identical subnet behavior results in different outcomes based solely on registration timing, which compresses ALPHA-halving intervals, increases AMM liquidity drain for later cohorts, creates liquidation discounts, and distorts the pace of root proportion decline. These asymmetries reduce fairness, predictability, and comparability across cohorts.

### Specification
This BIT modifies the coinbase injection to synchronize ALPHA emissions with the TAO halving schedule. The key change replaces per-subnet ALPHA block emissions with the global TAO block emission.

In the `run_coinbase` function in [run_coinbase.rs](https://github.com/opentensor/subtensor/blob/main/pallets/subtensor/src/coinbase/run_coinbase.rs#L22), the current ALPHA emission is defined as:
```rust
let alpha_emission_i: U96F32 = asfloat!(
    Self::get_block_emission_for_issuance(Self::get_alpha_issuance(*netuid_i).into())
        .unwrap_or(0)
);
```

and should be replaced with:
```rust
let alpha_emission_i: U96F32 = block_emission;
```

This change ensures that ALPHA emissions to participants (`alpha_out_i`) scale with the same global TAO halving that determines the block emission (`block_emission`), rather than each subnet maintaining its own independent ALPHA halving schedule.

To fix the root proportion decline proportionally with the decreasing TAO block emission and thus also the rate of ALPHA, the TAO weight should follow the TAO halving schedule. The current TAO weight is defined as:

```rust
let tao_weight: U96F32 = root_tao.saturating_mul(Self::get_tao_weight());
```

and should be replaced with:

```rust
let tao_weight: U96F32 = root_tao.saturating_mul(
    Self::get_tao_weight().saturating_mul(block_emission)
);
```

Other options would be to change the `TaoWeight` storage with each TAO halving, or modifying the `get_tao_weight` function to be aware of the current TAO block emission.

### Rationale
Synchronization removes the single cause of these asymmetries (the gap between TAO halving and per-subnet ALPHA halving). So that rates become cohort-invariant under one global schedule without per-subnet schedules. Other designs/alternatives such as synchronizing injections to ALPHA halvings or isolating ALPHA emissions for the halving schedule fail to neutralize asymmetries consistently.

### Backwards Compatibility
This change will not be backwards compatible after the first TAO halving in December 2025.

### Security Considerations
N/A

### Copyright
This document is licensed under [The Unlicense](https://unlicense.org/).
