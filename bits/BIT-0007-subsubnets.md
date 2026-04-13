# BIT-0007: Subnets Mechanisms

- **BIT Number:** 0007
- **Title:** Sub-subnets
- **Author(s):** Rhef, Greg Zaitsev
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-06-13
- **Updated:** 2025-06-13

## 🔍 Abstract

This BIT proposes the introduction of subnet mechanisms, a hierarchical layer within each subnet to support multiple, independent weight spaces, emissions, and incentive flows under a single subnet umbrella. Subnet mechanisms enable fine-grained control over miner task allocation, incentive distribution, and validator decision-making. Each subnet can define up to 8 subnet mechanisms, with weights and rewards tracked independently per subnet mechanism.

## 🔧 Motivation

Subnets today treat miner weights as a single flat structure, limiting expressivity in multi-task or multi-objective networks. Introducing subnet mechanisms allows a single subnet to support multiple distinct task markets, enabling specialized incentive tracking, validator scoring, and emission logic per group. This facilitates complex workloads, parallel task handling, and flexible protocol design without fragmenting network security or duplicating validator logic.

## 🧪 Specification

### Subnet mechanism Limits
- Each subnet defines a `MechanismCountCurrent` hyperparameter (default = 1), which becomes the subnet mechanism limit immediately.
- A global limit `MaxMechanismCount` acts as a ceiling.

### Weight Operations
- `set_weights`, `commit_weights`, and `reveal_weights` accept a `mech_id` argument.
- Legacy operations default to `mech_id = 0`.
- Weight writes are disallowed above the current `MechanismCountCurrent`.

### Validator Permissions
- Only the subnet owner or sudo can change the `MechanismCountCurrent` or emission proportions.
- Validators may set weights for any subnet mechanism within the current limit.

### Emission Logic
- Emission is distributed across subnet mechanisms according to a configured ratio (default is even distribution).
- Each subnet mechanism computes trust, consensus, and incentive separately.
- Final vtrust is aggregated across subnet mechanisms weighted by their emissions.
- Rounding is preserved across subnet mechanism splits to ensure exact conservation of emitted tokens.

### Edge Cases
- If a subnet mechanism has zero consensus, it enters “Yuma emergency mode” and allocates emission proportional to stake.
- Miners without weights in a subnet mechanism receive no emission from it.
- Subnet mechanisms with no miners are gracefully handled.

### Compatibility
- Subnets with a single subnet mechanism behave exactly as today (ID 0).
- Legacy miners/validators interoperate with subnet mechanism-enabled subnets via `mech_id = 0`.
- Storage and RPC interfaces remain backward-compatible.

## ✅ Rationale

Subnet mechanisms allow a single subnet to support multiple incentive partitions, enabling more advanced use cases (e.g. routing, filtering, classification, multitask models). They preserve validator overhead by avoiding new subnets while enabling greater expressivity. The design enforces strict backward compatibility and safe transitions when limits change.

## 📘 Reference Implementation

- Will be implemented in the `subtensor` core repo.
- Interfaces for weight setting, emission, and validator ranking will be extended to include `mech_id`.

## 🧱 Backward Compatibility

- All existing weight operations apply to `mech_id = 0`.
- Subnets not opting into subnet mechanisms will remain functionally identical.
- Miners and validators on older versions will continue functioning under mech_id 0.

## 📈 Test Cases

See BIT test document `subsubnet_test_plan.md`.

## 💬 Discussion

- Emission proportion customization per subnet mechanism opens design space for subnet-specific task prioritization.
- Validator voting on miner subnet mechanism weights (via kappa) may be used for consensus and reward routing.
- Handling of dynamic emission distribution and cleanups when limits decrease must be conservative and race-free.

## 🛠️ Future Work

- **Custom Emission Proportions**: Subnet owners will be able to customize the proportion of emission allocated to each subnet mechanism, enabling tailored incentive strategies based on task complexity or utility.

- **Dynamic Global Subnet mechanism Limit**: A globally enforced ceiling on subnet mechanism counts will be adjustable over time. Reductions to this limit will automatically trigger cleanup of excess subnet mechanism data across all subnets.

- **Hyperparameter Governance**: Subnet owners will gain control over additional subnet mechanism-specific hyperparameters beyond the subnet mechanism limit, allowing more granular tuning of behavior.

- **Validator-Driven Incentive Routing**: Using the kappa stake majority mechanism, validators may vote to adjust miner incentive shares within subnet mechanisms, supporting flexible prioritization of behaviors and tasks.

- **Additional Governance Extensions**: Future extensions may include subnet mechanism-specific pruning policies, trust calculation curves, or dynamic validator selection strategies.


## 🔐 Security Considerations

The introduction of subnet mechanisms introduces additional state surfaces and per-subnet mechanism tracking, which must be secured against manipulation:

- **Permission Enforcement**: Only subnet owners or sudo must be able to modify `desired_subnet mechanism_limit`, emission proportions, or trigger subnet mechanism resets. Improper permission checks could allow hostile takeovers of reward logic.
  
- **Weight Isolation**: Weights across subnet mechanisms must remain isolated. Cross-contamination could allow miners to gain rewards in unintended subnet mechanisms.
  
- **Rounding and Overflow**: Emission rounding and aggregation must be implemented carefully to prevent underflow/overflow or token inflation.
  
- **Backward Compatibility**: Any logic paths introduced for subnet mechanism IDs must default to safe values (e.g., mech_id = 0) to avoid denial-of-service for legacy miners and validators.

- **Cleanups**: When limits are decreased, weight purging must be idempotent and bounded to prevent validator or miner state desynchronization.

## © Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
