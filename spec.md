# Casper Final Spec

## What is consensus

paste from consensus chapter

## Goal

We want to create Light Clients in EVM blockhains. Light Client should be trustless updated with next justified checkpoint from consensus layer (beacon chain), verifying proof that validators with sum balance more than 2/3 of all active validators balance have agreed about next justified/finalized checkpoints.

It allows to use stored finalized checkpoints as a trusted source of execution layer state root.

## TLDR

1. Epoch N starts
2. Get finalized checkpoint header
4. Get state of finalized checkpoint
5. Get validators list from finalized state
6. Get active validators indexes
7. Calculate totalActiveBalance for set and save to DB
9. Get randao from end of epoch N-2
10. Shuffle active validators indexes
11. Calculate commitees memberships and save to DB
12.  Start listening to block attestations

for each attestation calculate balances:

1. get votedTotalBalance
3. save attestation and its balances to db

for each post-block check if we have enough balances voted for consensus:
1. group attestations by source/target
2. calculate sum(votedTotalBalances) for each group

if we have consensus, e.g. sum(votedTotalBalances) for attestations group with same source/target > 2/3 of totalActiveBalance (stored in DB):

Prove all consensus attestations:

for each attestation:
1. get attestation from db
2. get commitee validators from db
3. get voted validators from commitee and attestation.aggregation_bits
4. get multi-proof of these validators existense in finalizedHeader.state_root
5. build proof in proof-market

--- 

Attestation proof

tldr: 
* check all signers are validators from state
* check aggregated signature against attestation message and validators pub_keys
* calculate as output total balance of the voted validators

inputs:

1. attestation (source, target, slot, commitee_index, aggregated_signature)
2. commitee validators list
3. merkle multi proof of validators
4. finalizedHeader.state_root

public inputs:
1. source/target
3. slot/commitee_index

output: 
1. totalVotedBalance

proof logic:

input.source == attestation.source
input.target == attestation.target
input.commitee_index = attestation.commitee_index
input.slot = attestation.slot

create ssz.roots of validators list
merkleVerify(ssz.roots, merkle multi proof, state_root)

calculate sum(pub_keys)
bls.verify(attestation.data, attestation.signature, sum(pub_keys))

output totalVotedBalance = sum(validators.effectiveBalance)

---

Finalization proof

tldr:

* check all attestations have same source/target
* verify all attestation proofs
* verify validators set merkle proof agains finalizedHeader.state_root
* calculate total balance of all active validators in set
* verify sum(attestations totalVotedBalances) > 2/3 of total active set balance

inputs:
1. validators set list
2. validators merkle proof
3. attestations
4. attestation proofs

public inputs:
1. epochId
2. finalizedHeader.state_root
3. source/target

proof logic:

for each attestation
* attestation_proof.public_input.source = source
* attestation_proof.public_input.target = target

calculate totalAttestedBalance = sum(attestations.totalVotedBalance)

merkle.verify(ssz(validators set), validators merkle proof, finalizedHeader.state_root)

calculate total active validators balance = sum(active_validators.balances)

check totalAttestedBalance > 2/3 total active set balance

-- -- 

Then we call lightClient.update(final proof, source, target)




## Light Client

We create Light client in form of EVM smart contract deployed to the target blockchains.

### Light Client Storage

```solidity=
struct ExecutionHeader {
    execution_state_root: bytes32;
    execution_block_hash: bytes32;
    beacon_block_hash: bytes32;
}

struct BlockHeader {
    ExecutionHeader executionHeader;
    bytes32 beaconHeaderRoot;
}

struct Checkpoint {
    uint epochId;
    BlockHeader header;
}

struct Storage {
    Checkpoint public finalizedCheckpoint;
    /// @dev This is a mapping from an epochId to their justified checkpoints
    mapping(uint => Checkpoint) public justifiedCheckpoints;
}
```

### Light Client update function

```solidity=
struct UpdateData {
    BlockHeader source;
    BlockHeader target;
    
    Proof proof;
    /// @notice Validators set taken from a state of this epoch's checkpoint
    uint finalizedEpochId;
}

function update(UpdateData calldata update) external {
    // check if data taken from finalized block, known (hence verified before) by light client
    require(Storage.finalizedCheckpoint.epochId == update.finalizedEpochId);
    
    
    // verify final proof, revert if false
    Verifier.verify(update.source.block_hash, update.target.block_hash, Storage.finalizedCheckpoint.state_root, proof);
    
    // if proof is ok, append justified checkpoint for new epochId
    Storage.jusifiedCheckpoints[update.justifiedCheckpoint.epochId] = update.justifie
    
    // calculate finalized epoch from previously saved justified checkpoints 
    // @notice can we cover all the cases of casper here?
    
    calculateFinalizedEpoch(epochId);
}
```

### Calculate Finalized Epoch in light client

todo: describe all the cases and what light client has in Storage.justifiedCheckpoints mapping for this cases.

```solidity=
function calculateFinalizedEpoch(uint epochId) internal {
    // get previously justified checkpoints from storage
    // calculate epoch that can be finalized
    // update Storage.finalizedCheckpoint
}
```

### TBD

What do we really need to store in light client (on target chain)?

Telepathy light client stores:
1. uint The latest slot the light client has a finalized header for.
2. Maps from a slot to a beacon block header root.
3. Maps from a slot to the current finalized ethereum1 execution state root.

Todo: research and prove that we can trustlessly check validity of any execution layer's tx, event, log, contract storage, account balance with merkle proof against execution state root for any execution block id which is behind the finalized beacon checkpoint.

Todo: what would be real life inputs for bridges? 

## Beacon node

We use Nimbus beacon chain node to get following data from node state via api:

### finalizedBlockHeader

#### Request

```
GET /eth/v1/beacon/headers/{finalized_block_root}
```

where finalized_block_root is hash of block which is known by light client.

#### Response
```jsonld
{
  "execution_optimistic": false,
  "finalized": false,
  "data": {
    "root": "0x...",
    "canonical": true,
    "header": {
      "message": {
        "slot": "1",
        "proposer_index": "1",
        "parent_root": "0x...",
        "state_root": "0x...",
        "body_root": "0x..."
      },
      "signature": "0x..."
    }
  }
}
```

We will use state_root of this block (which is also stored in LightClient) to build and verify merkle proofs of state data parts.

### Full state

#### Request
```
GET /eth/v2/debug/beacon/states/{finalized_block_root}
```

#### Response
```jsonld=
{
    // fields omitted
    validators: [
         {
                "pubkey": "0x...",
                "withdrawal_credentials": "0x...",
                "effective_balance": "32000000000",
                "slashed": false,
                "activation_eligibility_epoch": "0",
                "activation_epoch": "0",
                "exit_epoch": "18446744073709551615",
                "withdrawable_epoch": "18446744073709551615"
            },
        // 800k+ validators objects
    ]
}
```

### Randao

To pre-calculate commitees members for epoch N, we have to use Randao value from the end of epoch N-2.

*todo: add link to related function in beacon spec*

We get randao from the full state after the last slot on epoch. 

*Todo: Is it a state of last state of epoch or next slot? Make research.*

## Relay backend

To spin up data processing, we build backend system, based on

* node.js/typescript
* redis as data storage
* bull as queues service
* nimbus beacon node

**todo: add diagram of relay architecture**

### Data processing

#### Listen to the end of epoch N
Beacon chain has a slots of 12 seconds each. We don't need to listen to some nodes events. We can be sure that node has state for slot after some timestamp passed.

#### Get finalized slot header
We store finalized slot in light client. Also we store finalized slot id in redis. In normal case we should already have this header stored.

#### Get epoch N-4 state
We get epoch N-4 state as json response of beacon node.

#### Get epoch N-2 post-state
Get RANDAO value from state.randao_mixes.

### Data pre-processing

Here we start to calculate data to get curcuit inputs.

####  Calculate commitees participants

First, we shuffle validators set. 
Then we calculate commitee lengths and split validators to 64 commitees.

64 commitees per slot is a constant.

```javascript=
// @returns shuffled indexes of full validatots set
function fullShuffle(randao, validators, epochId): Array<index: number> {
    // generate shuffle seed based on randao and epochId
}
```

We will pre-calculate commitees memberships, based on consensus spec compute_shuffled_index function.

Validators are divided among the committees in an epoch by the compute_committee() function.

First, we calculate active validators from state.validators:

todo: active validators filter folmula

Then, we get shuffled list of active validators:

```python=
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

```python=
def get_seed(state: BeaconState, epoch: Epoch, domain_type: DomainType) -> Bytes32:
    """
    Return the seed at ``epoch``.
    """
    mix = get_randao_mix(state, Epoch(epoch + EPOCHS_PER_HISTORICAL_VECTOR - MIN_SEED_LOOKAHEAD - 1))  # Avoid underflow
    return hash(domain_type + uint_to_bytes(epoch) + mix)
```

Note: full validators set shuffle implementation here:  [Java function](https://github.com/ConsenSys/teku/blob/04294427f2622c86326db68f3b88ed20d1e6cdc1/ethereum/spec/src/main/java/tech/pegasys/teku/spec/logic/common/helpers/MiscHelpers.java#L154 )

Then, we compute commitees for each slot:

```python=
def compute_committee(indices: Sequence[ValidatorIndex],
                      seed: Bytes32,
                      index: uint64,
                      count: uint64) -> Sequence[ValidatorIndex]:
    """
    Return the committee corresponding to ``indices``, ``seed``, ``index``, and committee ``count``.
    """
    start = (len(indices) * index) // count
    end = (len(indices) * uint64(index + 1)) // count
    return [indices[compute_shuffled_index(uint64(i), uint64(len(indices)), seed)] for i in range(start, end)]
```

```python=
def get_beacon_committee(state: BeaconState, slot: Slot, index: CommitteeIndex) -> Sequence[ValidatorIndex]:
    """
    Return the beacon committee at ``slot`` for ``index``.
    """
    epoch = compute_epoch_at_slot(slot)
    
    // always 64
    committees_per_slot = get_committee_count_per_slot(state, epoch)
    return compute_committee(
        indices=get_active_validator_indices(state, epoch),
        seed=get_seed(state, epoch, DOMAIN_BEACON_ATTESTER),
        index=(slot % SLOTS_PER_EPOCH) * committees_per_slot + index,
        count=committees_per_slot * SLOTS_PER_EPOCH,
    )
```


#### Get all attestations from 64 blocks

From slot 1 of epoch, we start getting blocks and from blocks we got attestations:

request: 

```
GET /eth/v2/beacon/blocks/{slot_id}
```

response: 
```json
{
    //...
    data: {
        // ...
         "attestations": [
          {
            "aggregation_bits": "0xffffffffbfffdffffffffffffffffffffffffffffffffffffffffffffffbffffffffffffffffffffffffffffffffffffffffffff01",
            "data": {
              "slot": "7517974",
              "index": "27",
              "beacon_block_root": "0x...",
              "source": {
                "epoch": "234935",
                "root": "0x..."
              },
              "target": {
                "epoch": "234936",
                "root": "0x..."
              }
            },
            "signature": "0x..."
          },
    }
}
```


#### Pre-calculate consensus

We use Postgres to save attestations from a recent block.

We for each attestation we calculate votedTotalBalance and totalCommiteeBalance of all commitee members.

Sum of all totalCommiteeBalances will be total active validators balance, used to determine if 2/3 of total voted balances leads us to the consensus. 

```javascript=
function getAttestationTotalBalances(commiteeIndex, slotId) {
    let votedTotalBalance = 0;
    let totalCommiteeBalance = 0;
    
    for (bit, index in attestation.aggregation_bits) {
        validatorIndex = 
    }
}
```


then we run function detecting if we have consensus already ot not yet:



```javascript=
function isConsensusAchieved(epochId) {
    
}
```

#### Get merkle-proofs of validators existence in validators set


Finally, we have following:

For each commitee:

* list of 400 validators objects from active_validators set
* list of multi-proof of inclusion of these validators into state over state root
* epochId
* commitee index


#### Build proofs for all of these attestations
inputs:
* attestation
    * source
    * target
    * signature
    * commitee index
    * pre-calculated pubkey = sum of voter's pubkeys
* list of validators
    * pubkey
    * balance
* multi merkle proof of validators existense in validators of finalized set

output: 
* sum of voters balances
* commiteeId/slotId commitment

#### Build final proof of finality

Inputs:

* 1500 proofs of attestations
* 1500 commiteeId/slotId commitments

Logic: 
Prove that for each attestation
* source/target the same
    * todo: the same or can be different?
* has slotId from target epoch
* commiteeIndexes/slotIds not overlapping
    * hence all validators signed these attestations are distinct by design
* attestation proof is valid
* sum of attestations balances more than 2/3 of total active balance

