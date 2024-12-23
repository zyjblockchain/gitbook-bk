---
description: >-
  It is recommended to understand the following basic concepts before diving
  into the FusionFi protocol.
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Basic Concepts

#### AO <a href="#ao" id="ao"></a>

AO is a world computer built on top of Arweave, a decentralized storage network. Its "hyper parallel" architecture can scale through an unlimited number of parallel processes, supporting scalable and efficient data storage and processing. AO provides FusionFi a flexible computing platform that enables users to perform complex computational tasks in a decentralized environment, underpinned by Arweave's data security and availability guarantees.

For detailed information about the AO network, please visit [here](https://cookbook_ao.arweave.dev/).

#### Process <a href="#process" id="process"></a>

The "process" is a core concept in AO, referring to a program executed in an independent computing environment. Each process interacts with the rest of the AO network according to its programmable logic. Processes are similar to smart contracts on other blockchains, but offer far more powerful and efficient computations. Each process has the complete history of its message logs on Arweave, and maintains an independent holographic state. Each individual AO process can act as a verifiable autonomous agents, capable of performing large-scale computations and executing advanced tasks on-chain without human intervention.

#### Agent <a href="#agent" id="agent"></a>

Agent is a specialized program entity with a clear function and goal. An Agent can perform a single task such as buying or selling tokens, or autonomously operate by executing complex logic, such as multi-step workflows, advanced conditional decisions, and dynamic interactions with other Agents.

#### AgentFi <a href="#agentfi" id="agentfi"></a>

AgentFi is a financial model that allows users to create autonomous agents that carry out financial operations through smart contracts. Through these personalized agents, users can automatically execute asset management, strategy execution, and other operations. Agents are able to achieved this by seamlessly interacting with financial contracts such as tokens and even other agents. Traditional DeFi protocols have issues with centralized management and rigid operations, while AgentFi enhances trading freedom and offers independence from centralized financial services, known as "sovereign finance."

#### Note <a href="#note" id="note"></a>

The Note is the unified data model of the FusionFi protocol, with the flexibility to represent any financial order, from simple [buy orders to lending requests](#user-content-fn-1)[^1]. Notes exists as a non-fungible token in the protocol, and it is the interactions of these tokens that enable all of financial interactions.

#### Settlement <a href="#settlement" id="settlement"></a>

Settlement is the process of validating and finalizing financial transactions within the FusionFi protocol. Settlement works by combining two[^2] or more Notes (e.g. a buy order and a sell order), and securely fulfilling each of their intents. [Settlement in FusionFi is executed both atomically and in parallel, giving the unprecedented combination of security with unlimited scalability](#user-content-fn-3)[^3].

[^1]: better example here?

[^2]: Can a single note be settled?

[^3]: Verify this claim!
