# BIT-0002: Proof of Weights

- **BIT Number:** 0002
- **Title:** Proof of Weights
- **Author(s):** Inference Labs (https://github.com/inference-labs-inc/)
- **Discussions-to:** [`#proof-of-weights`](https://discord.gg/4aHEVxVZMz)
- **Status:** Draft
- **Type:** Core, Subtensor
- **Created:** 2024-11-07
- **Updated:** 2025-06-13
- **Requires:** BIT-0008
- **Replaces:** None

## Abstract

This proposal introduces Proof of Weights, a real-time cryptographic solution for Bittensor's ecosystem. It ensures subnet owners' incentivization mechanisms are faithfully executed by validators, safeguarding intended behavior and reward distribution. By leveraging scalable, configurable zero-knowledge proofs, Proof of Weights maintains network bandwidth while rewarding honest validators for their genuine contributions, providing subnet operators with enhanced tools to direct validator behavior. Additionally, Proof of Weights benefits miners by ensuring that feedback on their performance remains immediate and uninterrupted. This allows miners to maintain the current level of insight into their real-time performance, facilitating faster adjustments and sustaining high operational and network efficiency.

## Motivation

In the Bittensor documentation, subnet owners are burdened with two key responsibilities:

- Defining the specific digital task to be performed by the subnet miners.
- Implementing an incentive mechanism that aligns miners with the desired task outcomes.

The existing protocol, however, does not offer subnet owners any method to verify that their specific digital task is being completed by the network, or that their incentive mechanism is being used by validators to score miner’s work. This inability to verify the behavior of subnet validators has lead to the existence of weight copying, whereby validators can avoid doing any of the work defined by the subnet owner, and gain higher profit by copying the weights distributed to miners by other validators who are (presumably) running the correct code. The lack of verification present in the protocol today could lead to situations where a subnet effectively goes rogue; validators could choose to simply not run the code provided by the subnet owner, modify the owner’s code, or run their own code to hijack the subnet entirely. This is a serious issue for builders entering the Bittensor ecosystem, as no certainty is guaranteed in the code they deploy for use in their subnet. This presents an additional economic risk as 5,904 TAO are distributed to validators and miners daily but no formal verification is present to ensure that this distribution is being applied as intended.

Proof of Weights is a necessary addition to the Bittensor network because it allows subnet owners to verify usage of their defined incentive mechanisms by validators participating in their subnet. Specifically, this addition to the protocol will allow subnet owners to define a penalty for validators that cannot cryptographically prove unique and genuine weight calculations conducted during runtime against the specific functionality of the subnet. This is a benefit to the Bittensor ecosystem as it provides subnet owners with the ability to verify their responsibilities are being carried out within the subnet they operate in, in a way that is trustless and transparent.

## Specification

**Proof of Weights** exclusively covers subnet **incentive mechanism**s.

Where **Proof of Weights** is defined as follows.

> A cryptographic verification system in Bittensor that requires validators to provide zero-knowledge proofs demonstrating they correctly executed a subnet's incentive mechanism when assigning weights to miners.

Where **incentive mechanism** is defined as follows.

> A mechanism which converts input vectors which represent qualitative signals about a miner’s work into succinct weight vectors which represent the outcoming score of that miner on a normal distribution.

### Zero Knowledge Circuits

Each subnet owner that intends to use Proof of Weights must supply a custom zero knowledge circuit representing their incentive mechanism. Technically, asking each subnet owner to implement their own zero knowledge circuit from scratch would be prohibitive in terms of development effort. To ease this barrier to entry, conversion of Python based incentive designs to Halo2 proof circuits is adopted through the use of EZKL.

High level, the process for conversion of the incentive mechanism into a zero knowledge circuit involves the following steps.

1. Implementation of the incentive mechanism as a torch neural network module
2. Exportation of the incentive mechanism to ONNX format
3. Conversion of the ONNX representation to the Halo2 circuit

After this process, the subnet owner will provide the exported Halo2 verification key to the chain as discussed in the hyperparameters section.

For reference, examples of the above process are provided in the [Proof of Weights simulation repository].

### Hyperparameters

Two new hyperparameters are required, each can be configured by the subnet owner.

- `weights_verification_key`

  This hyperparameter stores the cryptographic verification key unique to the incentive mechanism created by the owner of the subnet. Blockchain validators will use the value of this hyperparameter during proof verification, to attest validator supplied proofs.

- `valid_proof_value`

  This hyperparameter is a float value between 0 and 1. It represents the value that the subnet owner places on validators supplying cryptographic proofs alongside their weights within the subnet. When set to zero, validators that do not provide proofs alongside their weights are valued the same as validators that do. When set to 1, validators that provide proofs are maximally incentivized.

### Additional Weight Set Method

One new function is required, to allow validators running on proof of weights subnets to commit proofs and input signals along with the weights they set. This function has a signature as follows, and must include functionality to verify incoming proofs and provided signals against the verification_key hyperparameter, using the metadata parameters provided within the function call.

```rust
set_weights_with_proof(self,
    wallet: "bittensor.wallet",
    netuid: int,
    uids: Union[NDArray[np.int64], "torch.LongTensor", list],
    weights: Union[NDArray[np.float32], "torch.FloatTensor", list],
    proof_cid: str,
    wait_for_inclusion: bool = False,
    wait_for_finalization: bool = False,
    prompt: bool = False,
    max_retries: int = 5,
    parameters: dict = None
)
```

### Yuma Consensus

Changes to Yuma Consensus must be made to implement a penalty for validators that cannot provide a proof with their weights. Let Vp represent an array of proof validity signals for each validator such that Vp[i] ∈ {1, 0}. A value of 1 indicates the validator `i` provided a valid proof with their weights during the previous tempo. To implement the valid_proof_value hyperparameter, the following modification to clipped weights within Yuma consensus is proposed. The core functionality of this change involves boosting the weights of validators who supply valid proofs with their weights. When set to 1, the subnet owner places maximum value in validators who provide proof. This means that validators who supply proof with their weights will have their weights clipped to consensus without further changes, and validators who are unable to supply proofs along with their weights will be clipped to zero.

```rust
W_clipped *= (Vp + (1 - Vp) * (1 - hyperparameter(valid_proof_value)))
```

![Flowchart of Proof of Weights](https://github.com/user-attachments/assets/82438fa9-8c93-45cc-99fa-ad4b7e7678bd)

<details open>
<summary>Simulation of Proof of Weights in Action</summary>

![PoW Simulation](https://github.com/user-attachments/assets/e6ef0e53-9d96-43ce-9cdf-df77f82be0ad)

</details>

## Rationale

In response to the practise of weight copying, Bittensor has released Commit Reveal (and subsequently Commit Reveal 2.0) to combat this behavior. Commit reveal is designed to reduce emissions of copying validators in specific conditions. If enabled, Commit Reveal addresses this problem by enforcing all subnet validators to commit a hash of their responses on chain. Validators would then reveal these responses after the expiry of an interval that has been set as a parameter by the subnet owner, effectively delaying the feedback loop for miners and propective weight copiers alike.

While Commit Reveal effectively addresses weight copying, its time interval introduces significant latency that delays critical feedback to miners. This latency conflicts with Bittensor's core goal of rapidly incentivizing state-of-the-art development. The delayed feedback loop not only slows network innovation but also frustrates miners by reducing real-time performance visibility. Additionally, this delay enables a new attack vector where dishonest validators can attempt to predict weights before they are revealed, potentially circumventing the intended protections.

With Proof of Weights, validators are required to prove that they've done the correct scoring work to evaluate miners within the subnet based on the subnet owner's defined incentive mechanism. This opens the door for new strategies which can be used in conjunction with commit reveal to further mitigate the issue of weight copying.

### Alternatives Considered

#### Circom

The [Circom] proof system using [groth16] proofs was considered as an alternative to [EZKL]’s Halo2 implementation for several reasons listed below, however the barrier to developer adoption is significant considering it uses a DSL ([circom]) to define constraints. Migration to this proof system is recommended in the future as tooling surrounding the proof system improves, lowering the barrier to entry for developers.

- Proof sizes remain a fixed size of approximately 616 bytes
- Proving times have shown significant improvements within the [Omron] subnet, with 1024 scoring changes proven in under 1.5s for Subnet 2.

Benchmarks taken from an Apple M1 machine with 10 CPUs and 32GB of RAM, comparing the performance of [EZKL] and [Circom] for several Proof of Weights circuits are show below.

| Proof System | Circuit   | Batch Size | Proof Size | Public Signals Size | Verification Time (per proof) |
| ------------ | --------- | ---------- | ---------- | ------------------- | ----------------------------- |
| EZKL         | Subnet 2  | 1024       | 21.57kb    | 68kb                | 0.7s                          |
| EZKL         | Subnet 27 | 1024       | 10.56kb    | 137kb               | 0.1s                          |
| EZKL         | Subnet 48 | 1024       | 8.32kb     | 68kb                | 0.6s                          |
| Circom       | Subnet 2  | 1024       | ~616 bytes | 166kb               | 0.38s                         |
| Circom       | Subnet 48 | 1024       | ~616 bytes | 65kb                | 0.31s                         |

#### Jolt

Jolt is a novel proof system developed by a16z, which provides a fully featured zkVM. This virtual machine allows users to define their programs in the widely adopted rust programming language. Ultimately, despite the ease of development, the proof size and verification times are not competitive with [EZKL] and [Circom]. For these reasons, it does not currently offer advantages over existing proof systems and thus was rejected for use in Proof of Weights. The Jolt proof system continues to develop rapidly, and future progress will be reevaluated on an ongoing bases for use with Proof of Weights.

> For reference, metrics containing `PoW` refer to Proof of Weights circuits developed using the Jolt proof system. Metrics containing `Proof of Weights` refer to circuits developed using the Circom proof system.

[See proving benchmarks on the Omron Subnet →](https://wandb.ai/inferencelabs/omron/reports/SN2-PoW-256-min_response_time-SN2-Proof-of-Weights-1-56-min_response_time-SN27-PoW-256-min_response_time-24-11-06-19-39-03---VmlldzoxMDA1Nzk0MQ)

### Impact Assessment

#### Performance Metrics

- Proof verification times vary but are generally under 1s per proof
- Proof sizes vary depending on circuit complexity, in the range of 5kb-50kb
- Trusted setup files (KZG commitments) must be present on each subtensor node for the purposes of proof verification. In total, 16GB of persistent storage is required for the KZG files.
- Proving times, memory and CPU consumption are all subject to the dynamic complexity of each circuit defined. Decentralized proving clusters such as [Omron] can abstract the work of proving away from validators entirely as demonstrated in the [Proof of Weights SDK].
- Assuming full feature uptake and 64 subnets, the total number of proofs required per day is estimated at 25,600. To meet this demand, the following chart depicts the proving capacity of the [Omron] subnet against the number of proofs required per day.

![Proving demand vs capacity](https://github.com/user-attachments/assets/b1faafc6-85b6-4d73-a903-7ee1e4481f8c)

> Note that each circuit must be constrained in terms of maximum complexity to prevent the circuit from becoming too large to be practical for the network.

#### Scaling Considerations

Based on the average verification times from the benchmarks above, scaled load is estimated as follows assuming 100% feature uptake.

| Scenario             | Validators | Subnets | Total Verification Time per Tempo |
| -------------------- | ---------- | ------- | --------------------------------- |
| Current              | 20/subnet  | 64      | 597s                              |
| Current with groth16 | 20/subnet  | 64      | 384s                              |
| Future\*             | 10/subnet  | 64      | 298.5s                            |
| Future with groth16  | 10/subnet  | 64      | 192s                              |

(\*) As Bittensor progresses through the dTAO upgrade, validator consolidation is anticipated and the number of active validators on each subnet will decrease. Yuma consensus is less effective as the marginal validator is removed, and what happens when a single validator remains? Who will ensure they are honest and will treat all miners fairly? Proof of Weights is designed such that a single validator can prove they are honest and fair.

### Mitigating Chain Bloat

To mitigate chain bloat from proof sizes shown in the benchmarks, proofs will be stored off-chain using IPFS. Validators will only submit a 64-byte hash of the IPFS Content Identifier (CID) to the chain during commit, while storing the full proof data on IPFS. This approach minimizes on-chain storage requirements while maintaining proof verifiability through the immutable CID hash.

To complete an IPFS integration, the following steps are required.

1. **Validator Node Requirements:**

   - Each validator must either run an IPFS node or have access to an IPFS gateway (such as Pinata or Infura)
   - The validator software must include IPFS client capabilities for proof submission

2. **Proof Submission Process:**

   - The validator first pins the proof data to IPFS
   - Waits for confirmation of successful pinning
   - Retrieves the CID and submits only the CID hash to the chain
   - The proof data must remain pinned for a minimum duration to allow for verification

3. **Verification Process:**
   - Blockchain validators retrieve the proof data using the submitted CID
   - If the proof data cannot be retrieved, the weight submission is considered invalid
   - A timeout period is enforced for proof retrieval attempts

The efficacy of this approach is demonstrated in the below proof and public signal projections for a 30-day period.

![Proof and Public Signal Projections](https://github.com/user-attachments/assets/67d2dda7-9c8b-41ff-a677-587130a68020)

The process for proof submission using the IPFS off-chain storage mechanism is outlined as follows.

![Proof Submission Flowchart with IPFS](https://github.com/user-attachments/assets/2da11547-a59e-4f17-9222-db47e73b2e9c)

## Backwards Compatibility

To support validator submission of weights accompanied by cryptographic proofs, a new version of Bittensor will be released. Subnet owners that choose to enable this hyperparameter will require all validators participating in that subnet to upgrade to this new version in order to successfully commit proofs and public signals alongside their weights.

Similarly to how commit-reveal is implemented as an optional feature to be enabled at the discretion of a subnet owner, Proof of Weights is also optional. Subnet owners who choose not to enable Proof of Weights will not have any impact on the operation of their subnet, miners, or validators. Proof of Weights can be enabled or disabled at any time without impact to the subnets operations (minus the typical inconvenience of deploying a subnet software update across miners/validators).

## Reference Implementation

[Omron] has served as a testing ground for proof of weights since June 2024. Through extensive testing and iteration, the subnet has developed robust proof of weights circuits for subnets 2, 27 and 48, with additional circuits in active development. The reference implementations below demonstrate production-ready circuits that have been battle-tested in a live environment.

| Subnet    | Circuit Link                                                                                                                                                                                     |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Subnet 2  | https://github.com/inference-labs-inc/omron-subnet/tree/0627356363d2745d7511c1e9a28e27df7e242261/neurons/deployment_layer/model_55de10a6bcf638af4bc79901d63204a9e5b1c6534670aa03010bae6045e3d0e8 |
| Subnet 27 | https://github.com/inference-labs-inc/omron-subnet/tree/0627356363d2745d7511c1e9a28e27df7e242261/neurons/deployment_layer/model_b7d33e7c19360c042d94c5a7360d7dc68c36dd56c449f7c49164a0098769c01f |
| Subnet 48 | https://github.com/inference-labs-inc/omron-subnet/blob/d7d61c665835b21497f8e909de7aa51134f13a93/neurons/deployment_layer/sn48/src/main.circom#L1                                                |

Additionally, a fork of the developer documentation repository has been created to provide simulations of changes to Yuma consensus and the below referenced approaches to mitigate the issue of weight copying.

[Developer Documentation Simulation](https://github.com/inference-labs-inc/pow-simulations)

## Security Considerations

Though this proposal primarily exists to increase the level of security throughout the network, there are important considerations regarding the security of this update itself.

An additional layer of security is provided by the recently executed Master Services Agreement with [ImmuneFi], which establishes a $100,000 bug bounty program aimed at incentivizing white-hat hackers to identify and report vulnerabilities within the [Omron] codebase. By proactively identifying potential flaws through external audits and community-driven bounty programs, the subnet will ensure continuous security vigilance.

In the event that Inference Labs ceases to exist or Subnet 2 is deregistered, the work conducted remains secure as all development and contributions are open source. The Proof of Weights protocol is structured to operate independently of a subnet, ensuring that its core functionalities persist. The open-source nature of the project enables the broader community to scaffold and maintain decentralized infrastructure to execute all computations.

Additionally, the incentive structure built into the subnet model encourages continuous development, fostering innovation in proof generation times and sizes, thereby mitigating chain bloat and promoting scalability as the number of subnets increases.

Another potential concern is validators copying each other's proofs from extrinsics submitted to the chain. This risk is mitigated by incorporating the validator's UID as an input signal. The blockchain verifies the UID in the proof against the submitting key to ensure a match, preventing such copying.

A significant concern related to the issue of weight copying exists in a situation where a dishonest validator employs artificially generated input data designed to mimic real-world data, in an effort to supply cryptographic proof that they operated within the subnet and effectively weight copy by generating valid proofs based on false pretenses. As described, this situation could present a challenge in that validators could observe resulting weight vectors across submissions and pre-emptively calculate consensus weights, using these calculations to craft input data designed to provide proven “perfect” consensus weights.

In terms of weight copying, four potential solutions are proposed, all of which build upon advancements made in commit reveal and proof of weights as outlined in this proposal. The emphasis in all of these initiatives is on balancing instant feedback with risk of exposing critical data to weight copying parties.

### Countering Weight Copying

Proof of Weights used in conjunction with the following commit-reveal related strategies can provide a powerful and demonstrable defense against weight copying.

#### Peer Prediction

This strategy, first introduced in the [BT Weight Copier Paper], leverages peer prediction mechanisms to assess the authenticity of weight generation. By analyzing correlations between validator inputs and comparing them against expected statistical patterns, the system can identify likely cases of weight copying. When integrated with Proof of Weights, the circuit inputs themselves become valuable signals for peer prediction analysis, enabling robust detection of copied versus independently generated weights.(\*)

#### Range Proofs

Through the use of ZKRPs, validators prove that miners fall within specific percentile ranges on the incentive curve. For example, with a bucket size of 5, validators prove and immediately reveal that miners belong to ranges like [1,5], [6,10], etc., while committing to their exact weights for later revelation. This provides miners with actionable real-time feedback about their relative performance without exposing enough granular information for effective weight copying. Subnet owners can configure bucket sizes to balance miner feedback against copying resistance.

<details open>
<summary>View Analysis and Simulation</summary>

The range proofs approach builds upon the ranking-based solution by introducing bucketed ranges rather than exact rankings. While ranking-based solutions provide miners with precise ordinal positions but risk enabling weight copying, range proofs strike a balance by revealing only approximate ranking ranges (e.g. [1-5], [6-10], etc).

Through simulation analysis across multiple subnets and strategies, we observe that range proofs with appropriately sized buckets can significantly reduce a weight copier's ability to exploit the system while still providing miners with actionable feedback about their relative performance. Key findings include:

1. Larger bucket sizes (e.g. ranges of 10 vs 5) correlate with reduced weight copier effectiveness, as the granularity of ranking information decreases
2. Even sophisticated copying strategies like relative historical ranking struggle to achieve meaningful gains when bucket sizes are optimized
3. Miners retain sufficient visibility into their approximate network standing to make informed decisions and avoid deregistration
4. The approach maintains the security benefits of commit-reveal while addressing the critical need for real-time performance feedback

The simulations demonstrate that range proofs can effectively balance the competing priorities of miner transparency and copy resistance. By tuning bucket sizes, subnet owners can customize the tradeoff between security and usability based on their specific needs.

#### Subnet 2, Conceal Period of 13 intervals

![SN2 Conceal 13 Range Proofs](https://github.com/user-attachments/assets/b27bd010-5b41-41e0-a84b-88e550b1207a)

#### Subnet 19, Conceal Period of 13 intervals

![SN19 Conceal 13 Range Proofs](https://github.com/user-attachments/assets/2269114b-a2c3-4188-bc61-dd5a284ac960)

#### Subnet 27, Conceal Period of 13 intervals

![SN27 Conceal 13 Range Proofs](https://github.com/user-attachments/assets/bd293087-6caf-4850-9696-6e21bf4a2c3c)

</details>

#### Rank Proofs

Rank proofs extend range proofs by allowing validators to cryptographically prove and immediately reveal precise ordinal rankings while concealing actual weight values until later. This provides miners with exact performance standings in real-time, while making weight copying difficult since actual consensus values remain hidden during the concealment period. The system preserves the security benefits of commit-reveal while giving miners the continuous feedback needed to maintain competitive performance and avoid deregistration.

<details open>
<summary>View Analysis and Simulation</summary>

The rank proofs approach provides miners with precise ordinal positions while still concealing actual weight values during the concealment period. Through simulation analysis across multiple subnets and strategies, we observe that rank proofs can effectively balance miner transparency with copy resistance. Key findings include:

1. Weight copiers struggle to exploit ranking information alone, as the actual consensus values remain hidden until revelation
2. Miners receive exact performance standings in real-time, enabling rapid response to performance issues
3. The approach maintains the security benefits of commit-reveal while addressing the critical need for continuous feedback
4. Even sophisticated copying strategies like historical rank correlation achieve limited gains without access to actual weights

The simulations demonstrate that rank proofs successfully provide miners with actionable real-time performance data while preserving robust defense against weight copying. By revealing only ordinal positions and concealing weights, the system enables miners to track their standing and avoid deregistration without compromising security.

When combined with range proofs, subnet owners gain flexible options for balancing transparency and security based on their specific needs - using precise rankings when copy resistance is less critical, or bucketed ranges when additional security is required.

#### Subnet 2, Conceal Period of 13 intervals

![SN2 Conceal 13 Rank Proofs](https://github.com/user-attachments/assets/91fa012d-eedf-460e-9c27-6b1c765d8975)

#### Subnet 22, Conceal Period of 1 intervals

![SN22 Conceal 1 Rank Proofs](https://github.com/user-attachments/assets/0d7ff7ce-1355-45b5-a8ae-ae9ec83e041c)

#### Subnet 27, Conceal Period of 3 intervals

![SN27 Conceal 3 Rank Proofs](https://github.com/user-attachments/assets/b3ae411f-cd93-4619-b4d2-7ecdc5d08d24)

</details>

#### Partial revelation

In this strategy, subtensor nodes select a random subset of validators based on a stake threshold percentage, compelling a subset of validators to expose their weight vectors immediately. Miners will receive partial feedback instantaneously, allowing them to gauge their incentive without full visibility. During revelation, the remaining validators are compelled to reveal their previously proven weights and consensus is then calculated for the previous tempo. Weight copiers aren’t provided enough information to calculate consensus values, though miners are provided with a reasonably high degree of visibility into their incentive instantly.

A high level overview of the above strategies is presented in the below flowchart.

![Proof of Weights Strategies](https://github.com/user-attachments/assets/b954a40e-0a5c-4542-ba4b-9f402760c340)

## Copyright

This document is licensed under The Unlicense.

---

(\*) Public signals increase to a dynamic size in the case that input signals are used in a peer prediction style approach to weight copying. To reduce the amount of load on the chain to a smaller, fixed size and for the purposes of this BIT, output signals (weights and validator UID) are sufficient.

[BT Weight Copier Paper]: https://docs.bittensor.com/papers/BT_Weight_Copier-29May2024.pdf
[EZKL]: https://github.com/zkonduit/ezkl
[Circom]: https://github.com/iden3/circom
[ZKRPs]: https://eprint.iacr.org/2024/430.pdf
[groth16]: https://eprint.iacr.org/2016/260.pdf
[Omron]: https://github.com/inference-labs-inc/omron-subnet
[ONNX]: https://onnx.ai/
[Yuma Consensus]: https://docs.bittensor.com/yuma-consensus
[ImmuneFi]: https://immunefi.com/
[Proof of Weights simulation repository]: https://github.com/inference-labs-inc/pow-simulations
[Proof of Weights SDK]: https://github.com/inference-labs-inc/proof-of-weights-sdk
