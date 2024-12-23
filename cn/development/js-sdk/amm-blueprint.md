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

### Install

```bash
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

### Create AMM Agent

If the AMM Agent Process has not been created, it needs to be created first. The creation operation is as follows:

```javascript
import { createAmmProcess } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const signer = createDataItemSigner(arJWK)
const ammAgent = await createAmmProcess(signer)
const const ammProcessId = ammAgent.agentId
console.log('ammProcessId', ammProcessId)
```

To create a new AMM Agent Process instance, use the following command:

```javascript
import { Amm } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const signer = createDataItemSigner(arJWK)
const orderbookProcessId = 'your-amm-process-id'
const amm = new Amm(signer, ammProcess)
```

### Agent Operations

#### Deposit Funds

To deposit funds into the agent, use the following operation:

```javascript
const depositMessageId = await amm.deposit(tokenProcessId, quantity)
console.log('depositMessageId', depositMessageId)
```

#### Withdraw Funds

To withdraw funds from the agent, use the following operation:

```javascript
const withdrawMessageId = await amm.withdraw(tokenProcessId, quantity)
console.log('withdrawMessageId', withdrawMessageId)
```

#### Get All Orders in AMM

To retrieve all orders in the AMM, use the following command:

```javascript
const order = await ammRequest(signer, ammAgentId, tokenInProcess, tokenInAmount, tokenOut: string, amountOut?: string)

await new Promise((resolve) => {
  setTimeout(resolve, 5000)
})
console.log('order', JSON.stringify(order, null, 2))

/*
{
  ID: 2567,
  AssetID: 'AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM',
  MakeTx: 'wFVSJurGy3nObNeiFTHKciJruKGWMPEoW5q34OFzx-M',
  ExpireDate: 1733224050340,
  HolderAssetID: 'J0B80MpR_koLQpdqOKA5VcaPayqQPSR5ERzdtBVnkP4',
  NoteID: 'R05uXunK2XUMVIUz_DsBghbEqm7FOhqxiXFFmuwKJGg',
  IssueDate: 1733223960340,
  HolderAmount: '2',
  Amount: '2',
  Status: 'Open',
  Price: 1,
  Issuer: 'iksN-6mCCaG1nrAert49rfiA0ZBgtPKVE7YLTfhTzvE'
}
*/
```
