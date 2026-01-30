---
title: "Deep Dive into Uniswap V1: The DeFi Foundation of the AMM Era"
date: 2025-10-19 16:21:05
math: true
tags:
  - Uniswap
  - AMM
  - Solidity
  - ERC20
  - Liquidity Pool
categories:
  - DeFi
  - Blockchain
  - Smart Contracts
---

![uniswap](Deep-Dive-into-Uniswap-V1/uniswap.jpg)

#### Introduction: Farewell to the Order Book

In the blockchain world, decentralized exchanges (DEXs) have always been a challenge. Most traditional exchanges, whether centralized or not, rely on an "Order Book" to match buyers and sellers. However, executing this on-chain is not only slow but also extremely gas-intensive.

The birth of Uniswap V1 fundamentally changed this game. Instead of using an order book, it introduced the "Automated Market Maker" (AMM) mechanism, holding a series of liquidity reserves. All trades are executed directly against these reserves (liquidity pools), making it highly efficient.

At the core of all this is one simple and elegant formula: **$x * y = k$** (the Constant Product Market Maker mechanism).

#### Core Architecture of Uniswap V1

##### 1. Factory Contract: Creating Exchanges for Everything

A key role in Uniswap V1's architecture is the Factory contract (`uniswap_factory.vy`). This contract serves as both a "Factory" and a "Registry."

Its core function is `createExchange()`. Anyone can call this function to deploy a brand new, independent "Exchange Contract" for any ERC20 token that doesn't have one yet.

The Factory contract records all created tokens and exchange addresses, providing query functions like `getExchange()` (look up exchange address by token address) and `getToken()` (look up token address by exchange address).

**Important Note:** The Factory contract does not perform any checks on the token itself when creating an exchange (e.g., verifying its legitimacy or value). It only ensures that each ERC20 token can have only one corresponding exchange contract. Therefore, users must conduct their own due diligence and only interact with tokens they trust.

##### 2. Gas Efficiency: An Unparalleled Advantage

Uniswap V1's design places extreme emphasis on gas efficiency. According to the whitepaper benchmarks:

- ETH to ERC20 trades consume nearly 10x less gas than Bancor.
- ERC20 to ERC20 trades are also more efficient than 0x.
- Compared to on-chain order book exchanges like EtherDelta or IDEX, gas fees are also significantly lower.

#### Trading Mechanism: How Does x\*y=k Work?

In Uniswap V1, each exchange contract is bound to only "one" ERC20 token and holds a liquidity pool of both ETH and that ERC20 token.

The exchange rate (price) is not determined by orders, but by the "relative size" of the two assets in the pool.
The mechanism always maintains the invariant $eth\_pool * token\_pool = k$. This $k$ value only changes when liquidity is added or removed.

##### 1. ETH $\leftrightarrow$ ERC20 Trading

**Example: Swapping ETH for OMG (ETH $\rightarrow$ Token):**

Assume a liquidity pool was initially funded by a Liquidity Provider (LP) with 10 ETH and 500 OMG.

1. **Initial State:**

   - `ETH_pool` = 10
   - `OMG_pool` = 500
   - `Invariant (k)` = $10 * 500 = 5000$

2. **Trade Execution:**

   - A buyer sends **1 ETH** to the contract
   - The contract first collects a 0.3% fee (*0.25% used in this example*), which is 0.0025 ETH
   - The remaining 0.9975 ETH is added to the ETH pool
   - `ETH_pool` (new) = $10 + 0.9975 = 10.9975$

3. **Calculating the Swap:**

   - To maintain $k=5000$, the new `OMG_pool` must be:
   - `OMG_pool` (new) = $k / ETH\_pool(\text{new}) = 5000 / 10.9975 = 454.65$ OMG
   - OMG received by buyer = `OMG_pool` (old) - `OMG_pool` (new) = $500 - 454.65 = 45.35$ OMG

4. **Fee Distribution:**

   - After the trade completes, the 0.0025 ETH fee collected earlier is "added back" to the liquidity pool
   - `ETH_pool` (final) = $10.9975 + 0.0025 = 11$
   - `OMG_pool` (final) = 454.65
   - **New Invariant (k)** = $11 * 454.65 = 5001.15$

Notice that $k$ has slightly increased! This is the source of profit for LPs (Liquidity Providers).

**The reverse Token $\rightarrow$ ETH trade** (`tokenToEthSwap`) works the same way: the token pool increases, the ETH pool decreases proportionally, and the price adjusts accordingly.

##### 2. ERC20 $\leftrightarrow$ ERC20 Trading

Uniswap V1 does not support "direct" swaps between ERC20 tokens. However, since ETH is the universal trading pair for all ERC20 tokens, ETH can be used as an intermediary.

For example, to swap OMG for KNC:

1. The buyer calls `tokenToTokenSwap()` on the **OMG exchange**
2. The OMG exchange swaps the buyer's OMG for ETH
3. The OMG exchange **does not** return the ETH to the buyer; instead, it takes this ETH and calls the `ethToTokenTransfer()` function on the **KNC exchange**
4. The KNC exchange receives the ETH, swaps it for KNC, and sends the KNC directly to the **original buyer's** address

This entire process completes within a single transaction, so from the user's perspective, it feels like they directly swapped OMG for KNC.

#### The World of Liquidity Providers (LPs)

Uniswap's liquidity pools don't appear out of thin air—they rely on "Liquidity Providers" (LPs) to contribute funds.

##### 1. Adding and Removing Liquidity

- **Adding Liquidity:**
  - Anyone can call `addLiquidity()`
  - When providing liquidity, you must deposit "equivalent value" of both ETH and ERC20 tokens
  - The first LP is responsible for setting the pool's initial exchange rate. If the rate is set unreasonably, arbitrageurs will immediately step in to bring the price back to market equilibrium.
- **Liquidity Tokens (LP Tokens):**
  - When LPs deposit funds, the contract mints "Liquidity Tokens" (LP Tokens) for them.
  - LP Tokens are themselves ERC20 tokens, representing the LP's share of the total reserves.
- **Removing Liquidity:**
  - LPs can burn their LP Tokens at any time.
  - The contract will return the corresponding share of ETH and ERC20 tokens from the pool based on the proportion of LP Tokens burned.

##### 2. LP Earnings: Trading Fees

Why would LPs provide liquidity? Because they can share in the trading fees.

- **ETH $\leftrightarrow$ ERC20 trades:** Pay a 0.3% fee (ETH pays in ETH, ERC20 pays in ERC20)
- **ERC20 $\leftrightarrow$ ERC20 trades:** Fees are collected twice (once at the input exchange, once at the output exchange), totaling approximately 0.5991%

These fees are **immediately deposited into the liquidity reserves**. Since total reserves increase but LP Token supply remains the same, the value of each LP Token unit increases. This is the LP's profit.

##### 3. LP Risk: Impermanent Loss

- **Impermanent Loss (IL):** Refers to the value loss that occurs when the price ratio of tokens you deposited into the pool changes compared to when you deposited them.
- **Example:**
  - You deposit 10 ETH and 1000 TokenA (at this time, 1 ETH = 100 TokenA)
  - At this point, $k = 10 * 1000 = 10000$
  - Some time later, the market price becomes 1 ETH = 200 TokenA
  - Arbitrageurs will step in until the pool ratio also becomes 1:200. At this point, the pool will contain 7.071 ETH and 1414.21 TokenA (because $x*y=10000$ and $x=200y$)
  - **If you withdraw now:** You get 7.071 ETH and 1414.21 TokenA. Valued in TokenA, total value is $(7.071 * 200) + 1414.21 = 2828.41$ TokenA
  - **If you had never deposited (HODL):** You would have 10 ETH and 1000 TokenA. Valued in TokenA, total value is $(10 * 200) + 1000 = 3000$ TokenA
  - **Loss:** $3000 - 2828.41 = 171.59$ TokenA—this is the impermanent loss

The trading fees earned by LPs essentially serve as compensation for bearing this price volatility risk (impermanent loss).

#### Conclusion

Uniswap V1, with its minimalist design, excellent gas efficiency, and revolutionary AMM concept, laid an indispensable foundation for the DeFi world. While it has some limitations (such as requiring ETH as an intermediary, impermanent loss), it also charted the evolutionary path for later V2 and V3 versions.
