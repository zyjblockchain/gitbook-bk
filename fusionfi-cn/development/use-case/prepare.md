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

# Prepare

### Install

Integrate aoffp into the project.

```bash
npm install aoffp
```

Install dependencies.

```bash
npm install
# or 
yarn install
```

### Setup config

Create a wallet and save it to the wallets folder:

```bash
mkdir wallets
node ./generate.js
```

Configure WALLET1 and WALLET2 as environment variables in the .env.local file:

```bash
export $(cat .env.local | xargs)
```

### Claim test tokens

The example supports $HELLO and $KITTY as test tokens, which can be obtained using the following commands:

```bash
node ./airdrop.js
```

Check Balances:

```bash
node ./balance.js --address=$WALLET1
```

The output result is similar:

```bash
address: wBn7-31aDtChhLfUk_eXNG9Nbafa_ghT29XRxk7osiM  
$HELLO balance: 100000000000000  
$KITTY balance: 100000000000000
```
