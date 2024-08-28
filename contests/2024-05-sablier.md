# Sablier
## Contest Summary
Code under review: [2024-04-sablier](https://github.com/Cyfrin/2024-05-Sablier)

Contest Page: [sablier-contest](https://codehawks.cyfrin.io/c/2024-05-Sablier/results?lt=contest&page=1&sc=reward&sj=reward&t=report)

Placement: #20/42

# Findings

# [M-01] `SablierV2MerkleLockupFactory` is suspicious of the reorg attacks.

## Summary
Creating `SablierV2Merkle` contracts can be exploitable in the case of a blockchain reorg.

## Vulnerability Details
Sablier is deployed on all EVM compatible chains and among them there are some which are suspicious of the reorg attack like Polygon. In this particular chain, we can see transcations going for 1.5 minutes but then because of the reorg to be executed in different order or be discarded. Don't even mention [5 minutes](https://protos.com/polygon-hit-by-157-block-reorg-despite-hard-fork-to-reduce-reorgs/) attacks. We can fairly assume that in this time, a user can create his merkle airstream by calling `createMerkleLL` function and then transfer to this address the `aggregateAmount` that is needed. But in the case of a reorg, a malicious user can front run the deployment of the merkle airstream and create it in the same address since `create1` is used. Also, there is a possibility that a `recipient` has created his airstream in the previous contract (before reorg) so he will be able to double receive his airstream (after the reorg too). All in all, a reorg situation together with an acting of a malicious user can lead to loss of funds either for the `creator` or for the `Sablier`.

## Tools Used
Manual review

## Recommendations
Consider implementing the transfer of `aggregateAmount` in the same transaction with the deployment of the merkle airstream.