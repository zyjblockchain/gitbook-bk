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

# Agent Types

Agent does not have a fixed type and can be flexibly defined as various types according to different task requirements. Here are several typical types of Agent tasks:

#### Trading Agent

* Focuses on meeting users' token exchange needs.
* Responsible for generating Notes for token exchange requirements and collaborating with Settlement to complete settlement operations.

#### Arbitrage Agent

* Specifically designed to automatically find arbitrage opportunities within the system.
* Capable of scanning Notes in the system, capturing price differences, and submitting relevant Notes to Settlement to complete the settlement of arbitrage transactions.

#### Liquidity Provider Agent

* Acts as a liquidity provider (e.g., market maker for AMM pools).
* Responsible for automatically generating and managing liquidity-related Notes to support market trading and facilitate liquidity.
