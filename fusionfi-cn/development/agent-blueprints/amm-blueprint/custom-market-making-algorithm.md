---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Custom Market-Making Algorithm

The **AMM Blueprint**’s modular design allows developers to define and extend their own market-making algorithms by implementing core Pool instance methods. This design defines a universal interface, enabling different algorithms to seamlessly integrate into the AMM system.

### Interface

Custom AMM algorithms must implement the following interfaces:

* **Pool.new**: Initializes the Pool instance.
* **Pool:balances**: Returns the asset balances in the pool.
* **Pool:Info**: Returns metadata about the pool.
* **Pool:updateLiquidity**: Updates the liquidity of the pool.
* **Pool:updateAfterSwap**: Updates the pool’s input or output asset balances after a swap.
* **Pool:getAmountOut**: Calculates the output asset quantity based on a given input asset amount.

Developers must ensure that these interfaces are implemented to meet the AMM Blueprint’s call conventions.

Below, we will analyze the **BigOrder** market-making algorithm to help developers better understand how to create custom market-making algorithms.

### BigOrder Market-Making Algorithm Analysis

**BigOrder** is a simple AMM market-making algorithm that facilitates exchange based on a fixed ratio between the input and output assets.

**Use Case Example:**

For instance, a user wants to sell 10,000 AR at a price of 20 USDC per AR. If directly swapped on a DEX, there might be significant slippage due to insufficient liquidity. To avoid this, the user can deploy an AMM Agent based on the **BigOrder** algorithm, setting tokenIn as 10,000 AR and tokenOut as 200,000 USDC. The agent will then offer this liquidity to the FFP system, ensuring that matching orders for AR purchases are automatically filled. The agent will continue matching orders in FFP until all 10,000 AR are exchanged.

### Code Breakdown:

#### 1. Pool.new

<details>

<summary>code</summary>

```lua
function Pool.new(x, y, params)
    if not validateInitParams(params) then
        return nil
    end
    local self = setmetatable({}, Pool)
    self.algo = 'BigOrder'
    self.x = x
    self.y = y
    self.tokenIn = params.tokenIn
    self.tokenOut = params.tokenOut
    self.amountOut = params.amountOut
    self.amountIn = params.amountIn
    self.balanceTokenIn = '0'
    self.balanceTokenOut = self.amountOut
    return self
end
```

</details>

This method initializes a new BigOrder pool instance.

Parameters:

* tokenIn and tokenOut are the input and output asset types.
* amountIn and amountOut specify the exchange ratio.
* balancesTokenIn and balancesTokenOut track the asset balances.

#### 2. Pool:balances

<details>

<summary>code</summary>

```lua
function Pool:balances()
    local balance = {
        [self.tokenOut] = self.balanceTokenOut,
        [self.tokenIn] = self.balanceTokenIn
    }
    return balance
end
```

</details>

This method returns the current balances of the two assets in the pool.

#### 3. Pool:Info

<details>

<summary>code</summary>

```lua
function Pool:info()
    local info = {
        x = self.x,
        y = self.y,
        algo = self.algo,
        tokenIn = self.tokenIn,
        tokenOut = self.tokenOut,
        amountOut = self.amountOut,
        amountIn = self.amountIn,
        balances = self:balances()
    }
    return info
end
```

</details>

This method returns metadata about the pool.

#### 4. Pool:updateLiquidity

```lua
function Pool:updateLiquidity(params)
    return false
end
```

This algorithm does not require liquidity pool information updates.

#### 5. Pool:updateAfterSwap

```lua
function Pool:updateAfterSwap(tokenIn, amountIn, tokenOut, amountOut)
   self.balanceTokenIn = utils.add(self.balanceTokenIn, amountIn)
   self.balanceTokenOut = utils.subtract(self.balanceTokenOut, amountOut)
end
```

After a trade is executed, this method updates the asset balances in the pool.

#### 6. Pool:getAmountOut

```lua
function Pool:getAmountOut(tokenIn, amountIn)
    if tokenIn == self.tokenIn then
        local amountOut = utils.div(utils.mul(amountIn, self.amountOut), self.amountIn)
        if utils.lt(amountOut, self.balanceTokenOut) then
            return self.tokenOut, tostring(amountOut)
        end
    end

    return nil, nil
end
```

This method calculates the output amount of assets based on the input asset type and amount.

The formula for the fixed ratio is:

$$
\text{amountOut} = \frac{\text{amountIn} \times \text{amountOut}}{\text{amountIn}}
$$



### Complete Code

<details>

<summary>bigorder.lua</summary>

```lua
local bint = require('.bint')(1024)

local Pool = {}
Pool.__index = Pool 

local utils = {
    add = function (a,b) 
      return tostring(bint(a) + bint(b))
    end,
    subtract = function (a,b)
      return tostring(bint(a) - bint(b))
    end,
    mul = function(a, b)
        return tostring(bint.__mul(bint(a), bint(b)))
    end,
    div = function(a, b)
        return tostring(bint.udiv(bint(a), bint(b)))
    end,
    lt = function (a, b)
        return bint.__lt(bint(a), bint(b))
    end
}

local function validateAmount(amount)
    if not amount then
        return false, 'err_no_amount'
    end
    local ok, qty = pcall(bint, amount)
    if not ok then
        return false, 'err_invalid_amount'
    end
    if not utils.lt(0, qty) then
        return false, 'err_negative_amount'
    end
    return true, nil
end

local function validateInitParams(params)
    if not params then
        return false
    end
    return params.tokenOut and params.tokenIn and validateAmount(params.amountOut) and validateAmount(params.amountIn)
end

function Pool.new(x, y, params)
    if not validateInitParams(params) then
        return nil
    end
    local self = setmetatable({}, Pool)
    self.algo = 'BigOrder'
    self.x = x
    self.y = y
    self.tokenIn = params.tokenIn
    self.tokenOut = params.tokenOut
    self.amountOut = params.amountOut
    self.amountIn = params.amountIn
    self.balanceTokenIn = '0'
    self.balanceTokenOut = self.amountOut
    return self
end

function Pool:balances()
    local balance = {
        [self.tokenOut] = self.balanceTokenOut,
        [self.tokenIn] = self.balanceTokenIn
    }
    return balance
end

function Pool:info()
    local info = {
        x = self.x,
        y = self.y,
        algo = self.algo,
        tokenIn = self.tokenIn,
        tokenOut = self.tokenOut,
        amountOut = self.amountOut,
        amountIn = self.amountIn,
        balances = self:balances()
    }
    return info
end

function Pool:updateLiquidity(params)
    return false
end

function Pool:updateAfterSwap(tokenIn, amountIn, tokenOut, amountOut)
   self.balanceTokenIn = utils.add(self.balanceTokenIn, amountIn)
   self.balanceTokenOut = utils.subtract(self.balanceTokenOut, amountOut)
end

function Pool:getAmountOut(tokenIn, amountIn)
    if tokenIn == self.tokenIn then
        local amountOut = utils.div(utils.mul(amountIn, self.amountOut), self.amountIn)
        if utils.lt(amountOut, self.balanceTokenOut) then
            return self.tokenOut, tostring(amountOut)
        end
    end

    return nil, nil
end

return Pool

```

</details>

### Summary

**BigOrder** is a simple and easy-to-use market-making algorithm, with a clear and straightforward implementation logic, making it an ideal example for custom AMM market-making algorithms in the AMM Blueprint. Developers can use the **BigOrder** implementation as a reference to build more complex algorithms, such as **Uniswap V2** or **V3 market-making curve algorithms.**
