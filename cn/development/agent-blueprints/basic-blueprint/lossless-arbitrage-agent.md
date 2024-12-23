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

# Lossless Arbitrage Agent

The **Lossless Arbitrage Agent** is an extension of the **Basic Blueprint.** It can carry out arbitrage operations within the FFP protocol by leveraging price differences; all without without requiring upfront capital costs.

Below is an example implementation using the **wAR/wUSDC** trading pair.

### Target Scenario

1. The Agent listens for exchange orders of the **wAR/wUSDC** trading pair in the FFP protocol:
   1. The Agent fetches pending wAR/wUSDC orders from the FFP settlement center in real time.
   2. It compares the price of each order with the price from the **wAR/wUSDC AMM Agent**.
2. If there is a price difference, it performs arbitrage operations：
   1. **Order Example**: A user places an order to exchange 1 wAR for 20 wUSDC.
   2. **AMM Price:** The current price in the AMM Agent is 1 wAR = 21 wUSDC.
   3. **Arbitrage Process:**
      1. Request the AMM Agent to create a **21 wUSDC -> 1 wAR** order in FFP.
      2. Submit the user’s order as a counter-order to the AMM-created order in the FFP settlement center.
      3. The settlement center collects **1 wAR from the user** and **21 wUSDC from the AMM Agent.**
      4. It distributes **20 wUSDC to the user** and **1 wAR to the AMM Agent**, leaving a surplus of **1 wUSDC**.
      5. The surplus **1 wUSDC** is sent to the Arbitrage Agent as a reward, since the agent initiated the settlement.

### Arbitrage Agent Code Explanation



<details>

<summary>arbitrage.lua</summary>

```lua
local json = require('json')
local utils = require('.utils')
local bint = require('.bint')(512)

ProcessCfg = {
    PageSize = '10',

    PoolPid = 'vJY-ed1Aoa0pGgQ30BcpO9ehGBu1PfNHUlwV9W8_n5A', -- wUSDC/wAR
    wUSDCPid = '7zH9dlMNoxprab9loshv3Y7WG45DOny_Vrq9KrXObdQ', -- Y
    wARPid = 'xU9zFkq3X2ZQ6olwNVvr1vUWIjc3kXTWr7xKQD6dh10' -- X
}

--[[
example-notes:
    [
        {
            "Price": 0.000023,
            "IssueDate": 1732447831435,
            "Status": "Open",
            "ID": 2303,
            "HolderAssetID": "7zH9dlMNoxprab9loshv3Y7WG45DOny_Vrq9KrXObdQ",
            "AssetID": "xU9zFkq3X2ZQ6olwNVvr1vUWIjc3kXTWr7xKQD6dh10",
            "Amount": "50000000000",
            "Issuer": "ZsXb0JKGATswuk-qRs3iV49VbTbaiaMCEPYqk6GKdW0",
            "HolderAmount": "1150000",
            "NoteID": "ssfaQ3p3Ufj1rcKj1ZXVW3aYYW2zhcfAQ8vvIkSOsD4"
        }
    ]
-- ]]
function getNotesFromSettle(assetID, holderAssetID, page)
    print('getNotesFromSettle...'..assetID)
    local res = Send({
        Target = FFP.Settle,
        Action = 'GetNotes',
        AssetID = assetID,
        HolderAssetID = holderAssetID,
        Order = 'desc',
        Status = 'Open',
        Page = tostring(page),
        PageSize = ProcessCfg.PageSize
    }).receive()
    local notes = json.decode(res.Data)
    print('got notes: '.. #notes)
    return notes
end

function getPoolInfo()
    print('getPoolInfo: ' .. ProcessCfg.PoolPid)
    local res = Send({
        Target = ProcessCfg.PoolPid,
        Action = 'Info',
    }).receive()
    if not res then
        print('Error calling Send: ')
        return {}
    end
    print('got poolInfo')
    return {
        ["Executing"] = res.Tags.Executing,
        ["PX"] = res.Tags.PX,
        ["PY"] = res.Tags.PY,
        ["X"] = res.Tags.X,
        ["Y"] = res.Tags.Y,
        ["Fee"] = res.Tags.Fee
    }
end

-- return amountOut
local function calcAmmInputPrice(amountIn, reserveIn, reserveOut, Fee)
    local amountInWithFee = bint.__mul(amountIn, bint.__sub(10000, Fee))
    local numerator = bint.__mul(amountInWithFee, reserveOut)
    local denominator = bint.__add(bint.__mul(10000, reserveIn), amountInWithFee)
    return bint.udiv(numerator, denominator)
end

-- makeOrder for amm pool
local function makeOrderForAmmPool(tokenIn, amountIn, tokenOut, amountOut)
    print('makeOrderForAmmPool...')
    local res = Send({
        Target = ProcessCfg.PoolPid,
        Action = 'MakeOrder',
        TokenIn = tokenIn,
        AmountIn = amountIn,
        TokenOut = tokenOut,
        AmountOut = amountOut
    }).receive()
    if res.Error then
        return {
            ["Error"] = res.Error
        }
    end
    local note = json.decode(res.Data)
    return {
        ['Note'] = note
    }
end

local function takeOrder(noteIDs)
    -- start Settle
    local res = Send({
        Target = ao.id,
        Action = 'FFP.TakeOrder',
        Data = json.encode(noteIDs)
    }).receive()
    if res.Error then
        return {
            ["Error"] = res.Error
        }
    end
    return {}
end

-- arbitrageExecutor
function perNoteArbitrageExecutor(note)
    local tokenIn = note.AssetID
    local amountIn = note.Amount
    local tokenOut = note.HolderAssetID
    local expectedAmountOut = note.HolderAmount

    local poolInfo = getPoolInfo()
    if poolInfo.Executing == 'true' then
        print('ammPool is executing...')
        return {
            ["Error"] = 'err_pool_executing'
        }
    end

    local reserveIn = poolInfo.PX
    local reserveOut = poolInfo.PY
    if tokenIn == poolInfo.Y then
        reserveIn, reserveOut = reserveOut, reserveIn
    end

    local amountOut = calcAmmInputPrice(bint(amountIn), bint(reserveIn), bint(reserveOut), bint(poolInfo.Fee))
    if utils.lt(amountOut, expectedAmountOut) then
        print('amount out too small')
        return {
            ['Error'] = 'amm_prices_low'
        } -- can not settle, because pool price not ok
    end

    -- could process settle
    -- makeOrder for amm pool
    local ord = makeOrderForAmmPool(tokenIn, amountIn, tokenOut, tostring(amountOut))
    if ord.Error then
        print('makeOrderForAmmPool failed: ' .. ord.Error)
        return {
            ['Error'] = ord.Error
        }
    end

    print('amm makeorder note: '.. ord.Note.NoteID)
    -- calc settle income and outcome 
    local si, so = utils.SettlerIO({ord.Note, note})
 
    if next(so) then
        print('some calc error, settlerout must be nil')
        return {
            ["Error"] = 'err_settler_out'
        }
    end
    -- take order use basic api 
    print('Taking order for NoteIDs: ' .. json.encode({ord.Note.NoteID, note.NoteID}))
    local res = takeOrder({ord.Note.NoteID, note.NoteID})
    if res.Error then
        print('take order failed: ' .. res.Error)
        return {
            ['Error'] = res.Error
        }
    end
    return {
        ['SettlerIn'] = si
    }
end

function arbitrageExec(assetID, holderAssetID)
    local page = 1
    while true do
        -- get wUSDC swap wAR notes
        local notes = getNotesFromSettle(assetID, holderAssetID, page)
        if #notes == 0 then
            print('not notes...')
            return
        end

        for _, note in ipairs(notes) do
            -- execute per note
            local res = perNoteArbitrageExecutor(note)
            if res.Error then
                print('perNoteArbitrageExecutor failed: '..res.Error..' noteID: '.. note.NoteID)
            else
                print('success arbitrage note: '.. note.NoteID .. ' settlerIn: ' .. json.encode(res.SettlerIn))
            end
        end

        if #notes < tonumber(ProcessCfg.PageSize) then
            break
        end
        page = page + 1
    end
end

function ArbitrageExecutor()
    print('start wAR-wUSDC notes arbitrage...')
    arbitrageExec(ProcessCfg.wARPid,ProcessCfg.wUSDCPid)
    print('end wAR-wUSDC pairs arbitrage...')
    print('start wUSDC-wAR notes arbitrage...')
    arbitrageExec(ProcessCfg.wUSDCPid,ProcessCfg.wARPid)
    print('end wUSDC-wAR notes arbitrage...')
end




return {
    ArbitrageExecutor = ArbitrageExecutor
}

```

</details>

<details>

<summary>agent.lua</summary>

```lua
local json = require('json')
local bint = require('.bint')(512)
local utils = require('.utils')
local agent = require('.arbAgent')

FFP = FFP or {}
-- config
FFP.Settle = FFP.Settle or 'rKpOUxssKxgfXQOpaCq22npHno6oRw66L3kZeoo_Ndk'
FFP.Version = FFP.Version or '0.31'
FFP.MaxNotesToSettle = FFP.MaxNotesToSettle or 2

-- database
FFP.Notes = FFP.Notes or {}
FFP.Settled = FFP.Settled or {}

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

Handlers.add('ffp.takeOrder', 'FFP.TakeOrder',
  function(msg)
    assert(msg.From == Owner or msg.From == ao.id, 'Only owner can make order')
    
    local noteIDs = utils.getNoteIDs(msg.Data)
    if noteIDs == nil then
      msg.reply({Error = 'err_invalid_note_ids', ['X-FFP-TakeOrderID'] = msg.Id})
      return
    end
    
    if #noteIDs > FFP.MaxNotesToSettle then
      msg.reply({Error = 'err_too_many_orders', ['X-FFP-TakeOrderID'] = msg.Id})
      return
    end

    local notes = {}
    for _, noteID in ipairs(noteIDs) do
      local data = Send({Target = FFP.Settle, Action = 'GetNote', NoteID = noteID}).receive().Data
      if data == '' then
        msg.reply({Error = 'err_not_found', ['X-FFP-TakeOrderID'] = msg.Id})
        return
      end
      local note = json.decode(data)
      if note.Status ~= 'Open' then
        msg.reply({Error = 'err_not_open', ['X-FFP-TakeOrderID'] = msg.Id, Data = noteID})
        return
      end
      if note.Issuer == ao.id then
        msg.reply({Error = 'err_cannot_take_self_order', ['X-FFP-TakeOrderID'] = msg.Id, Data = noteID})
        return
      end
      if note.ExpireDate and note.ExpireDate < msg.Timestamp then
        msg.reply({Error = 'err_expired', ['X-FFP-TakeOrderID'] = msg.Id, Data = noteID})
        return
      end
      table.insert(notes, note)
    end
    
    local si, so = utils.SettlerIO(notes)
    -- todo: make sure we have enough balance

    -- start a settle session
    local res = Send({Target = FFP.Settle, Action = 'StartSettle', Version = FFP.Version, Data = json.encode(noteIDs)}).receive()
    if res.Error then
      msg.reply({Error = res.Error, ['X-FFP-TakeOrderID'] = msg.Id})     
      return
    end
   
    local settleID = res.SettleID
    FFP.Settled[settleID] = {SettleID = settleID, NoteIDs = noteIDs, Status = 'Pending'}
   
    if next(so) == nil then
      print('TakeOrder: Settler no need to transfer to settle process')
      Send({Target = FFP.Settle, Action = 'Settle', Version = FFP.Version, SettleID = settleID})
      msg.reply({SettleID = settleID})
      return
    end

    for k, v in pairs(so) do
      local amount = utils.subtract('0', v)
      Send({Target = k, Action = 'Transfer', Quantity = amount,  Recipient = FFP.Settle, 
        ['X-FFP-SettleID'] = settleID, 
        ['X-FFP-For'] = 'Settle'
      })
    end
    msg.reply({SettleID = settleID})
  end
)

Handlers.add('ffp.done',
  function(msg) 
    return (msg.Action == 'Credit-Notice') and (msg['X-FFP-For'] == 'Settled' or msg['X-FFP-For'] == 'Refund') 
  end,
  function(msg)
    if msg.Sender ~= FFP.Settle then
      print('Only settle can send settled notice')
      return
    end

    local settleID = msg['X-FFP-SettleID']
    if settleID and FFP.Settled[settleID] then
      if msg['X-FFP-For'] == 'Settled' then
        FFP.Settled[settleID].Status = 'Settled'
      else
        FFP.Settled[settleID].Status = 'Rejected'
      end
      FFP.Settled[settleID].SettledDate = msg['X-FFP-SettledDate']
      for _, noteID in ipairs(FFP.Settled[settleID].NoteIDs) do
        local data = Send({Target = FFP.Settle, Action = 'GetNote', NoteID = noteID}).receive().Data
        FFP.Notes[noteID] = json.decode(data)
      end
    else
      print('SettleID not found: ' .. settleID)
    end
  end
)

Handlers.add('ffp.getOrders', 'FFP.GetOrders',
  function (msg)
    msg.reply({Action = 'GetOrders-Notice', Data = json.encode(FFP.Notes)})
  end
)

Handlers.add('ffp.getSettled', 'FFP.GetSettled',
  function (msg)
    msg.reply({Action = 'GetSettled-Notice', Data = json.encode(FFP.Settled)})
  end
)

-- Trigger
ExecuteState = ExecuteState or 'ON'
Handlers.add(
    "CronTick",
    Handlers.utils.hasMatchingTag("Action", "Cron"), -- Send({Target=ao.id,Action='Cron'})
    function()
        print('Cron trigger arbitrage ...')
        if ExecuteState == 'ON' then
            -- find note: wUSDC swap wAR
            ExecuteState = 'OFF'
            agent.ArbitrageExecutor()
            ExecuteState = 'ON'
            print('finished..')
            agent.Withdrawal()
            print('finish withdrawal...')
        else
          print('ExecuteState is off...')
        end
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

#### Code Structure

* **arbitrage.lua:** Contains all execution logic for arbitrage.
* **agent.lua:** Extends basic.lua with arbitrage-specific methods.
* **utils.lua:** A direct copy from the Basic Blueprint.

#### Core Methods Explained

#### 1. ArbitrageExecutor

The main arbitrage scheduler that orchestrates the process:

* Executes arbitrage for wAR -> wUSDC orders first.
* Then processes wUSDC -> wAR orders to cover both arbitrage directions.

#### 2. arbitrageExec

Executes arbitrage for all orders in a given direction:

* Takes two asset types as input parameters to determine the arbitrage direction.
* Fetches order data (Notes) from FFP in batches.
* Performs arbitrage for each Note.

#### 3. getNotesFromSettle

A dedicated function to fetch Notes from the settlement process:

* Filters Notes by asset type and status.
* Retrieves the necessary Notes from the FFP settlement center.

#### 4. perNoteArbitrageExecutor

Executes arbitrage logic for a single Note:

* **AMM Liquidity Check:** Fetches liquidity information from the AMM Agent to ensure the pool is available and unlocked.
* **Price Comparison:**
  * If the AMM price is lower than the Note’s target price, arbitrage is not feasible.
  * If the AMM price is higher, the agent requests the AMM Agent to create a counter-order.
* **Example:**
  * A Note represents **1 wAR -> 20 wUSDC**.
  * The AMM offers **21 wUSDC -> 1 wAR**.
  * The agent requests the AMM to create this counter-order, resulting in a price difference of **1 wUSDC.**
* **Arbitrage Settlement:**
  * Calls the takeOrder method (implemented in the Basic Blueprint) to initiate settlement.
  * After settlement is complete, the agent automatically collects the arbitrage profit (the price difference).

#### 5. calcAmmInputPrice

This method calculates the exchange price based on the constant product formula (UniswapV2 algorithm) and fee structure:

* Fetches the reserveIn, reserveOut, and Fee parameters from the AMM pool.
* Extracts the input amount (AmountIn) from the Note.
* Computes the exchangeable AmountOut using AMM formulas.
* Compares AmountOut with the target amount in the Note.
* If AmountOut exceeds the target amount, arbitrage is feasible, and the operation proceeds.

### Summary

The **Lossless Arbitrage Agent** provides a practical example of leveraging the Basic Blueprint to implement a custom, automated arbitrage strategy within the FFP protocol. By exploiting price differences across the ecosystem, developers can create efficient and profitable agents while integrating seamlessly into the FusionFi financial activities.
