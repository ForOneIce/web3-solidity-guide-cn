# Smart Calculator

Let’s say you’re building a simple calculator in Solidity. You want it to handle basic math — addition, subtraction, multiplication, and division.

Things are smooth until a user says:

> “Hey, can this also calculate square roots? Or raise numbers to powers?”
> 

You could throw everything into one huge contract...

But there's a better approach: **split responsibilities**.

One contract will handle the basics.

Another contract will handle advanced stuff — like powers and square roots.

Then, we’ll make them **talk to each other**.

Let’s build this step by step.

---

## The Plan: Two Contracts, Two Files

We’re going to build **two smart contracts** that work together:

1. **`Calculator.sol`** – This will be our main calculator. It handles basic math like addition, subtraction, multiplication, and division. When it needs help with more advanced math, it delegates the task.
2. **`ScientificCalculator.sol`** – This contract is where we’ll put the advanced stuff — like exponentiation (powers) and square root calculations.

Each contract will live in its **own separate file**, but they’ll be connected.

<aside>
💻

Here is the complete code 👇🏼
1️⃣ Calculator.sol : 

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/Calculator.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/Calculator.sol)

2️⃣ ScientificCalculator.sol : 

[https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/ScientificCalculator.sol](https://github.com/snehasharma76/30daysSolidity_Web3Compass/blob/master/ScientificCalculator.sol)

</aside>

In order to call functions from `ScientificCalculator` inside the `Calculator` contract, we’ll **import** it and store its deployed address.

> ⚠️ Important: Keep both files in the same directory, so the import works without issues.
> 

---

## What Do These Files Look Like?

Before we dive into writing the logic, here’s what the basic structure — or "shell" — of each file looks like:

### File 1: `ScientificCalculator.sol`

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ScientificCalculator {
    // advanced functions will go here
}

```

This file defines the contract that handles powers and square roots — all the math that goes beyond the basics.

---

### File 2: `Calculator.sol`

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./ScientificCalculator.sol";

contract Calculator {
    // basic math functions}

```

This file defines the main `Calculator` contract. It includes:

- Basic arithmetic functions

With the structure in place, let’s now move into the `ScientificCalculator` and start building it out, one function at a time.

## ScientificCalculator – Starting with Advanced Math

We’ll begin with the contract that handles the advanced operations — `ScientificCalculator`.

Here’s the very first part of the file:

```solidity
 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

```

These are the standard declarations at the top of every Solidity file. They make sure your contract compiles properly and is legally usable.

Now let’s define our contract:

```solidity
 
contract ScientificCalculator {

```

We’re creating a new contract named `ScientificCalculator`. Inside this, we’ll start by writing a function to calculate powers.

---

### power(base, exponent)

```solidity
 
function power(uint256 base, uint256 exponent) public pure returns (uint256) {
    if (exponent == 0) return 1;
    else return (base ** exponent);
}

```

This function returns the result of raising `base` to the power of `exponent`.

- If the exponent is 0, it returns 1 — that’s just standard math.
- Otherwise, it uses Solidity’s `*` operator to calculate `base` to the power of `exponent`.
- It’s marked `pure` because it doesn’t read or change anything on the blockchain. It just does math.

Next, we’ll add a function to estimate square roots.

---

### squareRoot(number)

```solidity
 
function squareRoot(uint256 number) public pure returns (uint256) {
    require(number >= 0, "Cannot calculate square root of negative number");
    if (number == 0) return 0;

    int256 result = number / 2;
    for (uint256 i = 0; i < 10; i++) {
        result = (result + number / result) / 2;
    }
    return result;
}

```

This function estimates the square root of a number using a classic technique called **Newton’s Method** — a smart way to find square roots through repeated approximations.

Let’s break it down step by step:

- First, we check if the input number is negative using `require`. Solidity doesn’t support complex numbers, so square roots of negatives are off-limits.
- Then we handle a quick edge case: if the number is 0, the square root is also 0 — so we return that right away.
- Now comes the actual approximation. We start with a **rough guess** by dividing the number in half:
    
    `uint256 result = number / 2;`
    
- We then **refine that guess 10 times** using a simple formula:
    
    `(result + number / result) / 2`
    
    This formula works by combining the current guess with what we’d get if the guess were perfect — and averaging them. Each loop brings us closer to the actual square root.
    

Even though we’re not using floating-point math, this method still gives a pretty decent estimate for whole numbers — good enough for many practical use cases in Solidity.

---

## Calculator – The Main Math Engine (Plus Some Help)

This contract handles the basic math: adding, subtracting, multiplying, and dividing numbers. But when things get more advanced — like calculating powers or square roots — it doesn’t do everything on its own.

Instead, it reaches out to the `ScientificCalculator` contract for help.

Let’s start from the top of the file:

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./ScientificCalculator.sol";

```

Here’s what’s going on:

We’re using an `import` statement to bring the code from `ScientificCalculator.sol` into this file. This allows the `Calculator` contract to **use functions from that other contract**.

The `"./"` part tells Solidity:

> “Look in the same directory (or folder) as this file and find ScientificCalculator.sol.”
> 

So for this import to work properly, both files — `Calculator.sol` and `ScientificCalculator.sol` — **must be saved in the same folder**.

This setup is what allows one contract to know about and interact with the other.

Now, let’s start building the rest of the `Calculator` contract. We’ll begin by declaring the contract itself and setting up a few important variables.

Now, we declare the contract:

```solidity
 
contract Calculator {

```

Inside the contract, let’s define a few **state variables** to store useful information.

---

### State Variables

```solidity
 
address public owner;
address public scientificCalculatorAddress;

```

- `owner` will store the address that deployed this contract.
- `scientificCalculatorAddress` is where we’ll store the address of the already-deployed `ScientificCalculator`.

Now, let’s set the owner during deployment.

---

### constructor()

```solidity
 
constructor() {
    owner = msg.sender;
}

```

When someone deploys this contract, their address is stored as the `owner`.

We’ll use that later to restrict certain functions.

---

### onlyOwner Modifier

```solidity
 
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can perform this action");
    _;
}

```

This is a **modifier**, which is like a reusable gatekeeper.

Any function using `onlyOwner` can only be called by the original deployer of the contract.

Let’s use it now to allow the owner to link a ScientificCalculator.

---

### setScientificCalculator(address)

```solidity
 
function setScientificCalculator(address _address) public onlyOwner {
    scientificCalculatorAddress = _address;
}

```

Once your ScientificCalculator contract is deployed, you can copy its address and pass it here. This function saves that address so we can call its functions later.

---

## Basic Math Functions

Let’s now define some basic arithmetic operations. These are all `pure` functions — they don’t touch the blockchain state and just return results.

---

### add(a, b)

```solidity
 
function add(uint256 a, uint256 b) public pure returns (uint256) {
    uint256 result = a + b;
    return result;
}

```

Simple addition. Takes two numbers and returns the sum.

---

### subtract(a, b)

```solidity
 
function subtract(uint256 a, uint256 b) public pure returns (uint256) {
    uint256 result = a - b;
    return result;
}

```

Subtracts `b` from `a` and returns the result.

---

### multiply(a, b)

```solidity
 
function multiply(uint256 a, uint256 b) public pure returns (uint256) {
    uint256 result = a * b;
    return result;
}

```

Multiplies the two numbers together.

---

### divide(a, b)

```solidity
 
function divide(uint256 a, uint256 b) public pure returns (uint256) {
    require(b != 0, "Cannot divide by zero");
    uint256 result = a / b;
    return result;
}

```

Divides `a` by `b`. But before doing that, it checks if `b` is zero — just to avoid errors.

---

## Connecting to Another Contract: Power Function

Now here’s where things get really interesting — we’re going to make one smart contract **talk to another**.

Let’s say the user wants to calculate something like `5 to the power of 3`. The basic `Calculator` contract doesn’t know how to do that — but the `ScientificCalculator` does.

Instead of re-writing that logic again, we’ll **call the `power()` function from the other contract** and return the result.

---

### `calculatePower(base, exponent)`

```solidity

function calculatePower(uint256 base, uint256 exponent) public view returns (uint256) {
    ScientificCalculator scientificCalc = ScientificCalculator(scientificCalculatorAddress);
    uint256 result = scientificCalc.power(base, exponent);
    return result;
}

```

Let’s break this down step by step.

---

### Step 1: Address Casting

```solidity

ScientificCalculator scientificCalc = ScientificCalculator(scientificCalculatorAddress);

```

This line is doing something really important. It’s taking a plain Ethereum address (`scientificCalculatorAddress`) and **casting** it into a usable contract object — in this case, a `ScientificCalculator`.

You can think of it like this:

> “Solidity, here’s an address on the blockchain. I know it points to a ScientificCalculator contract. Please treat it like one so I can call its functions.”
> 

This process is called **address casting** — you’re converting an address into a contract reference so you can interact with it directly.

And this works **only because we’ve already imported `ScientificCalculator.sol`** at the top of the file. Solidity now knows the structureof that contract, so it allows us to call its functions safely.

---

### Step 2: Calling the Function

```solidity
uint256 result = scientificCalc.power(base, exponent);

```

Now that we have a contract reference, calling a function on it is as simple as using dot notation — just like calling a method in any object-oriented programming language.

This sends a read-only call to the deployed `ScientificCalculator` contract, asking it to compute `base ** exponent`, and then returns the result.

---

### Step 3: Returning the Result

```solidity
return result;

```

We simply return the value so the user gets the answer to their calculation.

---

### Why This Approach Works

This is the **high-level, type-safe** way to call another contract:

- It’s easy to read and write
- It requires that you’ve imported the external contract's code
- It lets the compiler catch mistakes — like calling a non-existent function

This is the recommended approach when you **know the structure of the other contract** and have access to its source code.

---

## Bonus: Using Low-Level Calls

Sometimes, you want to interact with another contract **without importing its source code** — maybe you only know the address and the name of the function you want to call.

In those cases, you can use a **low-level call**. It’s more flexible but also riskier because Solidity can’t protect you from mistakes at compile time.

Let’s see how this works in practice by calling the `squareRoot()` function from the `ScientificCalculator` contract **without using a direct import**.

---

### `calculateSquareRoot(number)`

```solidity

function calculateSquareRoot(uint256 number) public returns (uint256) {
    require(number >= 0, "Cannot calculate square root of negative number");

    bytes memory data = abi.encodeWithSignature("squareRoot(int256)", number);
    (bool success, bytes memory returnData) = scientificCalculatorAddress.call(data);
    require(success, "External call failed");

    uint256 result = abi.decode(returnData, (uint256));
    return result;
}

```

---

Let’s walk through this line by line.

---

### Step 1: Input Validation

```solidity

require(number >= 0, "Cannot calculate square root of negative number");

```

We start by making sure the number is not negative. Solidity doesn’t support imaginary numbers, so we reject negative inputs up front.

---

### Step 2: Encode the Function Call

```solidity

bytes memory data = abi.encodeWithSignature("squareRoot(int256)", number);

```

This is the **heart of the low-level call**. We use something called the **ABI** to prepare the function call.

### What is ABI?

ABI stands for **Application Binary Interface**. Think of it as a contract’s "communication protocol" — it defines how data must be structured when one contract calls another.

When using high-level function calls (like `otherContract.someFunction()`), Solidity handles ABI encoding for you. But with low-level calls, **you must do it manually**.

---

### What does `abi.encodeWithSignature` do?

It builds the exact binary format the EVM expects when calling a specific function.

In this case:

```solidity

abi.encodeWithSignature("squareRoot(int256)", number)
```

- `"squareRoot(int256)"` is the full function signature (name + parameter types).
- `number` is the value we're passing as an argument.
- The result is a byte array (`bytes memory`) that contains everything needed to call that function on the blockchain.

---

### Step 3: Make the Low-Level Call

```solidity

(bool success, bytes memory returnData) = scientificCalculatorAddress.call(data);
```

Here, we’re telling the Ethereum Virtual Machine (EVM):

> “Hey, send a raw call to this address, using the encoded data we just created.”
> 
- `.call(data)` sends that data to the address stored in `scientificCalculatorAddress`.
- It returns two things:
    - `success` (a boolean that tells us if the call worked)
    - `returnData` (a byte array that holds whatever the function returned)

> 🔧 This is like calling a function manually through the back door — you need to get everything exactly right.
> 

---

### Step 4: Check if the Call Succeeded

```solidity

require(success, "External call failed");

```

Always check if the call worked. If something went wrong (wrong signature, wrong address, function not found, etc.), `success` will be false — and we stop execution with a helpful error message.

---

### Step 5: Decode the Response

```solidity

uint256 result = abi.decode(returnData, (uint256));

```

Now we take the raw return data and decode it back into a usable value — in this case, a `uint256`.

Since we know the return type of `squareRoot()` is `int256` (and we're returning it as `uint256` here), you might choose to adjust this depending on how you handle signed values. For simplicity, we assume the number passed in is safe and positive.

---

### Step 6: Return the Final Result

```solidity
return result;

```

Finally, we return the result of the square root calculation to whoever called this function.

---

## Recap

By now, you’ve built two contracts that:

- Divide logic between basic and advanced math
- Talk to each other using both **high-level calls** and **low-level calls**
- Show how to **modularize logic** and keep things organized
- Demonstrate safe practices like access control and error checking

These are essential patterns for building real-world decentralized applications that are clean, maintainable, and scalable.