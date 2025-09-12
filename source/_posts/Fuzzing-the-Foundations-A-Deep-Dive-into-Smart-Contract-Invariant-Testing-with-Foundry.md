---
title: Fuzzing the Foundations: A Deep Dive into Smart Contract Invariant Testing with Foundry
date: 2025-09-12 18:00:00
tags:
- Fuzz Testing
- Invariant Testing
- Foundry
- Smart Contract Security
- Solidity
- Web3
categories:
- Web3 Security
---

### Beyond Unit Tests: An Introduction to Fuzz & Invariant Testing for Smart Contracts

After completing a comprehensive suite of unit tests, many developers feel confident in their project’s security. However, traditional unit tests are deterministic; they only verify scenarios you have already anticipated. The true test of a smart contract lies in its ability to withstand unexpected, malicious, and random interactions. This is where **fuzz testing** becomes an indispensable tool.

This article will introduce the fundamentals of fuzz testing, explain the crucial difference between stateless and stateful (invariant) fuzzing, and demonstrate why it is so vital for projects handling significant financial value, like stablecoin protocols.

![Fuzz_Testing_In_Foundry](Fuzzing-the-Foundations-A-Deep-Dive-into-Smart-Contract-Invariant-Testing-with-Foundry/Blockchain_security.jpg)

---

### The Core Concept of Fuzz & Invariant Testing

Fuzz testing is a dynamic and powerful testing methodology that aims to discover vulnerabilities you never thought of. Unlike traditional tests, which use known inputs to check for a predictable output, fuzzing feeds your contract with vast amounts of random data to see if it breaks.

It is often called **Invariant Testing** because its core purpose is to validate the contract's invariants—the properties that must hold true under any circumstances. Instead of testing a single function, the fuzzing engine continuously sends random transactions to the contract and checks if these invariants are ever violated.

**How It Works:**

1. **Define Invariants:** You must first define what an "invariant" is for your system. These are critical, never-changing properties. For example:
   - In a lending protocol, the total collateral value must always be greater than or equal to the total debt value.
   - In an ERC20 token, the `totalSupply` must always equal the sum of all user balances.
   - In an NFT contract, there can only ever be one owner for each unique `tokenId`.
2. **Randomized Input:** Testing frameworks like Foundry automatically generate random parameters for your contract's functions. For instance, it might randomly select a function like `mint`, `transfer`, or `burn`, and then generate random values for the `_to` address and `_amount`.
3. **Continuous Testing:** The framework then executes a massive, continuous sequence of these random transactions. After each transaction, it automatically calls your predefined invariant functions to check if they still hold true.
4. **Reporting a Failure:** If an invariant is ever broken (e.g., `totalSupply` no longer matches the sum of balances), the test will immediately halt. The framework will then provide a complete, reproducible transaction sequence that led to the failure. This sequence is a golden ticket for developers, as it pinpoints the exact series of random inputs that revealed the bug.

---

### Stateless vs. Stateful Fuzz Testing

It's important to distinguish between two types of fuzzing that Foundry handles differently:

#### **Stateless Fuzz Testing**

A stateless fuzz test runs each time with a clean, independent execution environment. The state of a previous test is discarded before the next one begins. This method is ideal for testing a single function's behavior and its boundary conditions, such as checking for overflow errors or unexpected reverts.

- **Goal:** Test the behavior of a single function in isolation.
- **State:** The contract state is reset for every run.
- **Example:** A function `testFuzz_add(uint256 a, uint256 b)` checks if `a + b` behaves as expected. The test runs many times, each time with random `a` and `b` values, but it doesn't care about the state left by the previous run.

#### **Stateful Fuzz Testing (Invariant Testing)**

Stateful fuzz testing, or invariant testing, is a more powerful approach that maintains the contract's state across multiple random transactions. It simulates a real-world, chaotic environment where a series of seemingly harmless actions can lead to a catastrophic failure.

- **Goal:** Test the contract's core properties across a sequence of complex transactions.
- **State:** The contract's state persists from one test run to the next.
- **Example:** A test might simulate a user `deposit`, followed by another user's `withdraw`, and then a malicious user's `borrow`. After each step, it checks the invariant `totalCollateralValue >= totalDebtValue` to see if the protocol remains solvent.

In Foundry, you write `stateless` fuzz tests with function names like `testFuzz_...`, while `stateful` invariant tests use functions starting with `invariant_...`.

---

### A Practical Example with Foundry

Let's use Foundry to demonstrate how a stateful fuzz test can find a subtle bug in a simple token contract.

#### 1. The Vulnerable Contract (`MyToken.sol`)

This contract has a critical bug in its `burn` function:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    // A variable with an intentional bug.
    uint256 public extraSupply;

    constructor() ERC20("MyToken", "MT") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }

    // The bug is here: This function should not increase supply, but it does.
    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
        // This line is the bug! It wrongly increases extraSupply.
        extraSupply += amount;
    }
}
```

The `burn` function should decrease the total supply, but it incorrectly increases the `extraSupply` variable, which will cause `totalSupply` to not match the sum of all user balances over time.

#### 2. The Invariant Test Script (`MyToken.t.sol`)

This Foundry test script is designed to catch the bug by continuously checking if the sum of all token balances equals the total supply.

```solidity
// MyToken.t.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {MyToken} from "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken private token;
    address public attacker;

    function setUp() public {
        token = new MyToken();
        attacker = makeAddr("attacker");
    }

    // This is our invariant function.
    // Its name must start with `invariant_`.
    function invariant_totalSupplyEqualsBalanceSum() public view {
        uint256 sumOfAllBalances = token.balanceOf(attacker) + token.balanceOf(address(this));

        // This assertion checks the core invariant.
        assertEq(token.totalSupply(), sumOfAllBalances, "Invariant broken: Total supply does not equal the sum of balances!");
    }

    // The random testing target functions.
    // Foundry will randomly call functions with `fuzz` parameters.

    function testFuzz_mint_transfer_and_burn(uint256 amount) public {
        // We randomly generate an amount.
        // Randomly mint.
        token.mint(attacker, amount);

        // Randomly burn.
        if (amount > 0) {
            vm.prank(attacker);
            token.burn(amount);
        }
    }
}
```

Foundry will automatically execute the `testFuzz_...` function with random inputs and, after each execution, it will call `invariant_totalSupplyEqualsBalanceSum()` to check if the contract's state is still valid. The test will eventually fail when a `burn` call leads to an inconsistent state.

#### 3. Running the Test and Finding the Bug

To run this invariant test, you would use the following command:

`forge test --invariant`

Foundry will then:

- Start randomly calling `testFuzz_...` functions.
- After each call, it automatically runs `invariant_totalSupplyEqualsBalanceSum()`.
- When the `burn` function is called with a random amount, the `totalSupply()` will no longer match the sum of balances, and the test will fail.
- Foundry will provide a detailed report showing the exact sequence of transactions that broke the invariant, allowing you to quickly locate and fix the `burn` function's bug.

---

### Conclusion

In the world of Web3, Fuzz (Invariant) testing is a critical line of defense. It's not about verifying "does this function work correctly?" but rather "under any possible circumstance, do the core properties of my contract remain unchanged?" This makes it an essential part of any robust smart contract security audit, ensuring your protocol is resilient enough for the unpredictable nature of the blockchain.
