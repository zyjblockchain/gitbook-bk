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

# AMM Agent

AMM Agent use case demonstrates how to manage liquidity pools and automated trading:

1. Create AMM Agent.
2. Deposit tokens and add liquidity.
3. Place AMM orders to enable automated trading.
4. Use the Agent from the Basic use case to take orders and complete the transaction.

### Create

Execute the following command to create AMM Agents:

```bash
node ./amm/create.js
```

Example output:

```bash
wBn7-31aDtChhLfUk_eXNG9Nbafa_ghT29XRxk7osiM create amm agent: <YourAMMAgent1>
ORHaLUrAiknTAq2Wszoyl6buJrd3MqDKLTF_2CggLtw create amm agent: <YourAMMAgent2>
```

Configure AMMAGENT1 and AMMAGENT2 in the .env.local file and load:

```bash
export $(cat .env.local | xargs)
```

### Deposit Token into Agent

```bash
node ./amm/deposit.js --walletN=2 --agentId=$AMMAGENT2
```

Balance check:

```bash
node ./balance.js --address=$AMMAGENT2
```

### Add Liquidity to the Pool

Deposit tokens into the AMM liquidity pool:

```bash
node ./amm/addPool.js --walletN=2 --agentId=$AMMAGENT2
```

Query liquidity pool information:

```bash
node ./amm/query.js --walletN=2 --agentId=$AMMAGENT2
```

Example output:

```bash
pools {
  "0fLIp-xxRnQ8Nk-ruq8SBY8icaIvZMujnqCGU79fnM0:AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM": {
    "py": "50",
    "algo": "UniswapV2",
    "fee": 30,
    "px": "50",
    "y": "AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM",
    "balances": {
      "0fLIp-xxRnQ8Nk-ruq8SBY8icaIvZMujnqCGU79fnM0": "50",
      "AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM": "50"
    },
    "x": "0fLIp-xxRnQ8Nk-ruq8SBY8icaIvZMujnqCGU79fnM0"
  }
}
```

### Create Order

Create an order from the AMM pool:

```bash
node ./amm/request.js --walletN=2 --agentId=$AMMAGENT2
```

Example output:

```bash
order {
  "ID": 2601,
  "AssetID": "0fLIp-xxRnQ8Nk-ruq8SBY8icaIvZMujnqCGU79fnM0",
  "MakeTx": "zkmqIcN2KNOS38zgAq5AxTfnMeod7OaZUcy_SMOGJqE",
  "ExpireDate": 1733297855923,
  "HolderAssetID": "AttsQGi4xgSOTeHM6CNgEVxlrdZi4Y86LQCF__p4HUM",
  "NoteID": "14ec8heZ5m3XR6j9ZBOIUT-3lhq6qgloaz9ObIG-_PI",
  "IssueDate": 1733297765923,
  "HolderAmount": "5",
  "Amount": "4",
  "Status": "Open",
  "Price": 1.25,
  "Issuer": "xhgS6MeQ4qhYqP21ptsCSHY3m9faNsPs0ewRKB9jvwo"
}
```

Set NoteID as an Environment Variable:

```bash
export NOTEID=<NoteID>
```

### Take Order

Use Basic Agent1 to take the order:

```bash
node ./basic/take.js --walletN=1 --agentId=$AGENT1 --noteId=$NOTEID
```

After the transaction is completed, query the balance of both agents:

```bash
node ./balance.js --address=$AGENT1
node ./amm/query.js --walletN=2 --agentId=$AMMAGENT2
```

### Arbitrage Operation

Combine OrderBook and AMM for a simple arbitrage strategy:

1. Create a low-price buy order or high-price sell order from the OrderBook.
2. Create a corresponding order on the AMM to capitalize on the price difference for arbitrage.
3. Use a script to complete the arbitrage process in one click.

Perform arbitrage using OrderBook and AMM:

```bash
node ./arbitrage.js --agentId=$AGENT1 --orderbookAgentId=$ORDERBOOKAGENT2 --ammAgentId=$AMMAGENT2
```
