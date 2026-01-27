# Unstoppable

## Challenge Overview

**Unstoppable** features an ERC4626-compatible vault holding 1,000,000 DVT tokens and offering zero-fee flash loans during a beta period. A monitoring contract tracks the availability of the flash loan functionality.

The objective is to stop the vault’s ability to issue flash loans with 10 DVT tokens.

## Protocol Summary

The system is centered around the `UnstoppableVault` contract, a tokenized vault that accepts DVT tokens in exchange for vault shares. The contract implements a zero-fee flash loan of DVT as long as funds are returned in the same transaction. 

The protocol includes an `UnstoppableMonitor` contract to verify the vault’s flash loan function remains operational. 

## Vulnerability Analysis

Before issuing a flash loan, `UnstoppableVault` enforces a check of the vault’s share supply against its asset balance:

```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

Normally, all assets enter the vault exclusively through `deposit` and no fees are accrued, so the check will pass.

However, tokens can be transferred directly to the vault without minting shares. This will imbalance the vault's assets and shares, resulting in `flashLoan` always reverting. This is a denial of service.

## Exploit Strategy

The attacker transfers DVT directly to the vault contract. This will break the asset/share balance invariant and disable the `flashLoan` function.

## Proof of Concept

```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

**Challenge solved** — flash loan liveness check fails.

<img src="./img/unstoppable-pass.png" width="800">






