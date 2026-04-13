# BIT-0018: Distributed Subnet Ownership

- **BIT Number:** 0018
- **Title:** Distributed Subnet Ownership
- **Author(s):** Loris Moulin (@l0r1s)
- **Discussions-to:** TBD
- **Status:** Draft
- **Type:** Subtensor
- **Created:** 2025-09-23
- **Updated:** 2025-09-23
- **Requires:** NA
- **Replaces:** NA

## Abstract

The current Bittensor subnet ownership model, which registers ownership to a single on-chain entity, creates significant limitations. It offers no native mechanism to distribute ownership rights or rewards, hindering collaborative governance and investor participation. Even when using a multisig wallet, coordination remains off-chain and inefficient.

This BIT proposes the introduction of **Distributed Subnet Ownership**, a protocol-level solution that allows both economic and governance rights to be fractionalized into on-chain shares. This enables multiple stakeholders to co-own and manage a subnet directly on-chain.

A native implementation provides a standardized, secure, and efficient framework, which is preferable to bespoke smart contract solutions. By integrating this functionality into the protocol, we can strengthen decentralization, improve transparency, and lower coordination costs for subnet operators, contributors, and investors.

## Motivation

As the Bittensor ecosystem matures, subnets are evolving from single-operator projects into complex ventures requiring diverse teams, external investment, and robust governance. The existing single-owner framework is ill-suited for this reality, creating bottlenecks in collaboration, fundraising, and security. This proposal is motivated by the need to provide subnets with a native, on-chain toolkit for multi-stakeholder management. The primary drivers for this evolution are:

1.  **Enhancing Governance and Reducing Operational Overhead:** Collective subnet management currently relies on off-chain coordination for multisig wallets. This process is often inefficient, opaque, and introduces friction. By moving governance on-chain, we enable transparent, auditable, and streamlined decision-making, reducing coordination costs and improving operational agility.

2.  **Unlocking Investment and Fundraising Opportunities:** The single-owner model creates high barriers for subnet financing. There is no straightforward, trust-minimized mechanism for investors to acquire a stake in a subnet's success or participate in its governance. This forces fundraising into complex off-chain legal agreements. Distributed ownership creates a direct, on-chain pathway for investment, broadening participation and enabling new capital formation models.

3.  **Incentivizing Collaboration:** Building a successful subnet requires a team with diverse skills. The current model makes it difficult to reward contributors with a direct share of ownership, misaligning incentives. Fractionalizing ownership allows core teams and key contributors to hold a tangible stake in the project's success, fostering long-term commitment and collaboration.

4.  **Strengthening Decentralization and Security:** Concentrating ownership in a single entity—even a multisig—creates a central point of failure. A compromise of the owner's keys could be catastrophic for the subnet. Distributing ownership across multiple independent stakeholders makes the subnet more resilient and aligns with the core ethos of decentralization.

5.  **Providing a Standardized and Secure Protocol:** While bespoke smart contracts could replicate some of this functionality, they would lead to a fragmented ecosystem of unaudited and potentially insecure solutions. A standardized, protocol-level implementation ensures that all subnets can benefit from a secure, efficient, and interoperable framework maintained by the OpenTensor Foundation.

## Specification

### Core Mechanisms

#### Distributed Subnet Conversion

A subnet can be converted to a distributed ownership model by calling the `convert_to_distributed_subnet` extrinsic as the subnet owner. The on-chain accounts for distributed subnets are derived directly from an incrementing `DistributedSubnetId`. This process allocates ownership rights across multiple stakeholders through two types of shares:

- **Profit Shares**: Economic rights that determine profit distribution
- **Governance Shares**: Voting rights that control subnet governance decisions

Both share types are represented as per-mille (‰) values, where 1000 shares represent 100% ownership.

```rust
pub fn convert_to_distributed_subnet(
    profit_shares: Vec<(T::AccountId /* Hotkey */, u64)>,
    governance_shares: Vec<(T::AccountId /* Coldkey */, u64)>
) -> DispatchResult
```

#### Share Allocation Rules

- The sum of all profit shares must not exceed 1000 (100%)
- The sum of all governance shares must not exceed 1000 (100%)
- If either share type is empty, the subnet owner retains 100% ownership of that type
- Minimum share allocation is configurable via `MinSubnetShare` (default: 1 per-mille (0.1%))
- Maximum number of shareholders is configurable via `MaxSubnetShareholders` (default: 50)

#### Storage Structures

Two new storage maps are introduced to track ownership:

```rust
type DistributedSubnetId = u32;

#[repr(transparent)]
struct SubnetShares(u64);

pub type SubnetProfitShares<T: Config> =
    StorageDoubleMap<_, Twox64Concat, DistributedSubnetId, Identity, T::AccountId /* Hotkey */, SubnetShares, ValueQuery>;

pub type SubnetGovernanceShares<T: Config> =
    StorageDoubleMap<_, Twox64Concat, DistributedSubnetId, Identity, T::AccountId /* Coldkey */, SubnetShares, ValueQuery>;
```

_Key Design Decisions:_

- **Profit shares are associated with hotkeys** - This enables direct alpha distribution to the hotkey associated with the shares, avoiding sell-pressure on the subnet.
- **Governance shares are associated with coldkeys** - This ensures voting and proposal creation are tied to the coldkeys.

#### Share Transferability

Both profit and governance shares are freely transferable between accounts, handling common errors where the transfer is not possible.

```rust
pub fn transfer_profit_shares(
    distributed_subnet_id: DistributedSubnetId,
    shares: SubnetShares,
    to_hotkey: T::AccountId,
) -> DispatchResult

pub fn transfer_governance_shares(
    distributed_subnet_id: DistributedSubnetId,
    shares: SubnetShares,
    to_coldkey: T::AccountId,
) -> DispatchResult
```

#### Events and Notifications

The distributed subnet system emits events for all major operations including proposal creation, voting, execution, share transfers, and profit distributions. These events must be defined accordingly and emitted correctly to provide transparency and enable off-chain monitoring of the system.

### Profit Distribution

The profit distribution system automatically distributes the subnet's owner cut periodically to shareholders based on their profit shares. Alpha tokens are distributed directly to associated hotkeys to avoid sell-pressure on the subnet.

#### Distribution Logic

1. The on-chain managed distributed subnet account receives the owner cut through coinbase logic
2. The coinbase notifies that distribution has been done using the `OnDistribution` hook
3. Owner cut is accumulated if we are not at the distribution interval `DistributedSubnetProfitInterval`
4. Otherwise, we distribute to shareholders proportional to their shares of the owner cut, with alpha distributed directly to associated hotkeys

The `DistributedSubnetProfitInterval` corresponds to an interval of blocks (now mod interval == 0) where the accumulated owner cut is distributed to shareholders.

#### Hook Integration

The subtensor pallet defines a configurable hook `OnDistribution` to handle shareholder distribution logic behind the scenes for distributed subnets, while standard subnets continue to use the existing distribution mechanism.

```rust
pub trait OnDistribution<AccountId, Balance> {
    fn notify(netuid: NetUid, owner_cut: Balance);
}

// The trait is also implemented for tuples of OnDistribution items to allow later use of multiple distribution hooks for diferrent use cases.
impl<A, B, AccountId, Balance> OnDistribution<AccountId, Balance> for (A, B)
where
    A: OnDistribution<AccountId, Balance>,
    B: OnDistribution<AccountId, Balance>,
{
    fn notify(netuid: NetUid, owner_cut: Balance) {
        A::notify(netuid, owner_cut);
        B::notify(netuid, owner_cut);
    }
}
```

#### Configuration

The runtime configures the distribution hook:

```rust
pub struct DistributedSubnetDistribution;

impl<AccountId, Balance> OnDistribution<AccountId, Balance> for DistributedSubnetDistribution {
    fn notify(netuid: NetUid, owner_cut: Balance) {
        // Handle distributed subnet profit distribution logic
        if let Some(distributed_subnet_id) = Self::get_distributed_subnet_id(netuid) {
            Self::distribute_to_shareholders(distributed_subnet_id, owner_cut);
        }
    }
}

impl pallet_subtensor::Config for Runtime {
    ...
    type OnDistribution = DistributedSubnetDistribution;
}
```

### Governance System

The governance system enables distributed subnet shareholders to collectively make decisions through proposals and voting. Shareholders with governance shares can create proposals and vote on them.

#### Proposal Creation

A proposal is a call execution using the subnet owner origin voted among the shareholders.

The call must be authorized and only a subset of calls will be permitted for execution. This will include hyperparameters changes extrinsics and other subnet management functions, but excludes certain sensitive operations.

Only a single proposal can be active at the same time for a distributed subnet. Anyone with at least `NewProposalMinShares` (default 10%) governance shares can create a proposal. Governance shares are locked for the duration of the proposal, and the proposal has an expiry in blocks from now of `ProposalExpiryBlocks` (default to 21600 blocks = 3 days).

```rust
pub enum VotingStrategy {
    ShareProportionMoreThan(Percent),  // e.g., X% of total shares
    MembersProportionMoreThan(Percent), // e.g., X% of voting members
}

pub fn create_proposal(
    distributed_subnet_id: DistributedSubnetId,
    call: Box<Call<T>>,
    voting_strategy: VotingStrategy,
    expiry_blocks: BlockNumber,
) -> DispatchResult
```

#### Proposal Lifecycle

- **Creation**: Requires minimum governance shares (configurable using `NewProposalMinShares`, default 10%)
- **Active**: Only one proposal can be active at a time
- **Locked Shares**: Proposer's governance shares are locked during proposal
- **Expiry**: Proposal expires after specified number of blocks `ProposalExpiryBlocks` (21600 blocks = 3 days)
- **Execution**: Approved proposals and cancelled proposals are executed automatically on new votes reaching the threshold

#### Voting Mechanism

Voting uses locked governance shares with the strategy defined at proposal creation:

```rust
pub fn vote(
    distributed_subnet_id: DistributedSubnetId,
    proposal_id: ProposalId,
    approve: bool,
) -> DispatchResult
```

#### Proposal Execution

When a proposal reaches the required threshold, it is executed automatically and immediately with the permission of the subnet owner. The execution happens as soon as the threshold is reached, without requiring manual intervention.

#### Configuration Parameters

The governance system uses several configurable parameters:

```rust
// Minimum governance shares required to create a proposal
pub const NewProposalMinShares: u64 = 100; // 10% of 1000 shares

// Default proposal expiry in blocks (21600 blocks = 3 days)
pub const ProposalExpiryBlocks: BlockNumber = 21600;

// Minimum voting threshold to prevent abuse
pub const MinVotingThreshold: Percent = Percent::from_percent(50); // 50% minimum
```

**Key Parameters:**

- **NewProposalMinShares**: Minimum shares to create proposal (default 10%)
- **ProposalExpiryBlocks**: How long proposals last (default 21600 blocks = 3 days)
- **MinVotingThreshold**: Minimum voting threshold to prevent abuse (default 50%)
- **No execution delays**: Proposals execute immediately when threshold is reached

**Voting Rules:**

- **Locked shares**: Governance shares are locked when voting, they are unlocked when the proposal is executed or cancelled
- **No cancellation**: Votes cannot be changed once cast
- **Binary voting**: Yes/No votes only
- **Smart cancellation**: If remaining votes cannot reach threshold, proposal is automatically cancelled
- **Forced consensus**: Single active proposal forces collective decision-making

**Example Scenarios:**

- **ShareProportionMoreThan(60%)**: If 45% vote No, only 55% can vote Yes → Proposal automatically cancelled
- **MembersProportionMoreThan(66%)**: If 40% of members vote No, only 60% can vote Yes → Proposal automatically cancelled

#### Storage Structures

Essential storage structures for governance:

```rust
pub type ProposalId = u32;

#[derive(Encode, Decode, Clone, PartialEq, RuntimeDebug, TypeInfo)]
pub struct ProposalInfo<AccountId, BlockNumber, Call> {
    pub proposal_id: ProposalId,
    pub proposer: AccountId,
    pub call: Call,
    pub voting_strategy: VotingStrategy,
    pub expiry_block: BlockNumber,
    pub created_block: BlockNumber,
}

// Active proposal per distributed subnet
pub type ActiveProposals<T> = StorageMap<
    _, Twox64Concat, DistributedSubnetId,
    ProposalInfo<T::AccountId, BlockNumberFor<T>, T::Call>
>;

// Votes cast on active proposals
pub type ProposalVotes<T> = StorageNMap<
    _,
    (Twox64Concat, DistributedSubnetId),
    (Twox64Concat, ProposalId),
    (Twox64Concat, T::AccountId),
    bool
>;

// Note: Proposal history is not stored on-chain to keep storage minimal.
// Events provide sufficient audit trail for off-chain monitoring.
```

## Rationale

### Split Between Profit and Governance Shares

The separation of profit and governance shares provides maximum flexibility for subnet ownership structures. This allows for scenarios where:

- **Investors** can receive economic benefits without governance control
- **Operators** can maintain governance control while sharing profits
- **Contributors** can be rewarded with profit shares without voting rights
- **Advisors** can have governance influence without economic exposure

### Alpha Instead of TAO Distribution

Alpha distribution is chosen over TAO to avoid sell-pressure on the subnet. By distributing alpha directly to hotkeys:

- **No market impact**: Alpha distribution doesn't affect TAO price
- **Direct benefits**: Shareholders receive subnet-specific rewards
- **Aligned incentives**: Rewards are tied to subnet performance
- **Reduced complexity**: No need for TAO conversion mechanisms

### Voting Strategies

The dual voting strategies (ShareProportionMoreThan and MembersProportionMoreThan) accommodate different governance philosophies:

- **Share-based**: Weighted by economic stake (capitalist approach)
- **Member-based**: One person, one vote (democratic approach)
- **Flexibility**: Proposers can choose the most appropriate strategy
- **Prevention of abuse**: Minimum thresholds prevent low-barrier proposals

### Single Active Proposal

Limiting to one active proposal per subnet forces consensus and prevents governance gridlock:

- **Forced focus**: All attention on one proposal at a time
- **Quick resolution**: Proposals must pass or fail quickly
- **Prevents spam**: Can't create competing proposals
- **Efficient governance**: Streamlined decision-making process

### Locked Shares During Voting

Locking governance shares during voting ensures commitment and prevents gaming:

- **Serious voting**: Forces careful consideration before voting
- **Prevents manipulation**: Can't vote strategically then change
- **Collective pressure**: Group must reach consensus or fail
- **Clean process**: No vote buying or double-dealing

### No Proposal History Storage

Proposal history is not stored on-chain to maintain efficiency:

- **Storage optimization**: Reduces on-chain storage costs
- **Event-based audit**: Events provide sufficient tracking
- **Off-chain monitoring**: External systems can maintain history
- **Focus on active governance**: System optimized for current operations

## Backwards Compatibility

No backwards incompatibilities, the system is designed to be additive and opt-in. Existing subnets continue working exactly as before, and only subnets that explicitly convert to distributed ownership get the new features.

## Security Considerations

### Call Authorization and Restrictions

The governance system must implement strict call authorization to prevent malicious proposals:

- **Whitelist approach**: Only pre-approved call types can be proposed
- **Sensitive operations excluded**: Subnet deletion, owner transfer, and other critical operations are forbidden
- **Hyperparameter limits**: Proposals can change hyperparameters within safe bounds
- **Execution context**: All calls execute with subnet owner authority, not root authority

### Share Manipulation Prevention

Several mechanisms prevent share-based attacks:

- **Minimum thresholds**: 50% minimum voting threshold prevents low-barrier proposals
- **Locked shares**: Governance shares are locked during voting, preventing double-voting
- **Single active proposal**: Prevents proposal spam and governance gridlock
- **Smart cancellation**: Automatic cancellation when thresholds become impossible

### Proposal Execution Security

Proposal execution is designed to be secure and predictable:

- **Automatic execution**: No manual intervention required, reducing human error
- **Immediate execution**: No delays that could be exploited
- **Subnet owner authority**: Executes with appropriate permissions, not root
- **Failed execution handling**: System must handle execution failures gracefully

### Economic Security

The profit distribution system includes several security measures:

- **Alpha distribution**: Avoids TAO market manipulation
- **Direct distribution**: No intermediate accounts that could be compromised
- **Proportional shares**: Fair distribution based on ownership
- **Accumulation mechanism**: Prevents manipulation of distribution timing

### Governance Attack Vectors

The system is designed to resist common governance attacks:

- **Proposal spam**: Single active proposal prevents competing proposals
- **Vote buying**: Locked shares prevent vote trading
- **Sybil attacks**: Share-based voting requires economic stake
- **Governance capture**: Minimum thresholds and voting strategies provide protection
- **Time-based attacks**: Expiry blocks prevent indefinite proposals

### Key Management

Shareholder key management is critical for system security:

- **Coldkey security**: Governance shares are tied to coldkeys for security
- **Hotkey distribution**: Profit shares use hotkeys for direct alpha distribution
- **Key rotation**: Shareholders must manage their own key security
- **Lost keys**: No recovery mechanism for lost shareholder keys

### Network Security

The distributed subnet system maintains network security:

- **Opt-in conversion**: Existing subnets are not automatically affected
- **Backwards compatibility**: No breaking changes to existing systems
- **Isolated governance**: Each distributed subnet has independent governance
- **Event transparency**: All operations are transparent through events

## Scenarios

### Economic Scenarios

#### Successful Profit Distribution

**Scenario**: A distributed subnet with 3 shareholders (Alice: 40%, Bob: 35%, Charlie: 25%) receives 1000 alpha in owner cut.

**Process**:

1. Owner cut accumulates until `DistributedSubnetProfitInterval` (e.g., every 1000 blocks)
2. System calculates: Alice gets 400 alpha, Bob gets 350 alpha, Charlie gets 250 alpha
3. Alpha distributed directly to their hotkeys
4. All shareholders receive their proportional share automatically

**Result**: Fair, automatic distribution based on ownership percentages.

#### Share Transfer Success

**Scenario**: Alice wants to sell 10% of her shares to David.

**Process**:

1. Alice calls `transfer_profit_shares(distributed_subnet_id, 100, david_hotkey)`
2. System validates Alice owns at least 100 shares
3. Alice's shares reduced from 400 to 300, David receives 100 shares
4. Future profit distributions: Alice gets 30%, Bob 35%, Charlie 25%, David 10%

**Result**: Clean ownership transfer with immediate effect on future distributions.

### Governance Scenarios

#### Successful Proposal Execution

**Scenario**: Bob proposes a change using `ShareProportionMoreThan(60%)` strategy.

**Process**:

1. Bob creates proposal (requires 10% minimum shares, Bob has 35%)
2. Alice votes Yes (40% shares), Bob has created the proposal (35% shares)
3. Total: 75% Yes votes, exceeds 60% threshold
4. Proposal executes automatically with subnet owner authority
5. Hyperparameter changed

**Result**: Democratic decision successfully implemented.

#### Proposal Failure - Insufficient Support

**Scenario**: Charlie proposes a change using `ShareProportionMoreThan(60%)` strategy.

**Process**:

1. Charlie creates proposal (has 25% shares, above 10% minimum)
2. Alice votes No (40% shares), Bob votes No (35% shares)
3. Total: 75% No votes, only 25% can vote Yes
4. System automatically cancels proposal (25% < 60% threshold)
5. Charlie's locked shares are unlocked

**Result**: Proposal fails quickly, preventing wasted time on impossible proposals.

#### Proposal Failure - Automatic Cancellation

**Scenario**: Alice proposes a controversial change with `ShareProportionMoreThan(70%)` strategy.

**Process**:

1. Alice creates proposal (40% shares, above 10% minimum)
2. Bob votes No (35% shares)
3. System automatically cancels proposal because we will never reach above 70%
4. All locked shares are unlocked immediately

**Result**: Proposal cancelled quickly, preventing wasted time on impossible proposals.

#### Proposal Failure - Expiry

**Scenario**: Alice proposes a change with `ShareProportionMoreThan(60%)` strategy, but not enough shareholders vote.

**Process**:

1. Alice creates proposal (40% shares, above 10% minimum)
2. Only Alice votes Yes (40% shares), others don't vote
3. Total: 40% Yes votes, below 60% threshold
4. Proposal expires after 21600 blocks (3 days) without reaching threshold
5. All locked shares are unlocked

**Result**: Proposal expires naturally due to insufficient participation, governance continues with new proposals.

## Future Improvements

The following features are considered for future development but are not included in the initial implementation:

## Share Lending

- **Time-based lending**: Lend shares to someone else for N blocks, with automatic return to the original lender
- **Flexible partnerships**: Enable temporary collaborations and testing periods
- **Risk management**: Share owners retain ultimate ownership while allowing others to benefit

## Governance Delegation

- **Delegation of governance shares**: Delegate voting power while retaining ownership
- **Expertise delegation**: Allow domain experts to vote on behalf of shareholders
- **Flexible governance**: Professional governance management without ownership transfer

## Advanced Governance Features

- **Multiple proposals**: Allow multiple active proposals per subnet
- **Partial voting**: Vote with only a portion of governance shares on specific proposals
- **Enhanced flexibility**: More sophisticated governance mechanisms

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
