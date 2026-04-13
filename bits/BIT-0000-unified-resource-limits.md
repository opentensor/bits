# BIT-0000: Unified resource limits

- **BIT Number:** 0000
- **Title:** Unified resource limits
- **Author(s):** Rhef
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-01-28
- **Updated:** 2025-01-28

# Abstract

Subnets need a more flexible way to manage resources the chain allows them to use - they are currently constrained unnecessarily and it limits their potential, requiring cumbersome workarounds, dragging down the progress of decentralized AI in Bittensor ecosystem.

# Unified resource limits

## Storage

The subnet owner has an extremely limited amount of storage that they can use to direct the subnet \- some hyperparamters have more precision that they actually need, so one could use lower bits of registration price settings, bond\_moving\_average etc  
Nowadays a validators neuron can use a certain amount of chain resources:

- Some information can be stored in the metagraph  
  - Axon IPv4 address and port which can be updated every 100 blocks, so 6 bytes of storage and one 6-byte input signed transaction per 100 blocks  
  - Two Encryption certificates with types \- 514 bytes  
  - Knowledge commitments  
    - 3100 max storage per neuron per epoch  
    - Minimum space per commitment: 100 bytes, so up to 31 writes  
    - Maximum space per commitment: 1024 bytes if timelocked, 512 bytes otherwise  
    - Multiple commitments can coexist when timelocked, occupying space  
  - H160 association (irrelevant here in practice)  
- Weights \- up to 64 validators \* 2 bytes \* number of mechanisms. One set of 64 can be written every 100 blocks.  
- Some validator-specific information is derived by Yuma, but still takes storage space for every validator neuron  
  - If bonds\_moving\_average is small and liquid alpha is off, bonds closely show the past state of weights

## Compute

Other than a minimum amount of compute required for storage, the subnet can only use compute power to execute Yuma in the epoch block before user transactions, which is forced to take place every 361 blocks. The computational complexity depends on max\_uids and max mechanisms.

## Problem

Some subnets need more compute or more storage space than knowledge commitments can provide, and in order to get that storage, they started using weight settings on mechanism IDs that they no longer use… While it is a solution that works, it’s hacky and it causes Yuma on that mechanism to needlessly calculate consensus, vtrust, incentive etc. This subnet has an extremely small amount of active neurons, it doesn’t use timelocked knowledge commitments or encryption certificates and it is unfair to penalize it for excessive storage usage on plaintext knowledge commitments while they strongly conserve the chain resource everywhere else.

Some subnets would like to use smart contracts to enhance the functionality or to provide server-side hooks to Yuma in order to deploy useful (and proven\!) modifications such as server-side winner takes all.

## Proposal

I propose that these limits are unified into SUBNET GAS. The key factors in a limit system that I can see are:

1. The amount of signed transactions  
2. The amount of data held at peak in the persistent storage  
3. The amount of data written to the chain (per 1000 blocks?)  
4. The amount of CPU used for Yuma and for smart contract hooks

These should be tracked and limited by the chain so they don’t exceed limits in some window such as 7200 blocks. How many megabytes and CPU seconds should we allow subnets to consume?

## Extra payment

If a subnet owner would like to use more resources than the chain normally allows them, it should be possible for them to pay for it. The storage and CPU fees will be taken from the subnet owner's wallet. 

## Control

The subnet owner should remain in control of how his miners and validators can spend resources. I propose to add a set of hyperparameters which will allow the subnet owner to:

- Set how often they need the epoch block to run \- in some subnets (sn12 for example) running it every 722 blocks would not change anything, but epoch is being run every 361 blocks wasting (half or so) resources. Some subnets could easily get by running the epoch once per day  
- Set various limits to tweak how much of which resources the neurons in the subnet should be able to consume per 7200 blocks  
  - Tempo length multiplier (causes tempo to run every 361 \* N blocks)  
  - Gas limit for the in-epoch smart contract (between  
  - Gas limit for the end-of-epoch smart contract  
      
  - Budget for fueling extra spending  
  - Minimum number of blocks between set weights / commit weights  
      
  - Minimum number of blocks between validator knowledge commitments (0 to disable kc for validators completely)  
  - Minimum number of blocks between miner knowledge commitments (0 to disable kc for miners completely)  
  - Maximum size of a validator knowledge commitment post  
  - Maximum size of a miner knowledge commitment post  
  - Maximum number of knowledge commitment slots for validators  
  - Maximum number of knowledge commitment slots for miners  
      
  - Encryption certificate space limit multiplier for validators (gives N \* 257 bytes of effective limit)  
  - Encryption certificate space limit multiplier for miners (gives N \* 257 bytes of effective limit)  
  - Minimum blocks between encryption certificate setting  
      
  - Minimum blocks between axon setting (0 to disable axons completely)  
- Set an address of a smart contract that should be executed during the epoch to modify Yuma in order to inject functionality such as server-side winner-takes-all (very exploitable \- not in the first version)  
- Set an address of a smart contract that should be executed after the epoch block. First burn uid as origin?

## Fees

Currently, CPU is available to EVM users, as is storage. Last time I calculated the price, it was around 0.3 Tao per megabyte written. These prices are just automatically according to supply and demand. We’ll use the exact same system to measure and price the resources used by subnets and to adjust those prices to supply / demand ratio.


## Motivation / Rationale / Alternatives

The alternatives are unfeasible:
- hacking, such as serializing knowledge commitment data into mechanism weights or encryption certificates
- using custom evm contracts as storage, which forces all miners and validators to generate, fund and maintain h160 keys
- using multiple neurons to bypass the per-neuron limits

Some basic tooling such as a small amount of storage is necessary to enable building subnets and 
in ambitious projects 512 bytes might simply not be enough.


## Backwards Compatibility

With default settings the current subnets should not be running into any limits, at least until 
the gas price increases due to high demand on compute/storage from running subnets or smart contracts.


## Security Considerations

The smart contract executed at the end of every epoch block could potentially try to overload
a validator, but a gas limit would catch it just as it catches regular runaway infinite loops 
in smart contracts for many years now. That smart contract wouldn't really be much different.


## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
