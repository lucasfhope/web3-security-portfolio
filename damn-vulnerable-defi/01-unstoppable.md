# Unstoppable

## Challenge Overview

The `UnstoppableVault` contract is an ERC4626-compatible vault holding DVT tokens and offering zero-fee flash loans. The `UnstoppableMonitor` contract can verify the flash loan function remains operational.

Starting with 10 DVT tokens, disable the vault’s ability to execute flash loans.

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

## Result

Transferring tokens directly to the vault breaks the asset-to-share invariant. The liveliness check by the `FlashLoanMonitor` will fail due to the denial-of-service on the vault’s flash loan functionality.