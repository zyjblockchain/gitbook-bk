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

# Agent Application Scenarios

Agents are completely programmable and so have unlimited possibilities for application scenarios. Here are several typical application scenarios:

#### Token Exchange

* When users need to exchange tokens (e.g., exchange 5 Token A for 10 Token B), they can achieve this through an Agent.
* The Agent monitors Notes in the system and finds a Note that matches the requirements, e.g. where the settlement condition is exchanging 10 Token B for 5 Token A.
* The user obtains the Note through the Agent and generates a new Note that meets their own needs, then submits both Notes to Settlement for settlement.

#### Arbitrage

* Users can create an Agent specifically for arbitrage. This Agent would autonomously scan for Notes in the system, detect price differences, and Settle any compatible Notes based on the opportunity to gain profits.

#### Provide Liquidity

* Agents can act as liquidity providers (e.g., market makers in AMM pools) and automatically generate Notes based on market demand. The Agents can then provide these Notes to counterparties for matching trades, thereby facilitating liquidity within the system.
