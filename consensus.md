# **Consensus chapter**
# Slots, Epochs, Validators and Blocks

The two most important intervals are the *slot*, which is 12 seconds exactly, and the *epoch*, which is 32 slots, or 6.4 minutes. 
![](https://hackmd.io/_uploads/H1327DRM6.png)
Validators need to deposit 32 ETH, run a validator node to propose blocks and sign attestations. Every validator receives its unique index.

Every slot, exactly one validator is selected to propose a block. The block contains updates to the beacon state, including attestations, slashings and user transactions. The proposer shares its block with the whole network. A slot can be empty.
![](https://hackmd.io/_uploads/HJsFQvRzT.png)


# Attestations and Committees

An attestation is a validator’s vote, weighted by the validator’s balance. An attestation has 32 slot chances for inclusion on-chain.

Every epoch, every validator gets to share its view of the world exactly once, in the form of an attestation. An attestation contains votes for the head of the chain that will be used by the LMD GHOST protocol, and votes for checkpoints that will be used by the Casper FFG protocol.

The work of attesting is divided among subsets of the validator set (64 committees for each slot) and spread across an epoch (6.4 minutes). Each validator participates in only one of the committees. One validator from the committee will be chosen to be the aggregator, while the other validators are attesting.

A random seed is used to select all the committees and proposers for an epoch. During each epoch, the beacon chain accumulates randomness from proposers via the RANDAO and stores it.

The composition of the committees for an epoch is fully determined at the start of an epoch by the active validator set for that epoch, and the RANDAO seed value at the start of the previous epoch. The RANDAO seed at the end of epoch N is used to compute validator duties for the whole of epoch N+2. Thus, validators have at least one full epoch to prepare themselves for any duties, but no more than two.

![](https://hackmd.io/_uploads/r150Nv0fp.png)

When making its attestation, the validator sets a single bit to indicate which member of the committee it is. That is sufficient, in conjunction with the slot number and the committee index, to uniquely identify the attesting validator in the global validator set. This attestation will later be aggregated with other attestations from the committee that contain identical data. An attestation is added to an aggregate by copying over its bit from the aggregation_bits field and adding (in the sense of elliptic curve addition) its signature to the signature field. Aggregate attestations can be aggregated together in the same way, but only if their aggregation_bits lists are disjoint: we must not include a validator more than once. This aggregate attestation will be gossiped around the network and eventually included in a block.




# Checkpoints, Justifying and Finalising

A transaction has finality in distributed networks when it is part of a block that can't change without a large amount of ETH getting burned. On proof-of-stake Ethereum, this is managed using checkpoint blocks. 

The first block in each epoch is a checkpoint. Validators vote for pairs of checkpoints that it considers to be valid: 
- If a pair of checkpoints attracts votes representing at least two-thirds of the total staked ETH, the checkpoints are upgraded
- the more recent of the two (target) becomes justified. The earlier of the two is already justified because it was the "target" in the previous epoch. Now it is upgraded to finalized.
![](https://hackmd.io/_uploads/SJewVwCM6.png)

Casper FFG finalises checkpoints, the first slots of epochs. When we have finalised the checkpoint in epoch N, we have finalised everything up to and including slot 32N. This includes all of epoch N-1 and the first slot of epoch N. But we have not yet finalised epoch N - it still has 31 unfinalised slots in it.