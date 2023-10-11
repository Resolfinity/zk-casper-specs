# Casper on zkllvm and Nil proof market

To get rid  reorgs issues, we are listening for new slots with some reasonable delay, e.g. 3-4 slots.

## High-level architecture

* beacon node with rpc api
* relay server
    * listens for new slots
    * retreives data from beacon node
    * prepares data as circuit inputs
    * sends proff-building orders to Nil proof market
    * gets proofs from market
    * sends proofs to evm smart contracts
* nil proof market
* evm smart contracts
    * deployed on target blockchains
    * verify proofs onchain
    * track source chain's finalized slots hashes
    * consist
        * nil verifier contract
        * custom gates

## What we have to prove

### For new epoch
* Get active validator set
* Calculate compute_shuffled_index

### For every new slot 

* Select attestations with majority of source/target/head/aggregation_bits
    * skip minor attestations at least for now to simplify logic
* For each attestation
    * get participated validators indexes
    * get participated validators balances
    * get participated validators public keys
    * check if aggregated signature is correct
    * generate proof that
        * attestation of commitee N 
        * has correct signature
        * and aggregated balance B
        * public inputs:
            * slot
            * balance
            * commitee index
            * target/source
* For new epoch
    * Prove that proofs generated before, with the same target/source, have more than 2/3 of total_active_balance
    * Public inputs
        * finalized slot id
        * finalized block hash
        * finalized eth1 block, state root, ...