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

# Basic Agent

The Agent is the core entity that performs operations on FFP. A Basic Agent (such as a personal account or business account) can be created with simple commands, and it supports depositing tokens for subsequent transactions or other operations.

### Create

Execute the following command to create two basic Agents:

```bash
node ./basic/create.js
```

Example output:

```
wBn7-31aDtChhLfUk_eXNG9Nbafa_ghT29XRxk7osiM create agent: <YourAgent1>  
ORHaLUrAiknTAq2Wszoyl6buJrd3MqDKLTF_2CggLtw create agent: <YourAgent2>
```

Configure AGENT1 and AGENT2 in the .env.local file and load:

```bash
export $(cat .env.local | xargs)
```

### Deposit Token into Agent

Use the following command to deposit tokens into the Agent:

```bash
node ./basic/deposit.js --walletN=1 --agentId=$AGENT1
node ./basic/deposit.js --walletN=2 --agentId=$AGENT2
```

Check the Agent’s balance:

```bash
node ./balance.js --address=$AGENT1
node ./balance.js --address=$AGENT2
```
