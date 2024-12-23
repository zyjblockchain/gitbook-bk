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

# OrderBook Blueprint

### Install

```javascript
npm install aoffp
# or
yarn add aoffp
```

### Get FFP Settlement Process ID

```javascript
import { getSettleProcessId } from 'aoffp'

const settleProcessId = getSettleProcessId()
console.log('settleProcessId', settleProcessId)
```

### Create OrderBook Agent

If the OrderBook Agent Process has not been created, it needs to be created first. The creation operation is as follows:

```javascript
import { createOrderbookProcess } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const address = 'your-arweave-address'
const signer = createDataItemSigner(arJWK)
const settleProcessId = getSettleProcessId()
const orderbookAgent = await createOrderbookProcess(signer)
const orderbookProcessId = orderbookAgent.agentId
console.log('orderbookProcessId', orderbookProcessId)
```

To create a new agent instance, use the following command:

```javascript
import { Orderbook } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const signer = createDataItemSigner(arJWK)
const orderbookProcessId = 'your-orderbook-process-id'
const settleProcessId = getSettleProcessId()
const orderbook = new Orderbook(signer, orderbookProcessId, settleProcessId)
```

### Agent Operations

#### Deposit Funds

To deposit funds into the agent, use the following operation:

```javascript
const depositMessageId = await orderbook.deposit(tokenProcessId, quantity)
console.log('depositMessageId', depositMessageId)
```

#### Withdraw Funds

To withdraw funds from the agent, use the following operation:

```javascript
const withdrawMessageId = await orderbook.withdraw(tokenProcessId, quantity)
console.log('withdrawMessageId', withdrawMessageId)
```

#### Get All Orders in FFP

```javascript
const allOrders = await orderbook.getOrders(tokenIn, tokenOut, status, desc, page, pageSize)
console.log('allOrders', allOrders)
```

#### Place an Order

To place a limit order, use the following operation:

```javascript
const order = await orderbook.makeOrder(tokenInProcessId, tokenOutProcessId, quantityIn, quantityOut)
console.log('order', order)
```

#### Cancel an Order

To cancel an existing order, use the following operation:

```javascript
const cancelOrderMessageId = await orderbook.cancelOrder(noteId)
console.log('cancelOrderMessageId', cancelOrderMessageId)

// result
const cancelOrderResult = await getProcessResult(cancelOrderMessageId, orderbookProcessId)
console.log('cancelOrderResult', JSON.stringify(cancelOrderResult, null, 2))
```

#### Take an Order

To take an order (eat the order), use the following operation:

```javascript
const takeOrderMessageId = await orderbook.takeOrder([noteId1, noteId2, ...])
console.log('takeOrderMessageId', takeOrderMessageId)

// result
const takeOrderResult = await getProcessResult(takeOrderMessageId, orderbookProcessId)
console.log('takeOrderResult', JSON.stringify(takeOrderResult, null, 2))
```
