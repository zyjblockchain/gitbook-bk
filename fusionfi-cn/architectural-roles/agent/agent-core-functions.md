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

# Agent Core Functions

#### Create Note

* The Agent is the role responsible for creating FusionFi Notes based on user requirements.
* Agents can define the specific functions and parameters of Notes according to different business scenarios (such as arbitrage, market making, token exchange, etc.).

#### Receive Note

* Agents can receive Notes created and sent by other Agents. From the Agent's perspective, these Notes represent the trading requirements of the other party, and the Agent may choose to include this Note in Settlement.

#### Submit Note

* Agents can submit Notes they have created or received to Settlement.
* Before submitting, the Agent should ensure that their Notes comply with the settlement rules and requirements.

#### Monitoring and Execution

* Agents can also monitor Notes in the system in real-time, actively seeking trading opportunities. They can achieve automated execution of simple or complex trading logic (e.g. arbitrage), based on their programmed strategies.
