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

# Note Schema

Each Note can be regarded as a token that possesses the following core characteristics:

* Uniqueness: Each note is identified by a unique NoteID, making it independent and irreplaceable.
* Non-fungibility: Each note has independent metadata and behavioral logic.
* Single Holder: Each note has at most one current Holder, which can be a User, Agent, etc. If no Holder is specified, it is considered a "Public Note" and can be settled by anyone.

FusionFi protocol defines the following schema for Notes:

<table data-full-width="false"><thead><tr><th width="226">Field</th><th>Description</th></tr></thead><tbody><tr><td>NoteID</td><td>The unique identifier of the Note, usually a system-generated hash value.</td></tr><tr><td>Issuer</td><td>The ID of the Agent that created the Note.</td></tr><tr><td>Holder</td><td>The current holder of the Note. If not specified, denotes a public note and can be settled by anyone.</td></tr><tr><td>Amount</td><td>The the quantity of the Note's indicated asset, <a data-footnote-ref href="#user-content-fn-1">usually</a> a BigInt.</td></tr><tr><td>AssetID</td><td>The identifier for the type of asset represented by the Note.</td></tr><tr><td>AssetYield</td><td>The annualized yield of the Note's underlying asset, indicating the asset's appreciation rate, commonly used for lending or or income-generating Notes.</td></tr><tr><td>HolderAmount</td><td>The number of underlying assets held by the holder (counterparty assets), describing the asset demand or liability on the other side of the note.</td></tr><tr><td>HolderAssetID</td><td>The identifier of the asset held by the holder, indicating the type of asset held by the counterparty.</td></tr><tr><td>HolderAssetYield</td><td>The annualized return rate of counterparty assets, indicating the appreciation of counterparty assets, is usually related to the counterparty's profit conditions.</td></tr><tr><td>IssueDate</td><td>Note creation time, marking the starting point of the note's lifecycle.</td></tr><tr><td>DueDate</td><td>The expiration time of the Note, indicating the validity period of the note, and it may not be settled after this time.</td></tr><tr><td>SettleDate</td><td>Note’s settlement time, indicating the time point when the note is fully settled.</td></tr><tr><td>CollateralAmount</td><td>The amount of collateral assets in the Note, indicating the amount of collateral assets required to ensure transaction security.</td></tr><tr><td>CollateralAssetID</td><td>Identifier of the collateral asset, indicating the specific asset type of the collateral in the note.</td></tr><tr><td>CollateralRate</td><td>Collateral ratio, indicating the proportion of collateral asset value to the total value of notes, used to measure transaction security (such as over-collateralization ratio).</td></tr><tr><td>HolderCollateralAmount</td><td>The amount of collateral assets provided by the counterparty, describing the margin amount provided by the other side of the transaction.</td></tr><tr><td>HolderCollateralAssetID</td><td>The identifier of the counterparty's collateral asset, indicating the specific type of collateral of the counterparty.</td></tr><tr><td>HolderCollateralRate</td><td>Counterparty's collateral ratio, used to measure the counterparty's risk management and security in the transaction.</td></tr></tbody></table>

[^1]: Usually? When is it not?
