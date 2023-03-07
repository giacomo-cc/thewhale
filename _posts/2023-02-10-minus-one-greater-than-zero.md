---
layout: post
title: "Sometimes -1 is greater than 0"
tags: solidity evm
---

Solidity offers the possibility for developers to write pseudo-assembly in ethereum smart contracts, this of course have huge impacts in terms of performance, gas costs _and readibility_, but sometimes it can introduce some unexpected bugs.

# yul

The assembly dialect is called [yul](https://docs.soliditylang.org/en/v0.8.17/yul.html) and looks like this:

```solidity
function readWithYul() public view returns (uint256 data) {
    assembly {
        data := sload(0)
    }
}
```

It provides a lot of asm instructions to interact with memory, storage, function calldata, etc.

When I was building a porting of [VyperProtocol](https://twitter.com/VyperProtocol) for EVM I was quite curious of this Solidity feature. Coming from a Solana rust-ecosystem this was one of the brand new features, so as soon as I had the opportunity I tried it out.

The result was quite rewarding.This is an extract from an _initialization_ method where we simply had to store on the ledger some external inputs (pretty common situation). Thanks to Yul I was able to save on storage multiple values with a single `sstore` call, this was conveniente to save some gas.

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

So far so good.

# bug explained

Some problem emerged modelling a digital option payoff formula with yul. For all of you with dusty familiarity with derivatives, a digital option contract pays out a fixed amount on the condition that a certain event happens, while paying 0 otherwise.

The first implementation with yul is the following (spoiler: its buggy, don't use in prod):

```solidity
function execute(
    uint256 payoffID,
    uint256 a,
    uint256 b
) external view returns (uint256 pnlLong, uint256 pnlShort) {
    DigitalData memory d = digitalData[payoffID];
    (int256 newSpot,) = d. oracleAdapter.getLatestPrice(d.oracleID);
    int256 strike_ = d. strike;
    bool isCall_ = d.isCall;

    // yul implementation
    assembly {
        let c_1 := and(isCall_, or(gt(newSpot, strike_), eq(newSpot, strike_)))
        let c_2 := and(not(isCall_), lt(newSpot, strike_))

        switch or(c_1, c_2)
        case 1 {
            pnlLong := b
            pnlShort := a
        }
        default {
            pnlLong := 0
            pnlShort := add(a, b)
        }
    }
}
```

Where the inputs:

- `uint256 payoffID` is the ID of the configuration inside the pool.
- `uint256 a` and `uint256 b` are the two deposited sizes, premium and collateral
  The outputs:
- `uint256 pnlLong` and `uint256 pnlShort` are the initial deposits split accordingly to the payoff outcome and the oracle price

At a first glance the method looks fine, but the bug was hiding there. Feel free to discover it on your own, otherwise keep reading.

# de-bug explained

In order to test the method I wrote a fuzzy test with foundry [foundry](https://github.com/foundry-rs/foundry). If you're not using foundry be worried is dramatically fast and provides a couple of outstanding features that Hardhat and Truffle lack. At the moment I'm using whenever is possible both Hardhat and Foundry together, will eventually write an article on the setup.

Following the foundry fuzzy test mentioned:

```solidity
function testExecute(int256 _strike, int256 _oraclePrice, bool _isCall, uint128 _longSide, uint128 _shortSide) public {
    vm.assume(_longSide > 0);
    vm.assume(_shortSide > 0);

    digitalPayoffPool.createDigitalPayoff(_strike, _isCall, chainlinkAdapter, 0);

    mockChainlinkAggregator.updateAnswer(_oraclePrice);
    (uint256 pnlLong, uint256 pnlShort) = digitalPayoffPool.execute(0, _longSide, _shortSide);

    assertEq(pnlLong + pnlShort, uint256(_longSide) + uint256(_shortSide));
    if ((_isCall && _oraclePrice >= _strike) || (!_isCall && _oraclePrice < _strike)) {
        assertEq(pnlLong, _shortSide);
        assertEq(pnlShort, _longSide);
    } else {
        assertEq(pnlLong, 0);
        assertEq(pnlShort, uint256(_longSide) + uint256(_shortSide));
    }
}
```

As you can see I was providing a bunch of inputs that froundry fuzzs for me. The problem was discovered with these inputs: `[-1, 0, true, 1, 1]`.

After a couple of research the error was clear: in yul the only type availble is `u256`.

> Currently, there is only one specified dialect of Yul. This dialect uses the EVM opcodes as builtin functions (see below) and defines only the type u256, which is the native 256-bit type of the EVM. [link](https://docs.soliditylang.org/en/v0.8.17/yul.html#motivation-and-high-level-description)

So with both inputs as `u256`: -1 > 0 of course.

Following the same `execute` method as before in plain solidity, surely less gas efficient (but definitely more readble) and primarly without the previous bug:

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
