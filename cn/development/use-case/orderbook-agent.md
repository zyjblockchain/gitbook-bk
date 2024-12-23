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

# OrderBook Agent

OrderBook Agent runs user-set buy and sell orders and supports order matching. This use case includes the following operations:

1. Create an OrderBook Agent.
2. Deposit tokens into the Agent, and the user creates a limit order.
3. Place a limit order and obtain the NoteID of the order.
4. Use the Agent from the Basic use case to take the order and complete the order matching and transaction.

### Create

Execute the following command to create two OrderBook Agents:

```bash
node ./orderbook/create.js
```

Example output:

```bash
wBn7-31aDtChhLfUk_eXNG9Nbafa_ghT29XRxk7osiM create orderbook agent: <YourOrderBookAgent1>  
ORHaLUrAiknTAq2Wszoyl6buJrd3MqDKLTF_2CggLtw create orderbook agent: <YourOrderBookAgent2>
```

Configure ORDERBOOKAGENT1 and ORDERBOOKAGENT2 in the .env.local file and load:

```bash
export $(cat .env.local | xargs)
```

### Deposit Token into Agent

```bash
node ./orderbook/deposit.js --walletN=2 --agentId=$ORDERBOOKAGENT2
```

Check balance:

```bash
node ./balance.js --address=$ORDERBOOKAGENT2
```

### Create a limit order

Use the following command to create a limit order:

```bash
node ./orderbook/make.js --walletN=2 --agentId=$ORDERBOOKAGENT2
```

Example output:

```bash
openOrders {  
  MVBggDjYkl3UxoHRZ2rO6ZLDcN4ax4Af6rehyYJ3CH0: {  
    ID: 2600,  
    AssetID: 'AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM',  
    HolderAssetID: '0fLIp-xxRnQ8Nk-ruq8SBY8icaIvZMujnqCGU79fnM0',  
    NoteID: 'MVBggDjYkl3UxoHRZ2rO6ZLDcN4ax4Af6rehyYJ3CH0',  
    IssueDate: 1733297306962,  
    HolderAmount: '3',  
    Amount: '1',  
    Status: 'Open',  
    Price: 3,  
    Issuer: 'RPFQd69SX2tbrtNBfVxzVSt9zQYk07WrHOCDhxDHu0o'  
  }  
}
```

Query the order:

```bash
node ./orderbook/query.js --agentId=$ORDERBOOKAGENT2
```

Set the NoteID as an environment variable:

```bash
export NOTEID=<NoteID>
```

### &#x20;Take order

Use Basic Agent1 to take the order:

```bash
node ./basic/take.js --walletN=1 --agentId=$AGENT1 --noteId=$NOTEID
```

After the transaction is completed, check the balance of both Agents:

```bash
node ./balance.js --address=$AGENT1
node ./balance.js --address=$ORDERBOOKAGENT2
```
