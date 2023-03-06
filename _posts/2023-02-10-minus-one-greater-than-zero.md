---
layout: post
title: "Sometimes -1 is greater than 0"
categories: solidity evm
tags: solidity evm
---

Solidity offers the possibility for developers to write pseudo-assembly in smart contract, this of course have huge impacts in terms of performance, gas costs and readibility. The dialect is called [yul](https://docs.soliditylang.org/en/v0.8.17/yul.html) and looks like this:

```solidity
function readWithYul() public view returns (uint256 data) {
    assembly {
        data := sload(0)
    }
}
```

When I was building a porting of VyperProtocol for EVM I was quite curious of this Solidity feature, missing in the Solana ecosystem where I had experience, so I tried it out. The result was quite good, for example this is an extract from an `initialization` method where thanks to Yul I was able to perform a single `sstore` to write on storage multiple values.

```solidity
function createTrade(
    IERC20 _collateral,
    IPayoffPoolPlugin _payoffPool,
    uint256 _payoffID,
    uint88 _depositEnd,
    uint88 _settleStart,
    uint256 _longRequiredAmount,
    uint256 _shortRequiredAmount
) public whenNotPaused {
    require(_longRequiredAmount + _shortRequiredAmount > 0, "invalid required amounts");

    uint256 tradeID = nextIdx++;

    Trade storage t = trades[tradeID];
    bytes32 slot_0 = (bytes32(bytes11(_depositEnd)) >> 8) | (bytes32(bytes20(address(_collateral))) >> 96);
    bytes32 slot_1 = (bytes32(bytes11(_settleStart)) >> 8) | (bytes32(bytes20(address(_payoffPool))) >> 96);
    assembly {
        sstore(t.slot, slot_0)
        sstore(add(t.slot, 1), slot_1)
        sstore(add(t.slot, 2), _payoffID)
        sstore(add(t.slot, 3), _longRequiredAmount)
        sstore(add(t.slot, 4), _shortRequiredAmount)
    }
    emit TradeCreated(tradeID);
}
```

So far all good.

The problems arises modelling a digital option storing the payoff formula in yul, here the first implementation (spoiler its buggy):

```solidity
function execute (uint256 payoffID, uint256 a, uint256 b) external view returns (uint256 pnlLong, uint256 pnlShort)
{
    DigitalData memory d = digitalData[payoffID];
    (int256 newSpot,) = d. oracleAdapter. getLatestPrice(d.oracleID);
    int256 strike_ = d. strike;
    bool isCall_ = d.isCall;
    // yul implementation
    assembly {
        let c_1 := and(isCall_, or(gt(newSpot, strike_), eq(newSpot, strike_)))
        let c_2 := and(not(isCall_), lt(newSpot, strike_))

        switch or(c_1, c_2)
        case 1 {
            pniLong := b
            pnlShort := a
        }
        default {
            pniLong := 0
            pnlShort := add(a, b)
        }
    }
}
```

At the first glance the the code looks fine, the bug was discovered during some fuzzy testing with [foundry](https://github.com/foundry-rs/foundry). In yul the only type availble is `u256`:

> Currently, there is only one specified dialect of Yul. This dialect uses the EVM opcodes as builtin functions (see below) and defines only the type u256, which is the native 256-bit type of the EVM. Because of that, we will not provide types in the examples below. ([ref](https://docs.soliditylang.org/en/v0.8.17/yul.html#motivation-and-high-level-description))

So interpreting both inputs as `u256` -1 > 0 of course.

Following the same `execute` method as before in plain solidity, without the previous bug:

```solidity
function execute(uint256 payoffID, uint256 a, uint256 b)
    external
    view
    returns (uint256 pnlLong, uint256 pnlShort)
{
    DigitalData memory d = digitalData[payoffID];
    (int256 newSpot,) = d.oracleAdapter.getLatestPrice(d.oracleID);

    if ((d.isCall && newSpot >= d.strike) || (!d.isCall && newSpot < d.strike)) {
        pnlLong = b;
        pnlShort = a;
    } else {
        pnlLong = 0;
        pnlShort = a + b;
    }
}
```
