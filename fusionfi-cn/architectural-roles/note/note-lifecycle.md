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

# Note Lifecycle

A FusionFi Note typically goes through the following stages:

#### 1. Create

* A Note is created by the Agent based on their requirements, and configured with all necessary fields (such as assets, amount, yield, collateral, etc.).

#### 2. Hold

* Note is held by the designated holder, entering the stage of awaiting settlement or matching settlement.

#### 3. Submit

* Note is submitted to Settlement, preparing to enter the settlement process.

#### 4. Settle

* Settlement performs settlement operations on the Note, completing asset transfer, profit distribution, etc., and the status is updated to "Settled".

#### 5. Complete

* After settlement is completed, the status of the Note changes to "Completed"; if the transaction demand is canceled or expired, the Note will be destroyed.
