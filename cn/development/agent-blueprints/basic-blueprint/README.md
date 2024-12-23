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

# Basic Blueprint

The **Basic Blueprint** is a pre-designed template that helps you quickly build a basic version of an Agent, which can then be customized as needed.

### Overview of the Basic Blueprint

#### Core Features of the Basic Blueprint:

* [**Withdraw Operation**](./#id-1.-withdraw-operation): Allows the Agent owner to withdraw specified assets from the Agent.
* [**Take Order Settlement**](./#id-2.-take-order-and-settle): Enables acquiring specific notes from the FFP system, validating their validity, and submitting them for settlement.
* [**Settlement Completion Notification**](./#id-3.-settlement-completion-notification)**:** Monitors settlement notifications from the FFP settlement center and updates the notes and settlement status accordingly.

#### Code Structure of the Basic Blueprint

Below is an outline of the basic.lua file:

```lua
local json = require('json')
local bint = require('.bint')(512)
local utils = require('.utils')

-- Agent FFP related configuration, customizable
FFP = FFP or {}
FFP.Settle = FFP.Settle or 'rKpOUxssKxgfXQOpaCq22npHno6oRw66L3kZeoo_Ndk' -- Specify the FFP settlement center
FFP.Version = FFP.Version or '0.31'   -- FFP version number
FFP.MaxNotesToSettle = FFP.MaxNotesToSettle or 2 -- Maximum number of notes to settle

-- Data storage
FFP.Notes = FFP.Notes or {}   -- Store notes
FFP.Settled = FFP.Settled or {} -- Store settled notes

-- Withdraw operation
Handlers.add('ffp.withdraw', 'FFP.Withdraw', function(msg)
  -- Withdraw logic
end)

-- Take order and settle
Handlers.add('ffp.takeOrder', 'FFP.TakeOrder', function(msg)
  -- Take order and settle logic
end)

-- Settlement completion notification operation
Handlers.add('ffp.done', function(msg)
  -- Settlement completion notification
end, function(msg)
  -- Settlement status update
end)
```

### Code Explanation

#### 1. Withdraw Operation

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.withdraw', 'FFP.Withdraw', function(msg)
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
end)
```

</details>

This function allows the Agent's owner to withdraw specified assets from the Agent.

Typical Scenarios:

* The Agent owner withdraws profits.
* Periodically clearing out remaining assets in the Agent.

#### 2. Take Order and Settle

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.takeOrder', 'FFP.TakeOrder', function(msg)
    assert(msg.From == Owner or msg.From == ao.id, 'Only owner can take order')

    local noteIDs = utils.getNoteIDs(msg.Data)
    if noteIDs == nil then
        msg.reply({
            Error = 'err_invalid_note_ids',
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    if #noteIDs > FFP.MaxNotesToSettle then
        msg.reply({
            Error = 'err_too_many_orders',
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    local notes = {}
    for _, noteID in ipairs(noteIDs) do
        local data = Send({
            Target = FFP.Settle,
            Action = 'GetNote',
            NoteID = noteID
        }).receive().Data
        if data == '' then
            msg.reply({
                Error = 'err_not_found',
                ['X-FFP-TakeOrderID'] = msg.Id
            })
            return
        end
        local note = json.decode(data)
        if note.Status ~= 'Open' then
            msg.reply({
                Error = 'err_not_open',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        if note.Issuer == ao.id then
            msg.reply({
                Error = 'err_cannot_take_self_order',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        if note.ExpireDate and note.ExpireDate < msg.Timestamp then
            msg.reply({
                Error = 'err_expired',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        table.insert(notes, note)
    end

    local si, so = utils.SettlerIO(notes)
    -- todo: make sure we have enough balance

    -- start a settle session
    local res = Send({
        Target = FFP.Settle,
        Action = 'StartSettle',
        Version = FFP.SettleVersion,
        Data = json.encode(noteIDs)
    }).receive()
    if res.Error then
        msg.reply({
            Error = res.Error,
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    local settleID = res.SettleID
    FFP.Settled[settleID] = {
        SettleID = settleID,
        NoteIDs = noteIDs,
        Status = 'Pending'
    }

    if next(so) == nil then
        print('TakeOrder: Settler no need to transfer to settle process')
        Send({
            Target = FFP.Settle,
            Action = 'Settle',
            Version = FFP.SettleVersion,
            SettleID = settleID
        })
        msg.reply({
            Action = 'TakeOrder-Settle-Sent-Notice',
            SettleID = settleID
        })
        return
    end

    for k, v in pairs(so) do
        local amount = utils.subtract('0', v)
        Send({
            Target = k,
            Action = 'Transfer',
            Quantity = amount,
            Recipient = FFP.Settle,
            ['X-FFP-SettleID'] = settleID,
            ['X-FFP-For'] = 'Settle'
        })
    end
    msg.reply({
        Action = 'TakeOrder-Settle-Sent-Notice',
        SettleID = settleID
    })
end)
```

</details>

This function enables the Agent to fetch a specific set of notes (Notes) from the FFP protocol, validate them, and submit them to the FFP protocol to create Settlement orders.

Key Code Logic:

* Validate the notes (e.g., status, source, expiration date).
* Call the StartSettle interface in the FFP protocol to generate settlement orders.
* Execute settlement order logic (e.g., funds transfer).

#### 3. Settlement Completion Notification

<details>

<summary>code</summary>

```lua
Handlers.add('ffp.done', function(msg)
    return (msg.Action == 'Credit-Notice') and (msg['X-FFP-For'] == 'Settled' or msg['X-FFP-For'] == 'Refund')
end, function(msg)
    assert(msg.Sender == FFP.Settle, 'Only settle can send settled or refund notice')

    local settleID = msg['X-FFP-SettleID']
    if settleID and FFP.Settled[settleID] then
        if msg['X-FFP-For'] == 'Settled' then
            FFP.Settled[settleID].Status = 'Settled'
        else
            FFP.Settled[settleID].Status = 'Rejected'
        end
        FFP.Settled[settleID].SettledDate = msg['X-FFP-SettledDate']
        for _, noteID in ipairs(FFP.Settled[settleID].NoteIDs) do
            local data = Send({
                Target = FFP.Settle,
                Action = 'GetNote',
                NoteID = noteID
            }).receive().Data
            FFP.Notes[noteID] = json.decode(data)
        end
    else
        print('SettleID not found: ' .. settleID)
    end
end)
```

</details>

This function listens for settlement completion notifications sent by the FFP protocol and updates the status of notes and settlement orders.

Main Purpose: To ensure that the status of notes and settlement orders saved by the Agent remains consistent with the status in the FFP protocol.

### Full Code

<details>

<summary>basic.lua</summary>

```lua
local json = require('json')
local bint = require('.bint')(512)
local utils = require('.utils')

FFP = FFP or {}
-- config
FFP.Settle = FFP.Settle or 'rKpOUxssKxgfXQOpaCq22npHno6oRw66L3kZeoo_Ndk'
FFP.SettleVersion = FFP.SettleVersion or '0.31'
FFP.MaxNotesToSettle = FFP.MaxNotesToSettle or 2

-- database
FFP.Notes = FFP.Notes or {}
FFP.Settled = FFP.Settled or {}

Handlers.add('ffp.withdraw', 'FFP.Withdraw', function(msg)
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
end)

Handlers.add('ffp.takeOrder', 'FFP.TakeOrder', function(msg)
    assert(msg.From == Owner or msg.From == ao.id, 'Only owner can take order')

    local noteIDs = utils.getNoteIDs(msg.Data)
    if noteIDs == nil then
        msg.reply({
            Error = 'err_invalid_note_ids',
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    if #noteIDs > FFP.MaxNotesToSettle then
        msg.reply({
            Error = 'err_too_many_orders',
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    local notes = {}
    for _, noteID in ipairs(noteIDs) do
        local data = Send({
            Target = FFP.Settle,
            Action = 'GetNote',
            NoteID = noteID
        }).receive().Data
        if data == '' then
            msg.reply({
                Error = 'err_not_found',
                ['X-FFP-TakeOrderID'] = msg.Id
            })
            return
        end
        local note = json.decode(data)
        if note.Status ~= 'Open' then
            msg.reply({
                Error = 'err_not_open',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        if note.Issuer == ao.id then
            msg.reply({
                Error = 'err_cannot_take_self_order',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        if note.ExpireDate and note.ExpireDate < msg.Timestamp then
            msg.reply({
                Error = 'err_expired',
                ['X-FFP-TakeOrderID'] = msg.Id,
                Data = noteID
            })
            return
        end
        table.insert(notes, note)
    end

    local si, so = utils.SettlerIO(notes)
    -- todo: make sure we have enough balance

    -- start a settle session
    local res = Send({
        Target = FFP.Settle,
        Action = 'StartSettle',
        Version = FFP.SettleVersion,
        Data = json.encode(noteIDs)
    }).receive()
    if res.Error then
        msg.reply({
            Error = res.Error,
            ['X-FFP-TakeOrderID'] = msg.Id
        })
        return
    end

    local settleID = res.SettleID
    FFP.Settled[settleID] = {
        SettleID = settleID,
        NoteIDs = noteIDs,
        Status = 'Pending'
    }

    if next(so) == nil then
        print('TakeOrder: Settler no need to transfer to settle process')
        Send({
            Target = FFP.Settle,
            Action = 'Settle',
            Version = FFP.SettleVersion,
            SettleID = settleID
        })
        msg.reply({
            Action = 'TakeOrder-Settle-Sent-Notice',
            SettleID = settleID
        })
        return
    end

    for k, v in pairs(so) do
        local amount = utils.subtract('0', v)
        Send({
            Target = k,
            Action = 'Transfer',
            Quantity = amount,
            Recipient = FFP.Settle,
            ['X-FFP-SettleID'] = settleID,
            ['X-FFP-For'] = 'Settle'
        })
    end
    msg.reply({
        Action = 'TakeOrder-Settle-Sent-Notice',
        SettleID = settleID
    })
end)

Handlers.add('ffp.done', function(msg)
    return (msg.Action == 'Credit-Notice') and (msg['X-FFP-For'] == 'Settled' or msg['X-FFP-For'] == 'Refund')
end, function(msg)
    assert(msg.Sender == FFP.Settle, 'Only settle can send settled or refund notice')

    local settleID = msg['X-FFP-SettleID']
    if settleID and FFP.Settled[settleID] then
        if msg['X-FFP-For'] == 'Settled' then
            FFP.Settled[settleID].Status = 'Settled'
        else
            FFP.Settled[settleID].Status = 'Rejected'
        end
        FFP.Settled[settleID].SettledDate = msg['X-FFP-SettledDate']
        for _, noteID in ipairs(FFP.Settled[settleID].NoteIDs) do
            local data = Send({
                Target = FFP.Settle,
                Action = 'GetNote',
                NoteID = noteID
            }).receive().Data
            FFP.Notes[noteID] = json.decode(data)
        end
    else
        print('SettleID not found: ' .. settleID)
    end
end)

Handlers.add('ffp.getOrders', 'FFP.GetOrders', function(msg)
    msg.reply({
        Action = 'GetOrders-Notice',
        Data = json.encode(FFP.Notes)
    })
end)

Handlers.add('ffp.getSettled', 'FFP.GetSettled', function(msg)
    msg.reply({
        Action = 'GetSettled-Notice',
        Data = json.encode(FFP.Settled)
    })
end)

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

### How to Build a Custom FFP Agent Based on the Blueprint

#### 1. Clone the Blueprint

Copy the basic.lua and utils.lua code into your development environment.

#### 2. Add Custom Logic

Generally, there’s no need to modify the core functionality already implemented in basic.lua. Simply add additional functionality based on your specific business requirements.

For example, you can build a [lossless arbitrage Agent](lossless-arbitrage-agent.md) within the FFP protocol.

### Summary

The **Basic Blueprint** is the standard template for building FFP Agents. All FFP Agents are built on this foundation and extended with additional features to meet specific business needs. Through customized Agents, developers can easily integrate into the FusionFi protocol ecosystem and participate in various financial activities.
