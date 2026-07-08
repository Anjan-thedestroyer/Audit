---
title: "Smart Contract Security Audit Report"
subtitle: "TSwap Protocol"
author: ["Abinash Paudel"]
date: "\\today"
titlepage: true
titlepage-color: "FFFFFF"
titlepage-text-color: "1B1F3B"
titlepage-rule-color: "5F5F5F"
titlepage-rule-height: 2
titlepage-logo: "logo.pdf"
logo-width: 35mm
toc: true
toc-own-page: true
toc-title: "Table of Contents"
numbersections: true
listings: true
listings-disable-line-numbers: false
code-block-font-size: "\\scriptsize"
colorlinks: true
linkcolor: "4077C0"
urlcolor: "4077C0"
footer-left: "Confidential"
footer-right: "\\thepage"
header-left: "TSwap — Security Audit"
header-right: "\\today"
---

<!--
LOGO SETUP:
Place your logo image (e.g. logo.png) in the same folder as this markdown file
and as the eisvogel template when you run pandoc. The `titlepage-logo` field
above already points to "logo.png" — just replace that file, or change the
filename in the YAML header to match yours. It renders at the bottom-left of
the cover page at the width set by `logo-width` (currently 35mm).
-->

# Executive Summary

**Project:** TSwap
**Repository:** [[link to repo](https://github.com/Cyfrin/5-t-swap-audit)]
**Commit hash audited:** `f426f57731208727addc20adb72cb7f5bf29dc03`
**Audit dates:** 28/06/2026 – 08/07/2026
**Auditor(s):** Abinash Paudel

TSwap is an automated market maker (AMM) protocol, structurally similar to Uniswap V1, allowing users to swap between WETH and any ERC20 pool token, and to add/remove liquidity via a factory-deployed pool per token.

## Scope

| File | SLOC | Description |
|---|---|---|
| `src/PoolFactory.sol` | 35 | Deploys and tracks one liquidity pool per ERC20 token |
| `src/TSwapPool.sol` | 315 | Core AMM logic: swaps, deposits, withdrawals, pricing |

**Total nSLOC:** 350
**Complexity score:** 174
**External interactions:** Many ERC20 tokens (arbitrary, including fee-on-transfer/rebasing/deflationary tokens not explicitly excluded)
**Test coverage at time of audit:** 40.91%

## Methodology

- Manual line-by-line review
- Static analysis with [Aderyn](https://github.com/Cyfrin/aderyn)
- Fuzz / invariant testing with Foundry

## Protocol Details

| | |
|---|---|
| Fork of existing protocol | Yes — UniswapV1 |
| Multi-chain | No (Ethereum mainnet only) |
| External oracles | No |
| External AMMs | No |
| Off-chain processes (keeper bots, etc.) | No |

## Severity Classification

| Severity | Definition |
|---|---|
| **Critical** | Directly exploitable, leads to loss of funds or full compromise |
| **High** | Exploitable under realistic conditions, significant impact |
| **Medium** | Exploitable under specific conditions, moderate impact |
| **Low** | Edge case or best-practice deviation, minor impact |
| **Informational / Gas** | Code quality, gas optimization, no security impact |

## Summary of Findings

| ID | Title | Severity | Status |
|---|---|---|---|
| H-1 | `deposit()` missing deadline check | High | Open |
| H-2 | Incorrect fee scaling in `getInputAmountBasedOnOutput()` | High | Open |
| H-3 | `x*y=k` invariant broken by extra reward transfer in `_swap()` | High | Open |
| L-1 | `LiquidityAdded` event parameters out of order | Low | Open |
| L-2 | `swapExactInput()` always returns 0 | Low | Open |
| I-1 | Unused error `PoolFactory__PoolDoesNotExist` | Informational | Open |
| I-2 | Missing zero-address check in constructor | Informational | Open |
| I-3 | `createPool` should use `.symbol()` not `.name()` | Informational | Open |
| I-4 | Magic numbers used instead of named constants | Informational | Open |
| I-5 | `swapExactInput()` should be `external` | Informational | Open |
| I-6 | Missing NatSpec for `deadline` param | Informational | Open |
| I-7 | Deprecated contract-type equality comparison | Informational | Open |

\newpage

# High Findings

## [H-1] `TSwapPool::deposit()` is missing deadline check causing the transaction to complete even if the deadline has passed

**Severity:** High
**Status:** Open

### Description

The `deposit` function accepts a `deadline` parameter which, per the documentation, is "the deadline for the transaction to be completed by." However, this parameter is never used. As a consequence, operations that add liquidity to the pool might execute at unexpected times, under market conditions where the deposit rate is unfavorable.

### Location

`src/TSwapPool.sol`

### Impact

Transactions could be mined when market conditions are unfavorable for the depositor, even though a deadline parameter was explicitly supplied to guard against this.

### Proof of Concept

The `deadline` parameter is accepted but unused anywhere in the function body.

### Recommendation

```diff
function deposit(
    uint256 wethToDeposit,
    uint256 minimumLiquidityTokensToMint,
    uint256 maximumPoolTokensToDeposit,
    uint64 deadline
)
    external
+   revertIfDeadlinePassed(deadline)
    revertIfZero(wethToDeposit)
    returns (uint256 liquidityTokensToMint)
{}
```

\newpage

## [H-2] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput()` causes the protocol to take too many tokens from users

**Severity:** High
**Status:** Open

### Description

`getInputAmountBasedOnOutput()` calculates the amount of input tokens required to receive a specific amount of output tokens. The fee calculation scales the amount by `10_000` instead of the correct `1_000`.

### Location

`src/TSwapPool.sol`

### Impact

The protocol takes roughly 10x more in fees than intended from users on every swap using this path.

### Proof of Concept

```solidity
uint256 expected =
    (output * inputReserve * 1000) /
    ((outputReserve - output) * 997);

uint256 actual =
    pool.getInputAmountBasedOnOutput(output, inputReserve, outputReserve);

assertEq(actual, expected * 10);
```

### Recommendation

```diff
-   ((inputReserves * outputAmount) * 10000) /
+   ((inputReserves * outputAmount) * 1000) /
    ((outputReserves - outputAmount) * 997);
```

\newpage

## [H-3] Invariant `x * y = k` is broken in `TSwapPool::_swap()` due to an extra reward transfer every `SWAP_COUNT_MAX` swaps

**Severity:** High
**Status:** Open

### Description

Where `x` is the pool-token balance, `y` is the WETH balance, and `k` is their constant product, `_swap()` sends an extra `1_000_000_000_000_000_000` (1e18) tokens to the swapper once the running swap count reaches `SWAP_COUNT_MAX`. This disrupts the AMM's core invariant — the ratio between the two balances no longer holds after the bonus payout.

### Location

`src/TSwapPool.sol`

### Impact

The protocol's pricing math and accounting become invalid after the bonus fires. A user can repeatedly trigger the bonus payout to drain the contract's WETH balance.

### Proof of Concept

<details>

```javascript
function testBroken() public {
    vm.startPrank(liquidityProvider);
    weth.approve(address(pool), 100e18);
    poolToken.approve(address(pool), 100e18);
    pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
    vm.stopPrank();
    uint256 wethAmount = 1e17;

    vm.startPrank(user);
    poolToken.approve(address(pool), type(uint256).max);
    poolToken.mint(user, 100e18);
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    int256 startingY = int256(weth.balanceOf(address(pool)));
    int256 expDeltaY = -int256(wethAmount);
    pool.swapExactOutput(poolToken, weth, wethAmount, uint64(block.timestamp));
    vm.stopPrank();

    uint256 endingY = weth.balanceOf(address(pool));
    int256 actualDeltaY = int256(endingY) - int256(startingY);
    assertEq(actualDeltaY, expDeltaY);
}
```

</details>

### Recommendation

Remove the extra incentive. If it must be kept, account for it against the invariant explicitly, the same way protocol fees are set aside.

```diff
- swap_count++;
- if (swap_count >= SWAP_COUNT_MAX) {
-     swap_count = 0;
-     outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
- }
```

\newpage

# Low Findings

## [L-1] `TSwapPool::LiquidityAdded` event has parameters out of order

**Severity:** Low
**Status:** Open

### Description

When `LiquidityAdded` is emitted in `_addLiquidityMintAndTransfer`, the logged values are in the wrong order relative to the event's declared parameters.

### Impact

Event emission is incorrect, which can cause off-chain indexers/consumers to misattribute values.

### Recommendation

```diff
-   emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+   emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

\newpage

## [L-2] `TSwapPool::swapExactInput()` always returns 0

**Severity:** Low
**Status:** Open

### Description

`swapExactInput` is expected to return the actual amount of tokens bought by the caller. It declares the named return value `output`, but never assigns it and has no explicit `return` statement.

### Impact

The return value will always be `0`, giving callers (and any composing contracts) incorrect information about the swap they just executed.

### Proof of Concept

```solidity
function testOutput() public {
    vm.startPrank(liquidityProvider);
    weth.approve(address(pool), 100e18);
    poolToken.approve(address(pool), 100e18);
    pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
    vm.stopPrank();
    vm.startPrank(user);
    poolToken.approve(address(pool), 1 ether);
    uint256 output = pool.swapExactInput(
        poolToken,
        1 ether,
        weth,
        1000,
        uint64(block.timestamp)
    );
    assertEq(output, 0);
}
```

### Recommendation

```diff
function swapExactOutput(
    IERC20 inputToken,
    IERC20 outputToken,
    uint256 outputAmount,
    uint64 deadline
)
    public
    revertIfZero(outputAmount)
    revertIfDeadlinePassed(deadline)
    returns (uint256 inputAmount)
{
    uint256 inputReserves = inputToken.balanceOf(address(this));
    uint256 outputReserves = outputToken.balanceOf(address(this));

    inputAmount = getInputAmountBasedOnOutput(
        outputAmount,
        inputReserves,
        outputReserves
    );
    _swap(inputToken, inputAmount, outputToken, outputAmount);
+   return inputAmount;
}
```

\newpage

# Informational Findings

## [I-1] `PoolFactory::PoolFactory__PoolDoesNotExist` is unused and should be removed

```diff
-error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

## [I-2] Lacking zero-address check in `PoolFactory` constructor

```diff
 constructor(address wethToken) {
+    if (wethToken == address(0)) revert PoolFactory__WethIsZeroAddress();
     i_wethToken = wethToken;
 }
```

## [I-3] `PoolFactory::createPool` should use `.symbol()` instead of `.name()`

```diff
-string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

## [I-4] Magic numbers should be replaced with named constants

Affects `getInputAmountBasedOnOutput()`, `getPriceOfOneWethInPoolTokens()`, and `getPriceOfOnePoolTokenInWeth()` in `TSwapPool.sol`.

```diff
-((inputReserves * outputAmount) * 10000) /
-    ((outputReserves - outputAmount) * 997);
+uint256 private constant FEE_PRECISION = 1000;
+uint256 private constant FEE_MULTIPLIER = 997;
```

## [I-5] `TSwapPool::swapExactInput()` should be `external`

```diff
function swapExactInput(
    IERC20 inputToken,
    uint256 inputAmount,
    IERC20 outputToken,
    uint256 minOutputAmount,
    uint64 deadline
)
-   public
+   external
    revertIfZero(inputAmount)
    revertIfDeadlinePassed(deadline)
    returns (uint256 output)
{}
```

## [I-6] `TSwapPool::swapExactOutput()` is missing NatSpec for the `deadline` parameter

```diff
 * @param inputToken ERC20 token to pull from caller
 * @param outputToken ERC20 token to send to caller
 * @param outputAmount The exact amount of tokens to send to caller
+ * @param deadline The deadline for the transaction to be completed by
 */
function swapExactOutput(
    IERC20 inputToken,
    IERC20 outputToken,
    uint256 outputAmount,
    uint64 deadline
)
```

## [I-7] `TSwapPool::_swap()` uses deprecated contract-type equality comparison

Comparing variables of contract type is deprecated and scheduled for removal; cast to `address` and compare explicitly.

```diff
-if (
-    _isUnknown(inputToken) ||
-    _isUnknown(outputToken) ||
-    inputToken == outputToken
-)
+if (
+    _isUnknown(inputToken) ||
+    _isUnknown(outputToken) ||
+    address(inputToken) == address(outputToken)
+)
```

\newpage

# Gas / Static Analysis Notes (Aderyn)

Automated static analysis via [Aderyn](https://github.com/Cyfrin/aderyn) surfaced the following, in addition to the manual findings above:

- `public` functions not used internally (e.g. `swapExactInput`) could be marked `external` — see I-5.
- Repeated literal values (`997`, `1e18`) should be declared as named constants — see I-4.
- `PoolCreated`, `LiquidityAdded`, `LiquidityRemoved`, and `Swap` events are missing `indexed` fields, reducing off-chain query efficiency.
- Solidity `0.8.20` targets the Shanghai EVM version by default, which emits `PUSH0` — confirm target chain(s) support this opcode before deployment.
- Large literals (e.g. `10000`) can be expressed in scientific notation (`1e4`) for readability.
- The unused `PoolFactory__PoolDoesNotExist` custom error should be removed — see I-1.

\newpage

# Disclaimer

This report is provided for informational purposes only and does not constitute a guarantee of security. A security audit is not a warranty of bug-free code, and does not guarantee the complete absence of vulnerabilities. The findings reflect the codebase as of the commit hash listed above; any changes made afterward are outside the scope of this report.
