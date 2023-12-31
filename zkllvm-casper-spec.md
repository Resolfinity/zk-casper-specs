# Casper on zkllvm and Nil proof market

To get rid reorgs issues, we are listening for new slots with some reasonable delay, e.g. 3-4 slots.

## High-level architecture

- beacon node with rpc api
- relay server
  - listens for new slots
  - retreives data from beacon node
  - prepares data as circuit inputs
  - sends proff-building orders to Nil proof market
  - gets proofs from market
  - sends proofs to evm smart contracts
- nil proof market
- evm smart contracts
  - deployed on target blockchains
  - verify proofs onchain
  - track source chain's finalized slots hashes
  - consist
    - nil verifier contract
    - custom gates

## What we have to prove

### For new epoch

- Get active validator set
- Calculate compute_shuffled_index

### For every new slot

- Select attestations with majority of source/target/head/aggregation_bits
  - skip minor attestations at least for now to simplify logic
- For each attestation
  - get participated validators indexes
  - get participated validators balances
  - get participated validators public keys
  - check if aggregated signature is correct
  - generate proof that
    - attestation of commitee N
    - has correct signature
    - and aggregated balance B
    - public inputs:
      - slot
      - balance
      - commitee index
      - target/source
- For new epoch
  - Prove that proofs generated before, with the same target/source, have more than 2/3 of total_active_balance
  - Public inputs
    - finalized slot id
    - finalized block hash
    - finalized eth1 block, state root, ...

# Casper zk proof system on nil's zkllvm

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Constants](#constants)
- [Configuration](#configuration)
  - [Misc](#misc)
- [Containers](#containers)
  - [`ZkClientSnapshot`](#ZkClientSnapshot)
  - [`ZkClientUpdate`](#ZkClientUpdate)
  - [`ZkClientStore`](#ZkClientStore)
- [Zk client state updates](#zk-client-state-updates)
  - [`validate_zk_client_update`](#validate_zk_client_update)
  - [`apply_zk_client_update`](#apply_zk_client_update)
  - [`process_zk_client_update`](#process_zk_client_update)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

The **casper zk proof system** is a zk-proof based client, storing eht2 finalized epochs headers, made in form of evm smart contract.

This diagram shows the basic procedure by which a zk client learns about finalized epochs:

https://whimsical.com/sync-commitees-DFQFTsKn3Wgpj7s4G3FuGw@7YNFXnKbYjQ3UhbX7EjGL

We assume a light client already has a block header at 1st slot of finalized epoch F.
The light client wants to finalize epoch F+1. The steps the light client needs to take are as follows:

1. get randao from epoch N, calculate seed used for shuffle function
2. calculate commitees members and balances of epoch N+2 
3. verify bls signatures of attestations
4. check that total balance of voted validators more than 2/3 of total balance
5. check CASPER rules 
6. update last justified and finalized epochs


## Commitee members randomization

1. The RANDAO seed at the end of epoch N is used to compute validator duties for the whole of epoch N+2. This interval is controlled by MIN_SEED_LOOKAHEAD via the get_seed() function. Thus, validators have at least one full epoch to prepare themselves for any duties, but no more than two.

### get_seed()
```python
def get_seed(state: BeaconState, epoch: Epoch, domain_type: DomainType) -> Bytes32:
    """
    Return the seed at ``epoch``.
    """
    mix = get_randao_mix(state, Epoch(epoch + EPOCHS_PER_HISTORICAL_VECTOR - MIN_SEED_LOOKAHEAD - 1))  # Avoid underflow
    return hash(domain_type + uint_to_bytes(epoch) + mix)
```

### 
```python
def compute_shuffled_index(index: uint64, index_count: uint64, seed: Bytes32) -> uint64:
    """
    Return the shuffled index corresponding to ``seed`` (and ``index_count``).
    """
    assert index < index_count

    # Swap or not (https://link.springer.com/content/pdf/10.1007%2F978-3-642-32009-5_1.pdf)
    # See the 'generalized domain' algorithm on page 3
    for current_round in range(SHUFFLE_ROUND_COUNT):
        pivot = bytes_to_uint64(hash(seed + uint_to_bytes(uint8(current_round)))[0:8]) % index_count
        flip = (pivot + index_count - index) % index_count
        position = max(index, flip)
        source = hash(
            seed
            + uint_to_bytes(uint8(current_round))
            + uint_to_bytes(uint32(position // 256))
        )
        byte = uint8(source[(position % 256) // 8])
        bit = (byte >> (position % 8)) % 2
        index = flip if bit else index

    return index
```

The composition of the committees for an epoch is fully determined at the start of an epoch by (1) the active validator set for that epoch, and (2) the RANDAO seed value at the start of the previous epoch.





## Constants 
todo:

| Name                        | Value                                                                |
| --------------------------- | -------------------------------------------------------------------- |
| `FINALIZED_ROOT_INDEX`      | `get_generalized_index(BeaconState, 'finalized_checkpoint', 'root')` |
| `MAX_VALIDATORS_PER_COMMITTEE`| **?? tbd ??** |
| `EPOCHS_PER_HISTORICAL_VECTOR`| **?? tbd ??** |


These values are the [generalized indices](https://github.com/ethereum/eth2.0-specs/blob/dev/ssz/merkle-proofs.md#generalized-merkle-tree-index) for the finalized checkpoint and the next sync committee in a `BeaconState`. A generalized index is a way of referring to a position of an object in a Merkle tree, so that the Merkle proof verification algorithm knows what path to check the hashes against.


## Beacon chain containers

### `BeaconBlockHeader`

```python
class BeaconBlockHeader(Container):
    slot: Slot
    proposer_index: ValidatorIndex
    parent_root: Root
    state_root: Root
    body_root: Root
```

## Containers

### `ZkClientSnapshot`

```python
class ZkClientSnapshot(Container):
    # Beacon block header
    header: BeaconBlockHeader # not sure if we need it
    last_finalized_epoch: BeaconBlockHeader 
    last_justified_epoch: BeaconBlockHeader
    next_commitee_shuffle_hash: Hash 
```

The `ZkClientSnapshot` represents the zk client's view of the most recent block header that the zk client is convinced is securely justified and finalized epochs of the chain. The zk client stores the last_finalized_epoch header itself, so that the zk client can then ask for Merkle branches to authenticate transactions and state against the finalized header.

next_commitee_shuffle_hash is a part of final proof public inputs and represents validators split by commitees for the future epoch. 


### `ZkClientUpdate`

```python
class ZkClientUpdate(Container):
    # Update beacon block header
    header: BeaconBlockHeader
    # ?? we update client once per epoch, should we pass all block headers from finished epoch as array to store in zk client? if yes - we can do it in form of merkle proofs or as a part of zk proof.

    # Finality proof for the update header
    finality_header: BeaconBlockHeader
    finality_branch: Vector[Bytes32, floorlog2(FINALIZED_ROOT_INDEX)]
    
    # Finality proof for the update header
    justification_header: BeaconBlockHeader
    justification_branch: Vector[Bytes32, floorlog2(FINALIZED_ROOT_INDEX)]
    
    # root of shuffled validators list
    next_commitee_shuffle_hash: Hash
    
    # Proof of finality
    finality_proof: Proof
    # Fork version for the aggregate signature
    fork_version: Version
```

A `ZkClientUpdate` is an object passed as an input to the proof builder, which contains all of the information needed to produce proof that convince ZkClient about new finalized header. The information included is:

- **`header`**: the header that the zk client will accept if the `ZkClientUpdate` is valid.
- **`finality_header`**: if nonempty, the header whose signature is being verified (if empty, the signature of the `header` itself is being verified)
- **`??? finality_branch`**: the Merkle branch that authenticates that the `header` actually is the header corresponding to the _finalized root_ saved in the `finality_header` (if the `finality_header` is empty, the `finality_branch` is empty too)
- **`??? justification_header`**: if nonempty, the header whose signature is being verified (if empty, the signature of the `header` itself is being verified)
- **`justification_branch`**: the Merkle branch that authenticates that the `header` actually is the header corresponding to the _finalized root_ saved in the `finality_header` (if the `finality_header` is empty, the `finality_branch` is empty too)
- **`finality_proof`**: Proof build by proof producer, which zk client verifies to be convinced about finality and justification epochs
- **`fork_version`**: needed to mix in to the data being signed (will be different across different hard forks and between mainnet and testnets)

### `Proof`
```python
class FinalityProof(object):
    # todo: add proof fields
```
Proof is an result of proof building which is send as a part of solidity calldata to the ZkClient smart contract.

### `ZkClientStore`

```python
@dataclass
class ZkClientStore(object):
    snapshot: ZkClientSnapshot
```

The `ZkClientStore` is the _full_ "state" of a zk client, and includes:

- A `snapshot`, reflecting a block header that a light client accepts as **finalized** 


The `snapshot` can be updated if the zk client sees a valid `ZkClientUpdate` containing a `finality_header`, and the `proof` that more than 2/3 of validators stake voted for it, it accepts the `update.header` as the new snapshot header. Note that the zk client uses the proof to verify `update.finality_header` (which would in practice often be one of the most recent blocks, and not yet finalized), and then uses the Merkle branch _from_ the `update.finality_header` _to_ the finalized checkpoint in its post-state to verify the `update.header`. If `update.finality_header` is a valid block, then `update.header` actually is finalized.


## Zk client state updates

A zk client maintains its state in a `store` object of type `ZkClientStore` and receives `update` objects of type `ZkClientUpdate`. Every `update` triggers `process_zk_client_update(store, update, current_slot)` where `current_slot` is the current slot based on some local clock.

#### `validate_zk_client_update`

```python
def validate_light_client_update(snapshot: ZkClientSnapshot,
                                 update: ZkClientUpdate,
                                 # genesis_validators_root: Root add some domain checks
                                ) -> None:
    # Verify update slot is larger than snapshot slot
    assert update.header.slot > snapshot.header.slot

    # Verify update does not skip an epoch
    snapshot_epoch = compute_epoch_at_slot(snapshot.header.slot)
    update_epoch = compute_epoch_at_slot(update.header.slot)
    assert update_epoch = snapshot_epoch + 1)
    
    # can we skip some epochs? we have justification bits in state

    # Verify proof in nil verifier
    assert verifier.verify(proof) == true
    
```

This function has next parts:

1. **Basic validation**: confirm that the `update.header` is newer than the snapshot header. **What about skips?**

2. **Verify the proof**: we use other data in update as public inputs.

#### `apply_zk_client_update`

```python
def apply_zk_client_update(snapshot: ZkClientSnapshot, update: ZkClientUpdate) -> None:
    snapshot_epoch = compute_epoch_at_slot(snapshot.header.slot) 
    update_epoch = compute_epoch_at_slot(update.header.slot) 
    if update_epoch == snapshot_epoch + 1:
        snapshot.next_commitee_shuffle_hash = update.next_commitee_shuffle_hash
        snapshot.last_finalized_epoch = update.last_finalized_epoch
        snapshot.last_jusified_epoch = update.last_jusified_epoch
    snapshot.header = update.header
```

#### `process_zk_client_update`

```python
def process_zk_client_update(store: ZkClientStore, update: ZkClientUpdate, current_slot: Slot,
                                # genesis_validators_root: Root need domain info
                            ) -> None:
    validate_zk_client_update(store.snapshot, update)
    # store.valid_updates.add(update) ? not sure if we need it or not.
    apply_light_client_update(store.snapshot, update)
      
```

The main function for processing a light client update. We first validate that it is correct.
*What other checks should we have made?*

# Circuits


### `VerifyAttestation`

Verifies that the signatures in the collected aggregated attestations match the aggregated public keys of the validators participating in the particular sub-committee.

```python
class FullAttestation(object):
    # proof that we have checked validators of this attestation are valid
    validators_proof: Proof
    participants_pub_keys: List[BLSSignature, MAX_VALIDATORS_PER_COMMITTEE]
    signature: BLSSignature
```

```python
def verify_aggregated_attestation(
    signing_root,
    attestation: FullAttestation
) -> None:
    domain = compute_domain(?DOMAIN_AGGREGATE_AND_PROOF?, update.fork_version, genesis_validators_root)
    signing_root = compute_signing_root(signed_header, domain)
    bls.FastAggregateVerify(attestation.participants_pub_keys, signing_root, attestation.signature) 
```


### `VerifyValidatorsShuffle`

```python
class FullAttestation(object):
    # proof that we have checked validators of this attestation are valid
    validators_shuffle: Map(commitee_index, list(validators_pubkeys))
    participants_pub_keys: List[BLSSignature, MAX_VALIDATORS_PER_COMMITTEE]
    signature: BLSSignature
```


# Calculation logic

## Shuffled commitees

Below are functions from ethereum spec

```python
def get_seed(state: BeaconState, epoch: Epoch, domain_type: DomainType) -> Bytes32:
    """
    Return the seed at ``epoch``.
    """
    mix = get_randao_mix(state, Epoch(epoch + EPOCHS_PER_HISTORICAL_VECTOR - MIN_SEED_LOOKAHEAD - 1)) 
    return hash(domain_type + uint_to_bytes(epoch) + mix)
```

```python 
def get_beacon_committee(state: BeaconState, slot: Slot, index: CommitteeIndex) -> Sequence[ValidatorIndex]:
    """
    Return the beacon committee at ``slot`` for ``index``.
    """
    epoch = compute_epoch_at_slot(slot)
    committees_per_slot = get_committee_count_per_slot(state, epoch)
    return compute_committee(
        indices=get_active_validator_indices(state, epoch),
        seed=get_seed(state, epoch, DOMAIN_BEACON_ATTESTER),
        index=(slot % SLOTS_PER_EPOCH) * committees_per_slot + index,
        count=committees_per_slot * SLOTS_PER_EPOCH,
    )
```

```python 
def get_randao_mix(state: BeaconState, epoch: Epoch) -> Bytes32:
    """
    Return the randao mix at a recent ``epoch``.
    """
    return state.randao_mixes[epoch % EPOCHS_PER_HISTORICAL_VECTOR]
```

```python
def get_committee_count_per_slot(state: BeaconState, epoch: Epoch) -> uint64:
    """
    Return the number of committees in each slot for the given ``epoch``.
    """
    return max(uint64(1), min(
        MAX_COMMITTEES_PER_SLOT,
        uint64(len(get_active_validator_indices(state, epoch))) // SLOTS_PER_EPOCH // TARGET_COMMITTEE_SIZE,
    ))
```