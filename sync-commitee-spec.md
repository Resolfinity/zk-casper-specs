# Sync-commitee spec

## Circuits

### Step (Aggregate BLS12-381 Signature Verification)

Verifying the sync committees signatures involves computing 512 elliptic curve adds over the BLS12-381 curve to get the sync committee aggregate public key, and a single elliptic curve pairing for the final BLS signature check.

We also do a few other checks in the circuit, such as validating the Merkle proof for the Ethereum execution state root and checking the finality proof for the finalized block header.

#### inputs
* @input  attested{HeaderRoot,Slot,ProposerIndex,ParentRoot,StateRoot,BodyRoot}
                                  The header attested to by the sync committee and all associated fields.
 * @input  finalized{HeaderRoot,Slot,ProposerIndex,ParentRoot,StateRoot,BodyRoot}
                                  The finalized header committed to inside the attestedHeader.
 * @input  pubkeysX               X-coordinate of the public keys of the sync committee in bigint form.
 * @input  pubkeysY               Y-coordinate of the public keys of the sync committee in bigint form.
 * @input  aggregationBits        Bitmap indicating which validators have signed
 * @input  signature              An aggregated signature over signingRoot
 * @input  domain                 sha256(forkVersion, genesisValidatorsRoot)
 * @input  signingRoot            sha256(attestedHeaderRoot, domain)
 * @input  participation          sum(aggregationBits)
 * @input  syncCommitteePoseidon  A commitment to the sync committee pubkeys from rotate.circom.
 * @input  finalityBranch         A Merkle proof for finalizedHeader
 * @input  executionStateRoot     The eth1 state root inside finalizedHeader
 * @input  executionStateBranch   A Merkle proof for executionStateRoot
 * @input  publicInputsRoot       A commitment to all "public inputs"

publicInputsRoot - commitment of all inputs, sent to smart contract instead of all inputs, e.g. only one succinct input sent to the contract as public input.

#### step circuit flow

1. proof of publicInputsRoot is composed from all inputs
2. VALIDATE BEACON CHAIN DATA AGAINST SIGNING ROOT
3. VERIFY SYNC COMMITTEE SIGNATURE AND COMPUTE PARTICIPATION
4. VERIFY FINALITY PROOF
5. VERIFY EXECUTION STATE PROOF

### Rotate 

#### inputs
 * @input  pubkeysBytes             The sync committee pubkeys in bytes
 * @input  aggregatePubkeyBytesX    The aggregate sync committee pubkey in bytes
 * @input  pubkeysBigIntX           The sync committee pubkeys in bigint form, X coordinate.
 * @input  pubkeysBigIntY           The sync committee pubkeys in bigint form, Y coordinate.
 * @input  syncCommitteeSSZ         A SSZ commitment to the sync committee
 * @input  syncCommitteeBranch      A Merkle proof for the sync committee against finalizedHeader
 * @input  syncCommitteePoseidon    A Poseidon commitment of the sync committee
 * @input  finalized{HeaderRoot,Slot,ProposerIndex,ParentRoot,StateRoot,BodyRoot}
                                    The finalized header and all associated fields.

#### rotate circuit flow
1. VALIDATE FINALIZED HEADER AGAINST FINALIZED HEADER ROOT
2. CHECK SYNC COMMITTEE SSZ PROOF
3. VERIFY PUBKEY BIGINTS ARE NOT ILL-FORMED ??
4. VERIFY BYTE AND BIG INT REPRESENTATION OF G1 POINTS MATCH 
5. VERIFY THAT THE WITNESSED Y-COORDINATES MAKE THE PUBKEYS LAY ON THE CURVE
6. VERIFY THAT THE WITNESSESED Y-COORDINATE HAS THE CORRECT SIGN
7. VERIFY THE SSZ ROOT OF THE SYNC COMMITTEE 
8. VERIFY THE POSEIDON ROOT OF THE SYNC COMMITTEE 

#### proof building flow

- Prepare input data
    1. get current sync committee validators [indexes]
    2. get all validators [pubkeys]
    3. calculate aggregated pubkey
    4. calculate pubkeysBigIntX, pubkeysBigIntY
    5. get syncCommitteeSSZ ?
    6. get sync commitee merkle proof against finalized header
    7. calculate poseidon commitment of the sync commitee
    8. get finalized header
        - header root
        - slot
        - proposer index
        - parent root
        - state root
        - body root 

- send data above as inputs to the proof market
    poseidon is public input
    
- get proof from market
- call sc's rotate function with data
    -  ``` 
        // LightClientStep
        uint256 attestedSlot; // slot id, number
        uint256 finalizedSlot; // slot id, number
        uint256 participation // how many validators
        bytes32 finalizedHeaderRoot; 
        bytes32 executionStateRoot;
        Groth16Proof proof;
        
        // LightClientRotate
        LightClientStep step (above)
        bytes32 syncCommitteeSSZ;
        bytes32 syncCommitteePoseidon;
        Groth16Proof proof;

#### sc rotate function (Sets the sync committee for the next sync committeee period)
    - processStep function with LightClientStep
    - increments sync committee period
    - verifies rotate proof with inputs
        - syncCommitteeSSZ
        - finalizedHeaderRoot
        - syncCommitteePoseidon
    - writes syncCommitteePoseidon for new period
        

#### sc processStep function

*LightClientStep as input, verifies that the header has enough signatures for finality*

calculates commiteePeriod from attestedSlot 
(slot / SLOTS_PER_PERIOD)

checks if attested slot has syncCommitteePoseidon in storage, e.g. previous rotate function has been processed before


verifies step proof

#### sc step function
LightClientStep as input

processStep - see above

set slot storage
* update.finalizedSlot
* update.finalizedHeaderRoot
* update.executionStateRoot

set 
head = finalizedSlot
headers[slot] = finalizedHeaderRoot;
executionStateRoots[slot] = executionStateRoot;





    



