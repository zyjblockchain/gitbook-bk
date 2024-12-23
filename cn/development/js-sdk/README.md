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

# JS-SDK

### Install

```bash
npm install aoffp
# or
yarn add aoffp
```

### Get Settlement Process

```javascript
import { getSettleProcessId } from 'aoffp'
const settleProcessId = getSettleProcessId()
console.log('settleProcessId', settleProcessId)js
```

### StartSettle

Start a settlement process.

```javascript
const toObject = (tags:any[]) => {
  let obj = {} as any
  tags.forEach((item:any) => {
    obj[item.name] = item.value
  })
  return obj
}

const waitSeconds = async (seconds: number): Promise<void> => {
  return await new Promise(resolve => {
    setTimeout(resolve, seconds * 1000)
  })
}

const getProcessResult = async (
  message: string,
  process: string
): Promise<{ message: any; output: any; spawns: any; err: any }> => {
  const { Messages, Error, Output, Spawns } = await ao.result({
    message: message,
    process: process
  })
  return { message: Messages, output: Output, spawns: Spawns, err: Error }
}

export const startSettle = async (signer: any, settleProcessId: string, noteIds: string[], version: string) => {
  const messageId = await ao.message({
    process: settleProcessId, // to settle
    signer,
    tags: [
      { name: 'Action', value: 'StartSettle' },
      { name: 'Version', value: version },
    ],
    data: JSON.stringify(noteIds)
  })
  await waitSeconds(2)
  let result:any
  // eslint-disable-next-line no-constant-condition
  while(true){
    try{
      result = await getProcessResult(messageId, settleProcessId)
      break
    }catch(e){
      console.log(e)
    }
    await waitSeconds(3)
  }
  if (result?.err) {
    throw result?.err
  }
  const data = toObject(result.message[0].Tags)
  if (data.Error) {
    throw data.Error
  }
  return data.SettleID
}

```

### Wallet Settle

The wallet user initiates the settlement operation.

```javascript
export const trade = async (signer: any, settleProcessId: string, tokenIn: string, amountIn: string, settleId: string) => {
  return await ao.message({
    process: tokenIn,
    signer: signer,
    tags: [
      { name: 'Action', value: 'Transfer' },
      { name: 'Recipient', value: settleProcessId }, // to settle
      { name: 'Quantity', value: amountIn },
      { name: 'X-FFP-For', value: 'Settle' },
      { name: 'X-FFP-SettleID', value: settleId },
    ],
  })
}
```
