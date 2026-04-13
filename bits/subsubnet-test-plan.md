# Subsubnet Functionality BIT

## Test Areas

### ✅ Weight Cleanup
- [ ] Weights are cleaned when `MechanismCountCurrent` decreases
- [ ] For each subnet mechanism, when a miner is deregistered or leaves, their weights are cleaned across **all** mechanisms

### ✅ Limit Update Logic
- [ ] Decreasing `MechanismCountCurrent` reduces `MechanismCountCurrent` immediately

### ✅ Validator Permissions & Hyperparameters
- [ ] Max allowed `MechanismCountCurrent` is 8 (ids 0-7)
- [ ] Validators **cannot** modify other subnets’ parameters
- [ ] `MechanismCountCurrent` is readable by API per subnet and globally

### ✅ Compatibility with Existing Systems
- [ ] Confirm `mech_id = 0` acts as default fallback for weight operations
- [ ] Validate `set_weights`, `commit_weights`, `reveal_weights` apply correctly on mech_id=0
- [ ] Confirm legacy operations still work as expected when subsubnet feature is not used (regression)

### ✅ Weight Setting Restrictions
- [ ] Prevent weight setting above `MechanismCountCurrent`
- [ ] Prevent CR2/CR3 weight commits/reveals for disabled mechanisms
- [ ] Validators **can** set weights above hyperparameter but below `MechanismCountCurrent`

### ✅ Miner-Mechanism Interaction
- [ ] Miner can participate in multiple or all mechanisms (bond, weights, rewards)
- [ ] Support miner existence with no weights on any mechanism
- [ ] Support mechanism existence with no miner weights at all
- [ ] Ensure correct weights can be retrieved per miner per mechanism

### ✅ Emissions and Incentives
- [ ] Ensure `MechanismCountCurrent` does not exceed global limit
- [ ] Validate weight independence across mechanisms
- [ ] Check total emission is split among mechanisms evenly
- [ ] Validate that per-mechanism incentives are distributed proportionally to miner weights
- [ ] Trust, vtrust, consensus, etc. are calculated per mechanism and then aggregated
- [ ] Rounding logic does not lose incentive
- [ ] Sum of all mechanism emissions matches total emission

### ✅ Emergency and Recycling Behavior
- [ ] Empty mechanism enters "Yuma Emergency Mode", emission distributed by stake
- [ ] If consensus sum is 0 for a mechanism, trigger emergency fallback
- [ ] Recycling incentives per mechanism via validator vote by subnet owner ID
- [ ] Miner without any mechanism weights receives **no** reward

### ✅ Subnet Compatibility Testing
- [ ] Subnets with `MechanismCountCurrent = 1` (id = 0 only) operate without change
- [ ] Regression tests confirm staking, pruning, emissions, and validator selection work as before
- [ ] Switching mechanisms off restores old behavior without side effects
- [ ] Storage layout and RPC responses remain compatible for older miners/validators
- [ ] Validate upgrade paths: backward compatibility for older nodes (v2+)

### ✅ Governance and Parameter Controls
- [ ] Subnet owner can set incentive proportions per mechanism
- [ ] Global mechanism limit is dynamically adjustable
- [ ] Decreasing global limit propagates cleanups in all subnets
- [ ] Validator vote (via `kappa`) determines miner emission share per mechanism

---

## References
- Affects: Miner incentives, emission mechanics, subnet behavior
