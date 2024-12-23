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

### Create Basic Agent

If the Basic Agent Process has not been created, it needs to be created first. The creation operation is as follows:

```javascript
import { createBasicProcess } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const signer = createDataItemSigner(arJWK)
const settleProcessId = getSettleProcessId()
const agent = await createBasicProcess(signer)
const agentProcessId = agent.agentId
console.log('agentProcessId', agentProcessId)
```

Use the following command to create a new agent instance:

```javascript
import { Basic } from 'aoffp'
import { createDataItemSigner } from '@permaweb/aoconnect'

const signer = createDataItemSigner(arJWK)
const agentProcessId = 'your-agent-process-id'
const settleProcessId = getSettleProcessId()
const agent = new Basic(signer, agentProcessId, settleProcessId)
```

### Agent Operations

#### Deposit Funds

If you need to deposit funds into the agent, use the following operation:

```javascript
const depositMessageId = await agent.deposit(tokenId, quantity)
console.log('depositMessageId', depositMessageId)
```

#### Withdraw Funds

If you need to withdraw funds from the agent, use the following operation:

```javascript
const withdrawMessageId = await agent.withdraw(tokenProcessId, quantity)
console.log('withdrawMessageId', withdrawMessageId)
```

### Get All Orders in FFP

```javascript
const allOrders = await agent.getOrders(tokenIn, tokenOut, status, desc, page, pageSize)
console.log('allOrders', allOrders)
```

### Take Order

```javascript
const takeOrderMessageId = await agent.takeOrder([noteId1, noteId2, ...])
console.log('takeOrderMessageId', takeOrderMessageId)
// result
const takeOrderResult = await getProcessResult(takeOrderMessageId, agent.agentId)
console.log('takeOrderResult', JSON.stringify(takeOrderResult, null, 2))
```
