---
layout: post
title: "welcome chantilly"
tags: evm chantilly
---

We just released open source an internal dev tool we developed at [Vyper Labs](https://github.com/vyper-protocol), its name is chantilly and is now [open source](https://github.com/vyper-protocol/chantilly).

Keep reading if you'd like to interact with any EVM smart contract, on any EVM-compatible chain without writing a single line of code.

Chantilly is sweet tool that allows developers to interact with a deployed smart contracts just providing the minimum ingredients required:

1. an rpc endpoint
2. an address
3. the contract ABI.

That's it, no more info are required, and most important thing: nwe don't need to write a line of code.
The UI will inject automatically the rpc endpoint, creating an old school `JsonRpc` client under the hood. FYI: we're not using ethers but the brand new [viem_sh](https://twitter.com/wagmi_sh/status/1632814443972395010).

Once the ABI is provided the UI is dynamically created allowing users to query the chain automatically encoding and decoding function inputs and outputs. The cool thing is that this can be used against any chain, including... localhost:8545. In this way developers can test in seconds smart contracts, with a 10x cut in productivity. Coming soon in the docs [Hardhat](https://twitter.com/HardhatHQ) and foundry integration tutorial.

In the UI we already provided some ready to use examples, including: [uniswap v2](https://app.uniswap.org/#/swap), [cryptopunk](https://cryptopunks.app/), BoredApeYC, $USDT and more..

The project is completely open source and available on github, currently only view functions are supported but the full ABI integration is on the roadmap and will be soon released.
