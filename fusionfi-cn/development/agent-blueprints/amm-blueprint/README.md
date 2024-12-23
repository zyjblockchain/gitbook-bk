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

# AMM Blueprint

The **AMM Blueprint** provides developers with a modular template for building AMM-based applications. It enables developers to implement custom market-making algorithms or use two built-in algorithms: the **UniswapV2 Curve** and the **Constant Price** **Algorithm**. Below is a detailed analysis of its code logic and features.

### Core Features

#### 1. Pool Management

* Supports adding new pools and setting liquidity for them.
* Each pool can use different market-making algorithms (e.g., UniswapV2, BigOrder).
* The state of each pool is maintained independently within FFP.Pools.

#### 2. Order Operations

* Allows users to execute trades via pools (e.g., swap transactions).
* Validates the input and output of orders.
* Creates Notes (tickets) that match pool prices to facilitate orders.

#### 3. Settlement Operations

* Executes settlement requests from the FFP settlement center to finalize Notes.
* Maintains the settlement status of orders (e.g., completed, expired, rejected).
* Automatically handles expired or completed Notes.

#### 4. Flexible Market-Making Strategies

* Uses the AlgoToPool mapping table to assign different market-making algorithms to pools.
* Developers can implement new market-making algorithms within this framework.

### Detailed Code Analysis

#### 1. Pool Management

**Creating a Pool**

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.addPool', 'FFP.AddPool', 
    function(msg)
        assert(msg.From == Owner or msg.From == ao.id, 'Only owner can add pool')
        assert(type(msg.X) == 'string', 'X is required')
        assert(type(msg.Y) == 'string', 'Y is required')
        assert(msg.X < msg.Y, 'X must be less than Y')
        assert(type(msg.Algo) == 'string', 'Algo is required')
        assert(type(msg.Data) == 'string', 'Params is required')

        local poolId = getPoolId(msg.X, msg.Y)
        if FFP.Pools[poolId] then
            msg.reply({Error = 'Pool already exists'})
            return
        end

        local P = FFP.AlgoToPool[msg.Algo]
        if not P then
            msg.reply({Error = 'Unsupported amm algo'})
            return
        end
        
        local pool = P.new(msg.X, msg.Y, json.decode(msg.Data))
        if not pool then
            msg.reply({Error = 'Invalid pool params'})
            return
        end

        FFP.Pools[poolId] = pool
        msg.reply({Action = 'PoolAdded-Notice', Pool = poolId})
        Send({Target = FFP.Publisher, Action = 'PoolAdded-Notice', Creator = ao.id, Data = json.encode(pool)})
    end
)
```

</details>

Logic:

* Check if a pool with the same configuration already exists.
* Specify the market-making algorithm to create the pool.
* Add the pool to FFP.Pools.

**Setting Pool Liquidity**

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.updateLiquidity', 'FFP.UpdateLiquidity', 
    function(msg)
        assert(msg.From == Owner or msg.From == ao.id, 'Only owner can update liquidity')
        assert(type(msg.Y) == 'string', 'Y is required')
        assert(msg.X < msg.Y, 'X must be less than Y')
        assert(type(msg.Data) == 'string', 'Params is required')
        
        local pool = getPool(msg.X, msg.Y)
        if not pool then
            msg.reply({Error = 'Pool not found'})
            return
        end
        
        local ok = pool:updateLiquidity(json.decode(msg.Data))
        if not ok then
            msg.reply({Error = 'Invalid updateLiquidity params'})
            return
        end

        msg.reply({Action = 'LiquidityUpdated-Notice', Data = json.encode(pool)})
    end
)
```

</details>

Logic:

* Locate the specified pool and call its updateLiquidity method.
* Notify the results after updating.

#### 2. Order Operations

**Creating Orders**

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.makeOrder', 'FFP.MakeOrder',
  function(msg)
    if FFP.Executing then
        msg.reply({Error = 'err_executing'})
        return
    end
    
    local ok, err = validateOrderMsg(msg)
    if not ok then
        msg.reply({Error = err})
        return
    end

    local pool = getPool(msg.TokenIn, msg.TokenOut)
    local tokenOut, amountOut = pool:getAmountOut(msg.TokenIn, msg.AmountIn)
    if not amountOut or amountOut == '0' then
        msg.reply({Error = 'err_no_amount_out'})
        return
    end

    if msg.AmountOut then
        local ok, err = validateAmountOut(pool, msg.TokenIn, msg.AmountIn, msg.TokenOut, msg.AmountOut)
        if not ok then
            msg.reply({Error = err})
            return
        end
        amountOut = msg.AmountOut
    end

    -- 90 seconds
    local expireDate = msg.Timestamp + FFP.TimeoutPeriod

    local res = Send({
        Target = FFP.Settle,
        Action = 'CreateNote',
        AssetID = msg.TokenOut,
        Amount = tostring(amountOut),
        HolderAssetID = msg.TokenIn,
        HolderAmount = msg.AmountIn,
        IssueDate = tostring(msg.Timestamp),
        ExpireDate = tostring(expireDate),
        Version = FFP.SettleVersion
    }).receive()
    local noteID = res.NoteID
    local note = json.decode(res.Data)
    note.MakeTx = msg.Id
    FFP.Notes[noteID] = note
    FFP.MakeTxToNoteID[msg.Id] = noteID
    -- remove expired notes in Notes
    for noteID, note in pairs(FFP.Notes) do
        if note.Status == 'Open' and note.ExpireDate and note.ExpireDate < msg.Timestamp then
            FFP.Notes[noteID] = nil
            FFP.MakeTxToNoteID[note.MakeTx] = nil
        end
    end

    msg.reply({
        Action = 'OrderMade-Notice',
        NoteID = noteID,
        Data = json.encode(note)
    })
  end
)
```

</details>

Logic:

* Validate input parameters and the current pool state. The pool’s Executing status acts as a global lock to ensure Notes are settled sequentially.
* Call pool.getAmountOut to calculate the trade’s output.
* Use the FFP protocol to create a Note and store it in FFP.Notes.

**Order Settlement Execution**

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.execute', 'Execute',
  function(msg)
    assert(msg.From == FFP.Settle, 'Only settle can start exectue')
    assert(type(msg.NoteID) == 'string', 'NoteID is required')
    assert(type(msg.SettleID) == 'string', 'SettleID is required')

    if FFP.Executing then
        msg.reply({Action= 'Reject', Error = 'err_executing', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    local note = FFP.Notes[msg.NoteID]
    if not note then
        msg.reply({Action= 'Reject', Error = 'err_not_found', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.Status ~= 'Open' then
        msg.reply({Action= 'Reject', Error = 'err_not_open', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.Issuer ~= ao.id then
        msg.reply({Action = 'Reject', Error = 'err_invalid_issuer', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.ExpireDate and note.ExpireDate < msg.Timestamp then
        FFP.Notes[note.NoteID] = nil
        FFP.MakeTxToNoteID[note.MakeTx] = nil
        msg.reply({Action= 'Reject', Error = 'err_expired', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    local pool = getPool(note.HolderAssetID, note.AssetID)
    local ok, err = validateAmountOut(pool, note.HolderAssetID, note.HolderAmount, note.AssetID, note.Amount)
    if not ok then
        msg.reply({Action= 'Reject', Error = err, SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    msg.reply({Action = 'ExecuteStarted', SettleID = msg.SettleID, NoteID = msg.NoteID})

    FFP.Executing = true
    note.Status = 'Executing'

    Send({Target = note.AssetID, Action = 'Transfer', Quantity = note.Amount, Recipient = FFP.Settle, 
      ['X-FFP-SettleID'] = msg.SettleID, 
      ['X-FFP-NoteID'] = msg.NoteID,
      ['X-FFP-For'] = 'Execute'
    })
  end
)
```

</details>

Triggered by the FFP settlement center for Note settlement operations.

Logic:

* Validate the Note and pool state.
* Use validateAmountOut to verify if the Note matches the AMM market-making price.
* Transfer the assets specified in the Note to the FFP settlement center for settlement.

**Settlement Completion Notification**

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.done',
    function(msg) return (msg.Action == 'Credit-Notice') and (msg['X-FFP-For'] == 'Settled' or msg['X-FFP-For'] == 'Refund') end,
    function(msg)
        assert(msg.Sender == FFP.Settle, 'Only settle can send settled or refund notice')
        
        local noteID = msg['X-FFP-NoteID']
        local note = FFP.Notes[noteID]
        if not note then
            print('no note found when settled: ' .. noteID)
            return 
        end

        local pool = getPool(note.HolderAssetID, note.AssetID)
        if msg['X-FFP-For'] == 'Settled' then
            pool:updateAfterSwap(note.HolderAssetID, note.HolderAmount, note.AssetID, note.Amount)
        end
        FFP.Notes[noteID].Status = msg['X-FFP-For']
        FFP.Executing = false

        FFP.Notes[noteID] = json.decode(Send({Target = FFP.Settle, Action = 'GetNote', NoteID = noteID}).receive().Data)
        
    end
)
```

</details>

Listens for asset transfers or refunds (in case of settlement failure) from the FFP settlement center.

Logic:

* Check if the Note exists.
* Ensure the related pool exists.
* For successful settlements, update the pool’s market-making liquidity.
* Update relevant variables.

#### 3. Query Interfaces

**Query Pools:** Returns information about all created pools.

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.pools', 'FFP.GetPools', 
    function(msg)
        local pools = {}
        for poolId, pool in pairs(FFP.Pools) do
            pools[poolId] = pool:info()
        end
        msg.reply({Data = json.encode(pools)})
    end
)
```

</details>

**Query Notes:** Retrieves a specific Note based on its MakeTx.

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.getNote', 'FFP.GetNote', 
    function(msg)
        assert(type(msg.MakeTx) == 'string', 'MakeTx is required')
        local note = FFP.Notes[FFP.MakeTxToNoteID[msg.MakeTx]] or ''
        msg.reply({NoteID=note.NoteID, Data = json.encode(note)})
    end
)
```

</details>

**Query Prices:** Provides real-time exchange price information.

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.getAmountOut', 'FFP.GetAmountOut',
  function(msg)
    if FFP.Executing then
        msg.reply({Error = 'err_executing'})
        return
    end
    
    local ok, err = validateOrderMsg(msg)
    if not ok then
        msg.reply({Error = err})
        return
    end

    local pool = getPool(msg.TokenIn, msg.TokenOut)
    local tokenOut, amountOut = pool:getAmountOut(msg.TokenIn, msg.AmountIn)
    if not amountOut or amountOut == '0' then
        msg.reply({Error = 'err_no_amount_out'})
        return
    end
    msg.reply({
        TokenIn = msg.TokenIn,
        AmountIn = msg.AmountIn,
        TokenOut = tokenOut,
        AmountOut = tostring(amountOut)
    })
  end
)
```

</details>

### Complete Code

<details>

<summary>amm.lua</summary>

```lua
local json = require('json')
local bint = require('.bint')(512)
local utils = require('.utils')

local poolUniswapV2 = require('.algo.uniswapv2')
local poolBigOrder = require('.algo.bigorder')

FFP = FFP or {}
-- config
FFP.Settle = FFP.Settle or 'rKpOUxssKxgfXQOpaCq22npHno6oRw66L3kZeoo_Ndk'
FFP.Publisher = FFP.Publisher or '8VzqSX_0rdAr99P3QIYd0bRG-XySTQUNFlfJf20zNEo'
FFP.SettleVersion = FFP.SettleVersion or '0.31'
FFP.MaxNotesToSettle = FFP.MaxNotesToSettle or 2
FFP.TimeoutPeriod = FFP.TimeoutPeriod or 90000

-- database or state
FFP.Pools = FFP.Pools or {}
FFP.Notes = FFP.Notes or {}
FFP.MakeTxToNoteID = FFP.MakeTxToNoteID or {}
FFP.Executing = FFP.Executing or false

-- amm pool functions
FFP.AlgoToPool = FFP.AlgoToPool or {
    UniswapV2 = poolUniswapV2,
    BigOrder = poolBigOrder
}

local function getPoolId(X, Y)
    if X > Y then
        return Y .. ':' .. X
    end
    return X .. ':' .. Y
end

local function getPool(X, Y)
    return FFP.Pools[getPoolId(X, Y)]
end

local function validateAmount(amount)
    if not amount then
        return false, 'err_no_amount'
    end
    local ok, qty = pcall(bint, amount)
    if not ok then
        return false, 'err_invalid_amount'
    end
    if not bint.__lt(0, qty) then
        return false, 'err_negative_amount'
    end
    return true, nil
end

-- tokenIn, tokenOut, amountIn, amountOut(optional)
local function validateOrderMsg(msg)
    local Pool = getPool(msg.TokenIn, msg.TokenOut)
    if not Pool then
        return false, 'err_pool_not_found'
    end
    
    local ok, err = validateAmount(msg.AmountIn)
    if not ok then
        return false, 'err_invalid_amount_in'
    end
    
    if msg.AmountOut then
        local ok, err = validateAmount(msg.AmountOut)
        if not ok then
            return false, 'err_invalid_amount_out'
        end
    end

    return true, nil
end

local function validateAmountOut(pool, tokenIn, amountIn, tokenOut, expectedAmountOut)
    local tokenOut_, amountOut = pool:getAmountOut(tokenIn, amountIn)
    if not amountOut or amountOut == '0' then
        return false, 'err_no_amount_out'
    end
    if tokenOut_ ~= tokenOut then
        return false, 'err_invalid_token_out'
    end
    if expectedAmountOut then
        if utils.lt(amountOut, expectedAmountOut) then
            return false, 'err_amount_out_too_small'
        end
    end 
    return true, nil
end

Handlers.add('ffp.withdraw', 'FFP.Withdraw',
  function(msg)
    assert(msg.From == Owner or msg.From == ao.id, 'Only owner can withdraw')
    assert(type(msg.Token) == 'string', 'Token is required')
    assert(type(msg.Amount) == 'string', 'Amount is required')
    assert(bint.__lt(0, bint(msg.Amount)), 'Amount must be greater than 0')
    
    Send({ 
      Target = msg.Token, 
      Action = 'Transfer', 
      Quantity = msg.Amount, 
      Recipient = Owner
    })
  end
)

Handlers.add('ffp.pools', 'FFP.GetPools', 
    function(msg)
        local pools = {}
        for poolId, pool in pairs(FFP.Pools) do
            pools[poolId] = pool:info()
        end
        msg.reply({Data = json.encode(pools)})
    end
)

Handlers.add('ffp.addPool', 'FFP.AddPool', 
    function(msg)
        assert(msg.From == Owner or msg.From == ao.id, 'Only owner can add pool')
        assert(type(msg.X) == 'string', 'X is required')
        assert(type(msg.Y) == 'string', 'Y is required')
        assert(msg.X < msg.Y, 'X must be less than Y')
        assert(type(msg.Algo) == 'string', 'Algo is required')
        assert(type(msg.Data) == 'string', 'Params is required')

        local poolId = getPoolId(msg.X, msg.Y)
        if FFP.Pools[poolId] then
            msg.reply({Error = 'Pool already exists'})
            return
        end

        local P = FFP.AlgoToPool[msg.Algo]
        if not P then
            msg.reply({Error = 'Unsupported amm algo'})
            return
        end
        
        local pool = P.new(msg.X, msg.Y, json.decode(msg.Data))
        if not pool then
            msg.reply({Error = 'Invalid pool params'})
            return
        end

        FFP.Pools[poolId] = pool
        msg.reply({Action = 'PoolAdded-Notice', Pool = poolId})
        Send({Target = FFP.Publisher, Action = 'PoolAdded-Notice', Creator = ao.id, Data = json.encode(pool)})
    end
)

Handlers.add('ffp.updateLiquidity', 'FFP.UpdateLiquidity', 
    function(msg)
        assert(msg.From == Owner or msg.From == ao.id, 'Only owner can update liquidity')
        assert(type(msg.Y) == 'string', 'Y is required')
        assert(msg.X < msg.Y, 'X must be less than Y')
        assert(type(msg.Data) == 'string', 'Params is required')
        
        local pool = getPool(msg.X, msg.Y)
        if not pool then
            msg.reply({Error = 'Pool not found'})
            return
        end
        
        local ok = pool:updateLiquidity(json.decode(msg.Data))
        if not ok then
            msg.reply({Error = 'Invalid updateLiquidity params'})
            return
        end

        msg.reply({Action = 'LiquidityUpdated-Notice', Data = json.encode(pool)})
    end
)

Handlers.add('ffp.getAmountOut', 'FFP.GetAmountOut',
  function(msg)
    if FFP.Executing then
        msg.reply({Error = 'err_executing'})
        return
    end
    
    local ok, err = validateOrderMsg(msg)
    if not ok then
        msg.reply({Error = err})
        return
    end

    local pool = getPool(msg.TokenIn, msg.TokenOut)
    local tokenOut, amountOut = pool:getAmountOut(msg.TokenIn, msg.AmountIn)
    if not amountOut or amountOut == '0' then
        msg.reply({Error = 'err_no_amount_out'})
        return
    end
    msg.reply({
        TokenIn = msg.TokenIn,
        AmountIn = msg.AmountIn,
        TokenOut = tokenOut,
        AmountOut = tostring(amountOut)
    })
  end
)

Handlers.add('ffp.makeOrder', 'FFP.MakeOrder',
  function(msg)
    if FFP.Executing then
        msg.reply({Error = 'err_executing'})
        return
    end
    
    local ok, err = validateOrderMsg(msg)
    if not ok then
        msg.reply({Error = err})
        return
    end

    local pool = getPool(msg.TokenIn, msg.TokenOut)
    local tokenOut, amountOut = pool:getAmountOut(msg.TokenIn, msg.AmountIn)
    if not amountOut or amountOut == '0' then
        msg.reply({Error = 'err_no_amount_out'})
        return
    end

    if msg.AmountOut then
        local ok, err = validateAmountOut(pool, msg.TokenIn, msg.AmountIn, msg.TokenOut, msg.AmountOut)
        if not ok then
            msg.reply({Error = err})
            return
        end
        amountOut = msg.AmountOut
    end

    -- 90 seconds
    local expireDate = msg.Timestamp + FFP.TimeoutPeriod

    local res = Send({
        Target = FFP.Settle,
        Action = 'CreateNote',
        AssetID = msg.TokenOut,
        Amount = tostring(amountOut),
        HolderAssetID = msg.TokenIn,
        HolderAmount = msg.AmountIn,
        IssueDate = tostring(msg.Timestamp),
        ExpireDate = tostring(expireDate),
        Version = FFP.SettleVersion
    }).receive()
    local noteID = res.NoteID
    local note = json.decode(res.Data)
    note.MakeTx = msg.Id
    FFP.Notes[noteID] = note
    FFP.MakeTxToNoteID[msg.Id] = noteID
    -- remove expired notes in Notes
    for noteID, note in pairs(FFP.Notes) do
        if note.Status == 'Open' and note.ExpireDate and note.ExpireDate < msg.Timestamp then
            FFP.Notes[noteID] = nil
            FFP.MakeTxToNoteID[note.MakeTx] = nil
        end
    end

    msg.reply({
        Action = 'OrderMade-Notice',
        NoteID = noteID,
        Data = json.encode(note)
    })
  end
)

Handlers.add('ffp.getNote', 'FFP.GetNote', 
    function(msg)
        assert(type(msg.MakeTx) == 'string', 'MakeTx is required')
        local note = FFP.Notes[FFP.MakeTxToNoteID[msg.MakeTx]] or ''
        msg.reply({NoteID=note.NoteID, Data = json.encode(note)})
    end
)

Handlers.add('ffp.execute', 'Execute',
  function(msg)
    assert(msg.From == FFP.Settle, 'Only settle can start exectue')
    assert(type(msg.NoteID) == 'string', 'NoteID is required')
    assert(type(msg.SettleID) == 'string', 'SettleID is required')

    if FFP.Executing then
        msg.reply({Action= 'Reject', Error = 'err_executing', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    local note = FFP.Notes[msg.NoteID]
    if not note then
        msg.reply({Action= 'Reject', Error = 'err_not_found', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.Status ~= 'Open' then
        msg.reply({Action= 'Reject', Error = 'err_not_open', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.Issuer ~= ao.id then
        msg.reply({Action = 'Reject', Error = 'err_invalid_issuer', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end
    if note.ExpireDate and note.ExpireDate < msg.Timestamp then
        FFP.Notes[note.NoteID] = nil
        FFP.MakeTxToNoteID[note.MakeTx] = nil
        msg.reply({Action= 'Reject', Error = 'err_expired', SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    local pool = getPool(note.HolderAssetID, note.AssetID)
    local ok, err = validateAmountOut(pool, note.HolderAssetID, note.HolderAmount, note.AssetID, note.Amount)
    if not ok then
        msg.reply({Action= 'Reject', Error = err, SettleID = msg.SettleID, NoteID = msg.NoteID})
        return
    end

    msg.reply({Action = 'ExecuteStarted', SettleID = msg.SettleID, NoteID = msg.NoteID})

    FFP.Executing = true
    note.Status = 'Executing'

    Send({Target = note.AssetID, Action = 'Transfer', Quantity = note.Amount, Recipient = FFP.Settle, 
      ['X-FFP-SettleID'] = msg.SettleID, 
      ['X-FFP-NoteID'] = msg.NoteID,
      ['X-FFP-For'] = 'Execute'
    })
  end
)

Handlers.add('ffp.done',
    function(msg) return (msg.Action == 'Credit-Notice') and (msg['X-FFP-For'] == 'Settled' or msg['X-FFP-For'] == 'Refund') end,
    function(msg)
        assert(msg.Sender == FFP.Settle, 'Only settle can send settled or refund notice')
        
        local noteID = msg['X-FFP-NoteID']
        local note = FFP.Notes[noteID]
        if not note then
            print('no note found when settled: ' .. noteID)
            return 
        end

        local pool = getPool(note.HolderAssetID, note.AssetID)
        if msg['X-FFP-For'] == 'Settled' then
            pool:updateAfterSwap(note.HolderAssetID, note.HolderAmount, note.AssetID, note.Amount)
        end
        FFP.Notes[noteID].Status = msg['X-FFP-For']
        FFP.Executing = false

        FFP.Notes[noteID] = json.decode(Send({Target = FFP.Settle, Action = 'GetNote', NoteID = noteID}).receive().Data)
        
    end
)
```

</details>

<details>

<summary>utils.lua</summary>

```lua
local json = require('json')
local bint = require('.bint')(512)

local mod = {
  add = function(a, b)
      return tostring(bint(a) + bint(b))
  end,
  subtract = function(a, b)
      return tostring(bint(a) - bint(b))
  end,
  toBalanceValue = function(a)
      return tostring(bint(a))
  end,
  toNumber = function(a)
      return tonumber(a)
  end,
  lt = function (a, b)
      return bint.__lt(bint(a), bint(b))
  end
}

mod.getNoteIDs = function(data)
  if string.find(data, "null") then
    return nil
  end
  
  local noteIDs, err = json.decode(data)
  if err then
    return nil
  end

  if type(noteIDs) == "string" then
    return {noteIDs}
  end

  if type(noteIDs) == "table" then
    for i = 1, #noteIDs do
      if noteIDs[i] == nil or type(noteIDs[i]) ~= "string" then
        return nil
      end
    end
    return noteIDs
  end

  return nil
end

mod.SettlerIO = function (notes)
  local settlerIn = {}
  local settlerOut = {}

  local bc = {}
  for _, note in ipairs(notes) do
      if not bc[note.AssetID] then bc[note.AssetID] = bint(0) end
      if not bc[note.HolderAssetID] then bc[note.HolderAssetID] = bint(0) end
      bc[note.AssetID] = mod.add(bc[note.AssetID], note.Amount)
      bc[note.HolderAssetID] = mod.subtract(bc[note.HolderAssetID], note.HolderAmount)
  end
  
  for k, v in pairs(bc) do
      if v == '0' then bc[k] = nil end
  end

  for k, v in pairs(bc) do
      if mod.lt(v, '0') then
          settlerOut[k] = v
      else
          settlerIn[k] = v
      end
  end

  return settlerIn, settlerOut
end
    
return mod

```

</details>

<details>

<summary>./algo/bigorder.lua</summary>

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

<details>

<summary>./algo/uniswapv2.lua</summary>

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

local function getAmountOut(amountIn, reserveIn, reserveOut, fee)
    local amountInWithFee = utils.mul(amountIn, utils.subtract(10000, fee))
    local numerator = utils.mul(amountInWithFee, reserveOut)
    local denominator = utils.add(utils.mul(10000, reserveIn), amountInWithFee)
    return utils.div(numerator, denominator)
end

local function validateInitParams(params)
    if not params then
        return false
    end
    return validateAmount(params.fee) and validateAmount(params.px) and validateAmount(params.py)
end

local function validateUpdateLiquidityParams(params)
    if not params then
        return false
    end
    return validateAmount(params.px) and validateAmount(params.py)
end

function Pool.new(x, y, params)
    if not validateInitParams(params) then
        return nil
    end
    local self = setmetatable({}, Pool)
    self.algo = 'UniswapV2'
    self.x = x
    self.y = y
    self.fee = params.fee
    self.px = params.px
    self.py = params.py
    return self
end

function Pool:balances()
    local balance = {
        [self.x] = self.px,
        [self.y] = self.py
    }
    return balance
end

function Pool:info()
    local info = {
        x = self.x,
        y = self.y,
        fee = self.fee,
        algo = self.algo,
        px = self.px,
        py = self.py,
        balances = self:balances()
    }
    return info
end

function Pool:updateLiquidity(params)
    if not validateUpdateLiquidityParams(params) then
        return false
    end
    self.px = params.px
    self.py = params.py
    return true
end

function Pool:updateAfterSwap(tokenIn, amountIn, tokenOut, amountOut)
    if tokenIn == self.x then
        self.px = utils.add(self.px, amountIn)
        self.py = utils.subtract(self.py, amountOut)
    else
        self.py = utils.add(self.py, amountIn)
        self.px = utils.subtract(self.px, amountOut)
    end
end

function Pool:getAmountOut(tokenIn, amountIn)
    local tokenOut = self.y
    local reserveIn = bint(self.px)
    local reserveOut = bint(self.py)
    if tokenIn == self.y then
        reserveIn, reserveOut = reserveOut, reserveIn
        tokenOut = self.x
    end
    local amountOut = getAmountOut(bint(amountIn), reserveIn, reserveOut, bint(self.fee))
    return tokenOut, tostring(amountOut)
end

return Pool

```

</details>

Before starting development, copy the above code into your development environment as per the specified paths.

### Summary

The **AMM Blueprint** uses a flexible architectural design to enable developers to easily implement and extend various AMM functionalities. It ensures high efficiency in transaction handling and liquidity management, making it a robust solution for building custom AMM applications.
