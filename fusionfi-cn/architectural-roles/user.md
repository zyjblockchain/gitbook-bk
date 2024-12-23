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

# User

The User can be an individual user, an organization, or even a smart contract, with a specific role depending on the needs of the business scenario. The user interacts with the FusionFi protocol by generating Agents, which in turn drive the behavior of the entire system.

The following is a detailed introduction to the User component:

#### Generate Agent

* Users create agents to participate in specific operations within the system. Agents act as users' representatives, responsible for executing a series of complex tasks such as creating [Notes](note/), receiving Notes, and submitting Notes to [settlement](settlement/).
* Each user can create multiple Agents and configure their respective responsibilities according to different needs, such as arbitrage, market making, or simple token exchange operations.

#### Authorization and Management

* Users need to authorize the Agents they create to ensure the Agents can perform actions on their behalf.
* Users can adjust or revoke the Agent's permissions at any time, achieving full control over the Agent's behavior to ensure security and flexibility.
