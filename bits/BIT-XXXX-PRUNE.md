# BIT-XXXX: veAlpha Governance Layer for Bittensor

- **BIT Number:** XXXX
- **Title:** veAlpha Governance Layer — Vote-Escrowed Subnet Governance with Liquidation Auctions
- **Author(s):** Victor Teixeira (0xVonBismarck on Discord & Twitter)
- **Discussions-to:** [https://github.com/opentensor/bits/discussions](https://github.com/opentensor/bits/discussions)
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 2025-03-12
- **Updated:** 2026-04-10
- **Replaces:** N/A

## Abstract

Bittensor currently lacks a dedicated governance layer for subnet oversight. This absence has produced a structural misalignment between subnet owners, stakers, and miners — one that recent events have made difficult to ignore. The abrupt exit of Covenant AI, whose owners mass-liquidated their token holdings, and the governance failures observed within Subnet 29 illustrate the risks of unchecked subnet ownership. Without formal mechanisms for community intervention, orderly transitions, or structured exits, subnet owners can act against the interests of their stakeholders with no recourse available to the network.

This BIT introduces the **veAlpha governance layer** to address this gap. By requiring participants to lock Alpha tokens for up to four years in exchange for time-weighted voting power, veAlpha ties governance influence directly to long-term economic commitment. The community gains the ability to democratically challenge misaligned subnets or initiate orderly ownership transitions — all through on-chain, auditable processes. When a challenge succeeds, the subnet enters a Dutch auction for the subnet slot where prospective owners can bid TAO to take over operations, preserving the slot's value rather than destroying it. Subnet owners, in turn, must maintain locked positions to defend their slots against these challenges, creating a credible cost of incumbency. The result is a cooperative equilibrium in which the incentives of owners, stakers, and the network are durably aligned.

The veAlpha governance layer does not alter the existing emission or deregistration pricing models. Instead, it introduces an additional layer beyond the current price-dependent deregistration mechanism — a market-driven, time-weighted incentive alignment system that enables the network to democratically prune subnets without requiring price decay.

## Motivation

### 1. Bittensor Has No Governance Layer

Bittensor currently lacks a formal governance mechanism for subnet oversight. Subnet owners receive 18% of emissions, control hyperparameters, and occupy scarce network slots — but face no structured accountability to the community that stakes into their subnets. There are no checks and balances. The network has no on-chain process for the community to challenge a misaligned owner, force a transition, or even influence subnet configuration. This absence of governance is not a missing feature — it is an active risk, as demonstrated by recent incidents where subnet owners acted against the interests of their stakers with no recourse available.

### 2. Price-Only Deregistration Creates Incumbency Bias

The sole mechanism for removing a subnet today is tied to price: a subnet is deregistered only when its EMA price falls to the lowest among non-immune subnets and a new registration displaces it. This creates a strong incumbency bias. Established subnets with accumulated liquidity can hoard their slots indefinitely — even when they are underperforming, inactive, or misaligned — simply because their price has not fallen far enough relative to other subnets. The network cannot prune itself; it can only wait for the market to do it, which may never happen for entrenched incumbents.

Existing voluntary exit proposals, such as the market-based liquidation model proposed by Tao Lover in December 2025 (which ties exit values to spot price p_i = τ_i / α_i and applies a 41/41/18 remainder split), address the economics of owner exits but still result in the destruction of the subnet slot. They do not solve the governance problem — they offer no path for the community to initiate a transition or for a new team to take over a functioning subnet.

### 3. Unlocked Emissions Invite Corruption

Subnet owners receive 18% of all emissions from their subnet with no lock, no vesting, and no governance obligation. Aside from a one-time registration cost, there is no ongoing requirement for owners to demonstrate long-term commitment: they can collect emissions and immediately sell them, extracting value from their subnet at virtually no cost. This low barrier, paired with the absence of capital lockup or governance accountability, makes it disproportionately easy to prioritize short-term extraction over responsible stewardship. Without a mechanism that ties ownership to locked capital and governance participation, the incentive structure invites corruption and undermines sustainable subnet management.

### The Gap This Proposal Fills

Given that the network has no way to prune itself other than waiting for a subnet's price to drop below the deregistration threshold, this proposal introduces two complementary mechanisms:

1. **A baseline governance layer (veAlpha)** that gives committed stakeholders — both Alpha holders and root stakers — democratic voting power over subnet lifecycle, grounded in long-term token locks rather than transient balances.
2. **A liquidation auction mechanism** that allows subnets to be democratically deregistered through a bonded challenge followed by a Dutch auction, preserving slot value through orderly transfer rather than destruction.

## Specification

### 1. veAlpha — Vote-Escrowed Alpha

veAlpha is a non-transferable voting power derived from locking a subnet's Alpha token, modeled on Curve Finance's vote-escrowed CRV (veCRV) system. The only way to obtain veAlpha is by locking Alpha. The maximum lock time is four years.

**Voting power.** A user's veAlpha balance decays linearly as the remaining time until their Alpha unlock decreases. One Alpha locked for four years provides an initial balance of one veAlpha. Formally, user voting power w_i(t) is a linearly decreasing function from the moment of lock:

> **w_i(t) = α_locked × (t_unlock − t) / T_max**

where T_max = 4 years. Because voting power is proportional to both the amount locked and the time remaining, different combinations of amount and duration can yield the same veAlpha balance. For example, 4,000 Alpha locked for one year provides the same veAlpha as 2,000 Alpha locked for two years, or 1,000 Alpha locked for four years.

**Per-subnet scope.** veAlpha is scoped to the subnet whose Alpha was locked. Locking Alpha_i grants voting power on subnet _i_ only. To participate in governance on multiple subnets, a user must hold and lock each subnet's Alpha independently.

**Lock management.** Users can offset decay in two ways:

- **Increase amount:** deposit additional Alpha into an existing lock, immediately increasing veAlpha balance.
- **Increase unlock time:** extend the lock further into the future (up to T_max from the current block), restoring decayed voting power.

Lock duration cannot be decreased. A new lock cannot be created when an existing lock already exists on the same subnet. Users may only withdraw their Alpha after the lock has fully expired.

**Alpha yield.** Locking Alpha does not forfeit staking yield. Alpha emissions continue to accrue to the holder's wallet as unlocked tokens, regardless of whether the underlying position is locked. Yield is not auto-locked — it arrives as freely transferable Alpha that the holder can use at their discretion: lock it into their existing veAlpha position to increase voting power, sell it, restake it, or use it for any other purpose. This ensures that locking Alpha for governance does not impose an opportunity cost on yield, and that subnet owners can continue to use their emission yield to fund ongoing operations while maintaining a locked governance position.

### 2. Root Staker Participation

Root (TAO) stakers — participants who stake TAO — can allocate voting weight to subnet governance via a gauge-style mechanism:

- **Weight calculation:** a root staker's effective veAlpha-equivalent weight on a subnet is **18% of what an equivalent Alpha locker would receive**. This preserves Alpha holders as the primary decision-makers (82% weight) while giving the broader network a meaningful voice.
- **Gauge-style allocation:** root stakers distribute their total voting budget across subnets of their choosing. Allocation is **zero-sum**: increasing weight on one subnet proportionally reduces weight available for others.
- **Reallocation:** root stakers can adjust their allocation at any time. Unlike Alpha lockers, root stakers do not face lock decay — their weight is a function of their TAO stake and chosen allocation ratios.

This design ensures that subnet governance is not a purely internal affair. A subnet that is widely perceived as harmful to the network can be challenged with root staker support even if the subnet owner holds a dominant veAlpha position.

### 3. Liquidation Challenges

A subnet enters the liquidation pipeline through a bonded governance challenge:

**Proposal submission.** Any account holding veAlpha on the target subnet may submit a liquidation challenge. The proposer must simultaneously post a **bond** — additional Alpha locked into escrow, separate from their veAlpha lock. The bond amount is set by governance (e.g. 10% of the subnet's outstanding Alpha) and serves as a cost of initiation that deters frivolous challenges.

**Voting period.** Once submitted, a 7-day voting window opens (duration configurable by governance). During this period, all veAlpha holders on the subnet plus root stakers with allocated weight cast **for** or **against** votes, each weighted by their veAlpha or root-staker-equivalent balance at the snapshot block. All veAlpha weight that abstains from the vote defaults to a vote against the challenge.

**Resolution.**

- **Challenge passes** (simple majority of participating weighted votes): the proposer's bond is returned in full and the subnet enters the subnet auction (Section 5).
- **Challenge fails** (majority votes against): the proposer's bond is **slashed** — 50% distributed pro-rata to the subnet's Alpha holders, 50% recycled to the protocol. This makes failed challenges materially costly.

**Cooldown.** After a failed challenge, no new liquidation challenge can target the same subnet for a configurable cooldown period (e.g. 30 days), preventing harassment through repeated challenges.

### 4. Subnet Owner Defense

The subnet owner occupies a privileged default position in liquidation governance:

**Automatic defense.** The owner's veAlpha balance on their own subnet **automatically counts as "against" votes** on any liquidation challenge. The owner does not need to actively monitor or respond to every challenge — their defense is passive and proportional to their locked Alpha.

**Voluntary exit.** If the owner wishes to exit, they can explicitly **abstain** from defense or **vote for** a liquidation challenge targeting their own subnet. The owner may also submit a challenge against themselves.

**Scaling incentive.** Larger subnets have more Alpha in circulation, meaning more veAlpha can potentially be mobilized by challengers. To maintain a credible defense, subnet owners and their aligned stakeholders must lock proportionally more Alpha. This creates a **cost of incumbency** that scales with subnet value — running a valuable subnet requires ongoing governance commitment, not just emissions farming.

### 5. Subnet Auction

Once a liquidation challenge passes, the subnet's deregistration is paused and a Dutch auction commences for the underlying slot:

- **Duration:** 5 days maximum from the initiation block (36,000 blocks at 5 × 7,200 blocks per epoch).
- **Starting price multiplier (M):** the price decay begins at M × τ_i, where τ_i is the subnet's pool TAO reserves and M is a protocol-configurable parameter (default M = 10, adjustable by governance).
- **Price decay formula:** the price at block _b_ (blocks elapsed since auction start) is:

> **P(b) = τ_i × M^((B − b) / B)**

  - B = total auction duration in blocks
  - At b = 0: P(0) = τ_i × M (starting price)
  - At b = B: P(B) = τ_i (reserve/final price)

- **Bidding:** the first participant to accept the current price — evaluated per block — claims the subnet slot. The auction is anchored to τ_i to ensure the minimum liquidation value is never breached: the pool is never sold below its TAO value, and Alpha holders receive at least their fallback entitlement.

### 6. Auction Resolution

**If a buyer is found within 5 days:**

1. The winning bidder pays the purchase price in TAO.
2. The network initiates a cold key swap, transferring subnet slot ownership to the buyer's designated wallet after the standard 5-day security delay.
3. The purchase price is distributed across Alpha holders based on their election:
   - **Opt-in (retain Alpha):** Alpha is retained under new management; the holder's TAO allocation is deposited into the pool, supporting incoming team. Alpha on the owner key automatically opts in at auction start.
   - **Opt-out (redeem):** Alpha is recycled; the holder receives TAO = a_j × (purchase_price / α⁰_i).

**If no buyer is found within 5 days:**

The subnet proceeds to fallback liquidation via the market-based distribution model. The Alpha Distribution Ratio (ADR) determines the path:

- **ADR > 1 (α⁰_i > α_i):** pro-rata only. Each holder receives a_j × (τ_i / α⁰_i). No remainder.
- **ADR ≤ 1 (α⁰_i ≤ α_i):** each holder redeems at spot price p_i. A remainder T_remainder = τ_i − V is generated and split:
  - 41% to Alpha holders (pro-rata)
  - 41% to Root stakers
  - 18% recycled to protocol

### 7. Incentive Dynamics

The veAlpha governance layer's game theory creates a self-reinforcing equilibrium that selects for long-term aligned participants on both sides of governance:

**Cost of incumbency.** Subnet owners must continuously maintain veAlpha locks. As a subnet grows in value, more Alpha enters circulation, and the owner needs proportionally larger locks to outweigh potential challengers. This cost scales naturally — a subnet worth defending is a subnet worth locking for.

**Cost of attack.** Challengers face compounding friction: (a) acquiring Alpha on the open market pushes the price up, (b) locking it for up to four years renders it illiquid, and (c) the bond requirement means failed challenges are materially costly. These constraints make hostile takeovers expensive and self-limiting.

**Symmetric alignment.** Both defenders and attackers are forced into long-term commitment. A successful takeover means the new controller holds significant locked Alpha that cannot be dumped post-acquisition. This ensures that governance transitions result in genuinely committed new leadership.

**Root staker tiebreaker.** The 18% network-level voice prevents subnet governance from becoming a purely internal affair. A subnet widely perceived as harmful to the network can be challenged with root staker support, even if the subnet owner holds a dominant veAlpha position within the subnet.

**Natural equilibrium.** Well-run subnets attract loyal Alpha lockers who passively reinforce the owner's defense. Poorly-run subnets see Alpha holders decline to lock — or sell their Alpha entirely — leaving the owner exposed to challenge. The mechanism is self-correcting without requiring active intervention from any central authority.

**Integrated Staking Utility:** Subnets can leverage veAlpha as a dynamic rewards mechanism, directly linking governance participation to tangible benefits aligned with the subnet's primary services. For example, subnets may offer enhanced API access, reduced transaction fees, premium features, or other valuable incentives to users who lock and stake Alpha. This not only drives greater community engagement but also fosters positive network effects and sustains long-term alignment between subnet growth and active participant contribution.

### 8. Flow Summary

```
Liquidation Challenge Flow:

  Subnet Operating Normally
        │
        ▼
  Liquidation Challenge (bonded)
        │
        ▼
  7-Day veAlpha Vote
       / \
      /   \
  Passes   Fails
    │        │
    ▼        ▼
  Bond     Bond Slashed
  Returned   (50% holders / 50% recycled)
    │        │
    ▼        ▼
  5-Day    30-Day Cooldown
  Subnet     │
  Auction    ▼
   / \     Back to Normal
  /   \
Buyer  No Buyer
  │      │
  ▼      ▼
Cold   Fallback Liquidation
Key    (41/41/18 Split)
Swap     │
  │      ▼
  ▼    Slot Released
New
Owner
```

## Rationale

### Benefits

- **Price discovery and value preservation.** The subnet auction ensures slots are transferred at or above their fallback TAO value (τ_i), safeguarding Alpha holders from fire-sale prices.
- **Orderly exit.** Alpha holders choose to opt in or out of transitions, allowing a personalized mix of continued exposure or safe exit with TAO payout.
- **Resistance to opportunistic takeover.** The bonded-challenge model, veAlpha lock requirements, and bond slashing make hostile challenges expensive and self-limiting.
- **Transparency and accountability.** All governance actions — challenges, votes, bond slashing, auctions — are on-chain and auditable.

### Why veAlpha

veAlpha governance is superior to simple validator-only voting for subnet lifecycle management:

- **Sybil resistance.** Voting power requires locking real capital for extended periods, not just running validator infrastructure.
- **Proportional representation.** Anyone holding a subnet's Alpha can participate in governance, not just validators — broadening the governance base while weighting for conviction.
- **Permissionless challenge and defense.** Any committed stakeholder can initiate or defend against a liquidation challenge. Governance is not gated by validator status.

### Alternative Approaches Considered

During the development of this proposal, several alternative approaches for subnet deregistration and governance were evaluated and ultimately found lacking for Bittensor’s evolving needs:

- **Shorts-Based Liquidation:** Previous discussions explored using synthetic shorts-based liquidations, where unbacked Alpha would be minted and sold into the pool to force a shutdown or payout. While mechanically appealing, this introduces price instability and breaks confidence in the protocol. Unbacked synthetic minting dilutes honest holders, disconnects price signals from real market conviction, and fails to guarantee a predictable (or fair) liquidation floor. These dynamics risk severe losses for passive Alpha holders, destabilize subnet economics, and complicate network-wide emission policy.

- **Validator-Only Voting:** Traditional validator-based governance, where network validators initiate or approve deregistration by on-chain vote, centralizes power in the hands of infrastructure operators. While this method ensures informed technical oversight, it leaves the majority of the community (especially Alpha holders and new stakeholders) without meaningful agency in subnet lifecycle decisions. Opaque validator strategies may also conceal misaligned incentives, further reducing protocol transparency and adaptability.

- **Snapshot-Style Voting:** Standard snapshot token voting—where voting rights are determined simply by live token balances at a snapshot—empowers large “whale” holders, rewarding them for transient or mercenary positions. This model can amplify short-term swings in governance and facilitate plutocratic control by large or opportunistic actors. By contrast, P.R.U.N.E’s veAlpha model ensures that voting power is aligned with protocol-wide health: only those who lock Alpha for a committed duration accrue proportional voting weight, forcing longer horizons and discouraging parachute-style manipulation.

- **Triumvirate Governance:** The current model, which disperses governance across a triumvirate structure (validators, subnet owners, protocol maintainers), was practical during early network growth but is not sustainable as Bittensor matures. It fragments authority, adds operational friction, and leaves unclear escalation or recourse mechanisms for contentious governance events. As the network scales, democratic, transparent, and stake-weighted community-driven mechanisms are needed to foster healthy subnet turnover and robust resistance to capture.

Each of these alternative approaches was found to be fragile, exclusionary, or unsustainable for long-term governance. By embracing on-chain auctions and veAlpha-based decision power, the veAlpha governance layer aligns incentives for all network participants and sets the foundation for network longevity through market-driven, community-oriented processes.

## Backwards Compatibility

The veAlpha governance layer introduces new on-chain state (veAlpha lock records, governance proposals, bond escrow, auction tracking) and new dispatchable functions. Existing subnets continue to operate unchanged until they interact with the liquidation pipeline. The fallback liquidation logic is consistent with the established market-based distribution path. No migration of existing subnet data is required.

### Implementation Paths

The veAlpha governance layer can be realized through either of two paths:

1. **Native Subtensor pallet (runtime upgrade).** The veAlpha locking, governance proposal, and auction logic are implemented directly as a Subtensor pallet in Rust, integrated via a standard runtime upgrade. This approach offers the tightest integration with existing Subtensor primitives (cold key swaps, emission hooks, hyperparameter storage) and avoids cross-layer bridging complexity. However, it requires coordination with the Opentensor Foundation for inclusion in a runtime release and is subject to the existing governance approval process for chain upgrades.

2. **SubtensorEVM smart contracts.** Bittensor's EVM runtime operates as an application layer on top of Subtensor, allowing deployment of Solidity smart contracts that execute directly on the Bittensor blockchain. The veAlpha vote-escrow contract (modeled on Curve's `VotingEscrow.vy`) can be deployed as an EVM contract, with governance proposal logic, bond escrow, and auction mechanics implemented in Solidity. Existing precompiles — including the Subnet precompile (`0x803`) for hyperparameter management and the Staking precompile (`0x805`) for TAO operations — provide the necessary bridge between EVM contract state and native Subtensor state. This path enables faster iteration and community-driven deployment without requiring a runtime upgrade, though it introduces EVM-to-Subtensor bridging considerations for operations like cold key swaps and emission hooks that are native to the Subtensor runtime.

Either path is technically viable. The SubtensorEVM approach may be preferable for an initial deployment, allowing the community to validate the governance mechanics in production before pursuing native pallet integration for tighter runtime coupling.

## Reference Implementation

N/A — To be developed upon BIT acceptance.

## Security Considerations

- **Bond griefing:** wealthy actors could spam liquidation challenges to drain bond capital from repeated slashing. Mitigated by per-subnet cooldown periods after failed challenges and the material cost of slashing itself.
- **Whale dominance:** a single actor locking massive Alpha could dominate a subnet's governance. Mitigated by the root staker counterweight (18% weight), the natural cost of acquiring large Alpha positions on the open market, and the 4-year lock commitment required for maximum voting power.
- **Lock manipulation:** actors might time locks to maximize vote weight during proposal windows, then withdraw. Mitigated by linear decay — short locks yield minimal veAlpha, and withdrawal is only possible after full expiry.
- **Gauge-weight manipulation:** root stakers could rapidly shift weight to swing active votes. Mitigated by the 18% weight cap relative to Alpha lockers and the 7-day voting period which limits last-minute reallocation impact.
- **Auction gaming:** bidders might attempt to delay or manipulate the subnet auction. The 5-day hard cap and deterministic per-block price decay limit such strategies.
- **Opt-in/opt-out timing:** Alpha holders must have a clear, bounded window to elect during auctions. Default behavior is opt-out if no action is taken, protecting passive holders.
- **Cold key swap security:** the key swap mechanism follows existing Subtensor cold key swap security guarantees to prevent theft or unauthorized transfer.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
