
## Informationals
### [I-1] `PoolFactory::PoolFactory__PoolDoesnotExist` is not used and should be removed

**Description:**
```diff
-error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking zero address check in `PoolFactory::createPool`

```diff
 constructor(address wethToken) {
        i_wethToken = wethToken;
    }
```
### [I-3] `poolFactory::createPool` should use `.symbol()` instead of `.name()`

```diff
        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
```
### [I-4] `TSwapPool::getInputAmountBasedOnOutput()` `TSwapPool::getPriceOfOneWethInPoolTokens()` `TSwapPool::getPriceOfOnePoolTokenInWeth()`shouldnot use magic numbers, the required numbers should first be declared and then used.

```diff
   ((inputReserves * outputAmount) * 10000) /
            ((outputReserves - outputAmount) * 997);
```
```diff
function getPriceOfOneWethInPoolTokens() external view returns (uint256) {
        return
            getOutputAmountBasedOnInput(
                1e18,
                i_wethToken.balanceOf(address(this)),
                i_poolToken.balanceOf(address(this))
            );
    }
```
```diff
 function getPriceOfOnePoolTokenInWeth() external view returns (uint256) {
        return
            getOutputAmountBasedOnInput(
                1e18,
                i_poolToken.balanceOf(address(this)),
                i_wethToken.balanceOf(address(this))
            );
    }
```
### [I-5] `TSwapPool::swapExactInput()` should be `external` function
```diff
    function swapExactInput(
            IERC20 inputToken,
            uint256 inputAmount,
            IERC20 outputToken,
            uint256 minOutputAmount,
            uint64 deadline
        )
            public
            revertIfZero(inputAmount)
            revertIfDeadlinePassed(deadline)
            returns (uint256 output)
        {}
```

### [I-6] `TSwapPool::swapExactOutput()` should have the natspec for its parameter `uint64 deadline`
```diff
 * @param inputToken ERC20 token to pull from caller
     * @param outputToken ERC20 token to send to caller
     * @param outputAmount The exact amount of tokens to send to caller
     */
    function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
    )
```
### [I-7] `TSwapPool::_swap()` Comparison of variables of contract type is deprecated and scheduled for removal. Use an explicit cast to address type and compare the addresses instead.
```diff
 if (
            _isUnknown(inputToken) ||
            _isUnknown(outputToken) ||
            inputToken == outputToken
        )
```







## High

### [H-1] `TSwapPool::deposit()` is missing deadline check causing the transcation to complete even if the deadline is completed

**Description:** The `deposit` funciton accepts a deadline parameter which according to the documentation is "the deadline for the transcation to be completed by". However, thi parameter is never used. As a consequence, operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

**Impact:** Transactions could be sent when market condition are unfavorabe to deposit, even when adding a deadline parameter.

**Proof of Concept:** The `deadline` parameter is unused.

**Recommended Mitigation:** Consider making th following change to the code
```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfZero(uint64 deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {}
```

### [H-2] Incorrect Fee calculation in `TSwapPool::getInputAmountBaseOnOutput()` causes protocol to take too many tokens from users, resulting in more fee

**Description:** The `getInputAmountBasedOnOutput()` function is used to calculate the amount of input tokens required to receive a specific amount of output tokens. However, the fee calculation in this function is incorrect, it scales the amount by 10_000 instead of 1_000.

**Impact:** Protocol takes more fees than expected from user.

**Proof of concepts:**
```diff 
    uint256 expected =
        (output * inputReserve * 1000) /
        ((outputReserve - output) * 997);

    uint256 actual =
        pool.getInputAmountBasedOnOutput(output, inputReserve, outputReserve);

assertEq(actual, expected * 10);
```
**Recommended Mitigation:** 
change the value
```diff
-   ((inputReserves * outputAmount) * 10000) /
+   ((inputReserves * outputAmount) * 1000) /
    ((outputReserves - outputAmount) * 997);
```
### [H-3] Invariant-break of `x * y = k` in `TSwapPool::_swap` function due to transfer of extra tokens to the user after every `swapCount` equals to `SWAP_COUNT_MAX`

**Description:**
- `x`: The balance of the pool tokens
- `y`: The balance of WETH
- `k`: The constant product of two balances

In this function `_swap` the protocol tries to send `1_000_000_000_000_000_000` to the swapper after  number of swap is equals to `SWAP_COUNT_MAX`, ultimately disrupting the invariant. This means the ratio between the two amount will not remains same.

**Impact:** Protcols math and accounting gets disturbed and the calculation becomes invalid. User can also drain the whole contract by swapping again and again

**Proof of Concept:**
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
         pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expDeltaY = -int256(wethAmount);
        pool.swapExactOutput(poolToken, weth,wethAmount, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);
        assertEq(actualDeltaY, expDeltaY);

    }
```

</details>

**Recommended Mitigation:**Remove the extra incentive. If you want to keep in, we should account for the change in the invariant. Or, we should set aside tokens in the same way we do with fees.
```diff
- swap_count++;
-       if (swap_count >= SWAP_COUNT_MAX) {
-           swap_count = 0;
-           outputToken.safeTransfer(msg.sender,         1_000_000_000_000_000_000);
-       }
```

## Low
### [L-1] `TSwapPool::LiquidityAdded` event has parameter out of order.

**Description:** When the `LiquidityAdded` event is emmited in the `_addLiquidityMintAndTransfer` function, its log the value in incorrect order.

**Impact:** Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation:** 
```diff
-        emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+                 emit LiquidityAdded(msg.sender,wethToDeposit,poolTokensToDeposit);
```

### [L-2] `TSwapPool::swapExactInput` results in incorrect return value

**Description:** The `swapExactInput` funciton is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `output` it is nver assigned a value nor uses an explict return statement.

**Impact:** The return value will always be 0, giving incorrect information to the caller.

**Proof of Concept:**
```diff
    function testOutput() public{
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
**Recommended Mitigation:** 
Apply this change to the code
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
+       return inputAmount;
    }
```