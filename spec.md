# Casper Final Spec

## What is consensus

paste from consensus chapter

## Goal

We want to create Light Clients in EVM blockhains. Light Client should be trustless updated with next justified checkpoint from consensus layer (beacon chain), verifying proof that validators with sum balance more than 2/3 of all active validators balance have agreed about next justified/finalized checkpoints.

It allows to use stored finalized checkpoints as a trusted source of execution layer state root.


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

### Data requests

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

*todo: describe why 64 commitees. do we need to calculate commitees count on the fly? links to spec functions if yes*

```javascript=
// @returns shuffled indexes of full validatots set
function fullShuffle(randao, validators, epochId): Array<index: number> {
    // generate shuffle seed based on randao and epochId
}
```

We will pre-calculate commitees memberships, based on consensus spec compute_shuffled_index function.

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




#### Get merkle-proofs of validators existence in validators set

Each commitee consists of 400+ validators plus-minus one.

*todo: function that calculates commitee length
how exactly commitees formed from shuffled indexes*


Finally, we have following structs:

For each commitee:

* list of 400 validators objects
* list of multi-proof of inclusion of these validators into state over state root
* epochId
* commitee index

#### Get all attestations from 64 blocks

#### Pre-calculate and select attestations that give us majority

We select around 1500 attestations with same source and targets which have supermajority.

#### Build proofs for all of these attestations
inputs:
* attestation
    * source
    * target
    * signature
    * pre-calculated pubkey = sum of voter's pubkeys
* list of validators
    * pubkey
    * balance
* multi merkle proof of validators existense in validators of finalized set

output: sum of voters balances

#### Build final proof of finality

Inputs:

* 