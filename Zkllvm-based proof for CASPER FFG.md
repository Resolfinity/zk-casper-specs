# Zkllvm-based proof for CASPER FFG

## Description

We have to proof, that for given eth1 block with hash H

1. This block is a part of slot S in epoch  N
3. Supermajority of validators by stake have finalized this epoch

### This block is a part of slot S in epoch N

### Supermajority of validators by stake have finalized this epoch

* Process attestations from all commitees
* For every commitee
    * validate aggredated signature against aggregated bls keys
    * get validators list of committee
    * get aggregated attestation balance
* Calculate winning source with supermajority of attestations
    * *todo: define rules to know if source epoch has been finalized*
    * *some attestations could overlay others etc.*
    * 