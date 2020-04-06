<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Ethereum 2.0 Phase 1 -- Shard Transition and Fraud Proofs](#ethereum-20-phase-1----shard-transition-and-fraud-proofs)
  - [Table of contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Fraud proofs](#fraud-proofs)
    - [Shard state transition function](#shard-state-transition-function)
    - [Verifying the proof](#verifying-the-proof)
  - [Honest committee member behavior](#honest-committee-member-behavior)
    - [Helper functions](#helper-functions)
    - [Make attestations](#make-attestations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Ethereum 2.0 Phase 1 -- Shard Transition and Fraud Proofs

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- TOC -->

 TODO

<!-- /TOC -->

## Introduction

This document describes the shard transition function and fraud proofs as part of Phase 1 of Ethereum 2.0.

## Fraud proofs

### Shard state transition function

```python
def shard_state_transition(beacon_state: BeaconState,
                           shard: Shard,
                           slot: Slot,
                           shard_state: ShardState,
                           beacon_parent_root: Root,
                           proposer_index: ValidatorIndex,
                           signed_block: SignedShardBlock,
                           validate_result: bool=True) -> None:
    # TODO: We will add something more substantive in phase 2

    # Verify the proposer_index and signature
    assert proposer_index == signed_block.message.proposer_index
    if validate_result:
        assert verify_shard_block_signature(beacon_state, signed_block)

    # Update shard state
    shard_state.slot = slot
    shard_state.latest_block_root = hash(
        hash_tree_root(shard_state) + hash_tree_root(beacon_parent_root) + hash_tree_root(signed_block.message.body)
    )
```

```python
def verify_shard_block_signature(beacon_state: BeaconState,
                                 signed_block: SignedShardBlock) -> bool:
    proposer = beacon_state.validators[signed_block.message.proposer_index]
    signing_root = compute_signing_root(signed_block.message, get_domain(beacon_state, DOMAIN_SHARD_PROPOSAL))
    return bls.Verify(proposer.pubkey, signing_root, signed_block.signature)
```

### Verifying the proof

TODO. The intent is to have a single universal fraud proof type, which contains the following parts:

1. An on-time attestation `attestation` on some shard `shard` signing a `transition: ShardTransition`
2. An index `offset_index` of a particular position to focus on
3. The `transition: ShardTransition` itself
4. The full body of the shard block `shard_block`
5. A Merkle proof to the `shard_states` in the parent block the attestation is referencing

Call the following function to verify the proof:

```python
def verify_fraud_proof(beacon_state: BeaconState,
                       attestation: Attestation,
                       offset_index: uint64,
                       transition: ShardTransition,
                       signed_block: SignedShardBlock,
                       subkey: BLSPubkey,
                       beacon_parent_block: BeaconBlock) -> bool:
    # 1. Check if `custody_bits[offset_index][j] != generate_custody_bit(subkey, block_contents)` for any `j`.
    shard = get_shard(beacon_state, attestation)
    slot = attestation.data.slot
    custody_bits = attestation.custody_bits_blocks
    for j in range(custody_bits[offset_index]):
        if custody_bits[offset_index][j] != generate_custody_bit(subkey, signed_block):
            return True

    # 2. Check if the shard state transition result is wrong between
    # `transition.shard_states[offset_index - 1]` to `transition.shard_states[offset_index]`.
    if offset_index == 0:
        shard_state = beacon_parent_block.shard_transitions[shard][-1]
    else:
        shard_state = transition.shard_states[offset_index - 1].copy()  # Not doing the actual state updates here.

    shard_state_transition(
        beacon_state=beacon_state,
        shard=shard,
        slot=slot,
        shard_state=shard_state,
        beacon_parent_root=hash_tree_root(beacon_parent_block),
        proposer_index=get_shard_proposer_index(beacon_state, slot, shard),
        signed_block=signed_block,
    )
    if shard_state.latest_block_root != transition.shard_states[offset_index].data:
        return True

    return False
```

```python
def generate_custody_bit(subkey: BLSPubkey, block: ShardBlock) -> bool:
    # TODO
    ...
```

## Honest committee member behavior

### Helper functions

```python
def get_winning_proposal(beacon_state: BeaconState, proposals: Sequence[SignedShardBlock]) -> SignedShardBlock:
    # TODO: Let `winning_proposal` be the proposal with the largest number of total attestationsfrom slots in
    # `state.shard_next_slots[shard]....slot-1` supporting it or any of its descendants, breaking ties by choosing
    # the first proposal locally seen. Do `proposals.append(winning_proposal)`.
    return proposals[-1]  # stub
```

```python
def get_empty_proposal(shard_state: ShardState, slot: Slot) -> SignedShardBlock:
    # TODO
    return SignedShardBlock()
```

```python
def is_empty_proposal(proposal: SignedShardBlock) -> bool:
    # TODO
    return proposal == SignedShardBlock()  # stub
```

```python
def compute_shard_data_roots(proposals: Sequence[SignedShardBlock]) -> Sequence[Root]:
    return [hash_tree_root(proposal.message.body) for proposal in proposals]
```

### Make attestations

Suppose you are a committee member on shard `shard` at slot `current_slot` and you have received shard blocks `shard_blocks`. Let `state` be the head beacon state you are building on, and let `QUARTER_PERIOD = SECONDS_PER_SLOT // 4`. `2 * QUARTER_PERIOD` seconds into slot `current_slot`, run `get_shard_transition(beacon_state, shard, shard_blocks)` to get `shard_transition`.

```python
def get_shard_transition(beacon_state: BeaconState,
                         shard: Shard,
                         shard_blocks: Sequence[SignedShardBlock]) -> ShardTransition:
    proposals, shard_states, shard_data_roots = get_shard_state_transition_result(beacon_state, shard, shard_blocks)
    start_slot = shard_states[0].slot

    shard_block_lengths = [len(proposal.message.body) for proposal in proposals]
    proposer_signature_aggregate = bls.Aggregate([proposal.signature for proposal in proposals])

    return ShardTransition(
        start_slot=start_slot,
        shard_block_lengths=shard_block_lengths,
        shard_data_roots=shard_data_roots,
        shard_states=shard_states,
        proposer_signature_aggregate=proposer_signature_aggregate,
    )
```

```python
def get_shard_state_transition_result(
    beacon_state: BeaconState,
    shard: Shard,
    shard_blocks: Sequence[SignedShardBlock]
) -> Tuple[Sequence[SignedShardBlock], Sequence[ShardState], Sequence[Root]]:
    proposals = []
    shard_states = []
    shard_state = beacon_state.shard_states[shard].copy()

    for slot in get_offset_slots(beacon_state, shard):
        choices = []
        beacon_parent_root = get_block_root_at_slot(beacon_state, get_previous_slot(beacon_state.slot))
        shard_blocks_at_slot = [block for block in shard_blocks if block.message.slot == slot]
        for block in shard_blocks_at_slot:
            temp_shard_state = shard_state.copy()  # Not doing the actual state updates here.
            # Try to apply state transition to temp_shard_state.
            try:
                shard_state_transition(
                    beacon_state=beacon_state,
                    shard=shard,
                    slot=slot,
                    shard_state=temp_shard_state,
                    beacon_parent_root=beacon_parent_root,
                    proposer_index=get_shard_proposer_index(
                        beacon_state,
                        slot,
                        shard
                    ),
                    signed_block=block,
                )
            except Exception:
                pass
            else:
                choices.append(block)

        if len(choices) == 0:
            proposals.append(get_empty_proposal(shard_state, slot))
        elif len(choices) == 1:
            proposals.append(choices[0])
        else:
            proposals.append(get_winning_proposal(beacon_state, choices))

        if not is_empty_proposal(proposals[-1]):
            # Apply state transition to shard_state.
            shard_state_transition(
                beacon_state=beacon_state,
                shard=shard,
                slot=slot,
                shard_state=shard_state,
                beacon_parent_root=beacon_parent_root,
                proposer_index=get_shard_proposer_index(
                    beacon_state,
                    slot,
                    shard
                ),
                signed_block=block,
            )

        shard_states.append(shard_state)

    shard_data_roots = compute_shard_data_roots(proposals)

    return proposals, shard_states, shard_data_roots
```