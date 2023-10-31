# ZkCasper on custom snark

## Description

The document below is based on [Polytope research](https://research.polytope.technology/zkcasper#18b08383b7cb4500ac1cfc12c405c0a9)


### KZG commitment to all validators in beacon state validators registry

Beacon state has record field named "validators", which is an append-only list of validators

```jsonld
 { 
    "pubkey": "0x933ad9491b62059dd065b560d256d8957a8c402cc6e8d8ee7290ae11e8f7329267a8811c397529dac52ae1342ba58c95",
    "withdrawal_credentials": "0x0100000000000000000000000d369bb49efa5100fd3b86a9f828c55da04d2d50",
    "effective_balance": "32000000000",
    "slashed": false,
    "activation_eligibility_epoch": "0",
    "activation_epoch": "0",
    "exit_epoch": "18446744073709551615",
    "withdrawable_epoch": "18446744073709551615"
},
```

We retreive this list from the finalized block of epoch N-4, consensus specs guarantees that validators, active at that point, will remain active in epoch N, which we will work with.


We take this list as SSZ compressed data, and take merkle-proof over finalazed block's state root.

Note: this is an append-only list, which track active/left/new validators:
* to mark validator as not active anymore, "exit_epoch" edited
* or "slashed" set to true
* or for new validators became active, "activation_epoch" becomes less then current epoch

Beacon chain calculates active validators set as follows: 

```python
def get_active_validator_indices(state: BeaconState, epoch: Epoch) -> Sequence[ValidatorIndex]:
    """
    Return the sequence of active validator indices at ``epoch``.
    """
    return [ValidatorIndex(i) for i, v in enumerate(state.validators) if is_active_validator(v, epoch)]
```

```python
def is_active_validator(validator: Validator, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is active.
    """
    return validator.activation_epoch <= epoch < validator.exit_epoch
```


### Approach

A. Verifier holds a commitment **C** to the list of public keys ${pk_{i}} \space \forall i \in T$, where **T** is public keys list.
Verifier holds bitlist of disabled validators **D**.

*ps. initial upload of 800+k of bitlist into blockchain worth of 100kb (800k bit) storage.*

B. Prover sends to the verifier:
1. set **A** of signature, aggregate of public keys and signed msg from each commitee:
A = ($\tilde{\sigma}_{i}$, $apk_{i}$, $m_{i}) \forall i \in [0, n-1)$
2. bitlist **B** of all participating public keys of all the individual validators.
3. proof $\pi_{apk}$ for participating public keys {$apk_{i}$}

C. The verifier performs 
1. B = B &oplus; D (&oplus; = xor) e.g. excludes disabled validators from list B (??)
2. $verify(C, \sum_{i = 0}^{n-1} apk_{i}, \pi_{apk}, B) \in \{1,0\}$ e.g. all public keys of participated validators are in registry T
3. Naively verifies aggregated signature over participated commitee members and message via pairing $e(g_{1}, \sum_{i=0}^{n-1} \tilde{\sigma}_{i}) = \sum_{i=0}^{n-1} e(apk_{i}, H_{1}(m_{i}))$


As a result, we have **verified that all attestations** with non-overlapping public keys **were signed by public keys, included in the validator set** of beacon chain state.

### Custom SNARK

[Polytope research](https://research.polytope.technology/zkcasper#18b08383b7cb4500ac1cfc12c405c0a9) suggests to use custom SNARK, described in Web3 Foundations "[Accountable Light Client Systems for PoS Blockchains](https://eprint.iacr.org/2022/1205.pdf)" paper.

Web3 Foundation research said proposed method **could be implemented by common used SNARK systems such as PLONK**, but they had created custom SNARK to improve efficiency.

While they achieved very fast proving time in their SNARKs implementation, this came at
the cost of **not using a general purpose SNARK protocol**, in turn leading to a more involved security
model and the necessity of additional security proofs.

Below are the high-level overview of custom SNARK properties:

1. custom SNARK is an instance of the commit-and-prove paradigm
2. it has kzg commitment as an input
3. it uses two-step compiler
    * PLONK Compiler - from Polynomial Protocols to SNARKs)
    * Mixed Vector and Commitments based NP Relations and Associated SNARKs

Overall, this custom SNARK can verify whether the constituent public keys in an aggregate BLS public key exist as a subset of a list of potential signers. Using this SNARK the verifier only needs to maintain a KZG commitment to the BLS public keys of all potential signers.

Full description and concrete math of custom SNARK can be found [here](https://eprint.iacr.org/2022/1205.pdf)

They also made Rust-based POC, built on the Arkworks library.

Benchmark results:
![](https://hackmd.io/_uploads/Hy1OfvCGa.png)

Here we can see that for $2^{20}$ validators (order of validators in beacon chain), prover time allows us to make proofs realtime, epoch by epoch (we have to prove consensus in time of two epochs, 12+ minutes)

