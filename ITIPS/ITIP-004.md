# ITIP-4
*Using template v0.1*

## Abstract

The Index community wants to modify the existing Manager Contracts to remove the ability for Index Coop
(designated as the `operator` in the Manager contracts) to unilaterally change the:
+ fee
+ fee split
+ fee recipient

They require that **both** the Index Coop and external methodologists approve any changes related to these 3 parameters.

This issue is being deliberated by the Index Community as [IIP-64][21]

[21]: https://gov.indexcoop.com/t/iip-64-methodologist-smart-contract-permissioning/2200

We want to implement a new permissions model for the manager that lets us configure which kinds of functionality are strongly or weakly permissioned. We need to be able to restrict fee changes as requested while preserving the operator's flexibility to update extensions and modules more generally.

## Background Information

The [original Index Manager Contract][1] defined fee management functions and gated them with a [mutualUpgrade][2] modifier which required both operator and methodologist consent for any changes.

In early 2021 Index Coop developed a new Manager system which delegates most functionality to extension contracts. Its architecture helps Index engineers adhere to common "separation of concerns" software design principles and easily upgrade Index token features.

The [FeeSplitAdapter][6] and [StreamingFeeSplitExtension][3] contracts developed for the new system restrict fee changing methods with the `onlyOperator` modifier. In July, the DefiPulse methodologist expressed concern about this change. They are declining to migrate to the new system unless their existing fee prerogatives are preserved.

The new system's `BaseManager` contract has a small set of methods that permit module and extension additions and removals.

By design, the operator is free to add extensions and modules at will and
+ extensions can call any module
+ extensions can be arbitrarily permissioned
+ the `BaseManager` is blind to the purpose of the calls routed through it

In order to comply with the Index community's requirement that fee arrangements be firewalled from any possibility of unilateral change we will need to:

+ update the existing fee split contracts to restore `mutualUpgrade` permissions settings
+ make changes to the manager contract that give the methodologist control over how fee specific contracts are upgraded
+ ensure that any fees collected are controlled by correctly permissioned extension contracts

Contracts affected by the required changes include:

+ [BaseManager][4]
+ [FeeSplitAdapter][6] (only used on FLI products atm)
+ [StreamingFeeSplitExtension][7]

[1]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/manager/ICManager.sol
[2]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/b654195c7c0c4c50fb2568399c5d6e0f2659d508/contracts/manager/ICManager.sol#L257-L282
[3]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/b654195c7c0c4c50fb2568399c5d6e0f2659d508/contracts/adapters/StreamingFeeSplitExtension.sol#L97-L133
[4]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/manager/BaseManager.sol
[5]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/lib/BaseAdapter.sol
[6]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/adapters/FeeSplitAdapter.sol
[7]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/master/contracts/adapters/FlexibleLeverageStrategyExtension.sol

## Open Questions
- [ ] What are the drawbacks to delegating more authority to the methodologist?
  + *Answer* The methodologist might abuse this power. They could withold consent for contract
  upgrades to gain leverage in negotiations over unrelated matters. It complicates the operational environment for engineers by introducing unknowns about which extensions might be enabled for a module and when.

- [ ] Are there any extension contract operations for which the possibility of a (retroactive) methodologist veto poses unacceptable risks?
  + *Answer* We can't confidently answer this with "no", so the answer is effectively "yes". We should assume that unexpectedly disabling an extension could break assumptions in the system with negative consequences.

## Feasibility Analysis

### FeeSplitExtension Permissions

Restoring the `mutualUpgrade` modifier to the relevant fee methods to FeeSplitExtensions is straightforward.

### Fee Security

At the moment, the `BaseManager` is the receipient of fees prior to distribution and fee accrual
can be initiated by anyone as an atomic step (via the `StreamingFee` module). This arrangement exposes the fees to extensions that might not conform to Index Community fee requirements. We should update `FeeExtension` contracts so fees accrue to directly to themselves - by definition they'll be appropriately permissioned.

### Module and Extension Updates

At the core of this ITIP is a dillema about finding the right balance between protecting methodologist prerogatives and ensuring that Index Coop's engineering team can operate the
system securely.

A solution has to take the following constraints into consideration:
1. the operator must be able to add and remove modules and extensions at will to mitigate vulnerabilities and fix bugs
2. to design system interactions safely, the operator must have durable guarantees about which modules and extensions are enabled

Below we sketch two strategies to balance these requirements and evaluate their risks for both methodologists and operators.

#### Protected Fee Contracts with Permissioned Manager Initialization

--------

This model for *BaseManager* has the following properties:
+ fee specific module and extension identities are initialized on deployment in the manager contract constructor
+ the manager contract must be enabled by the methodologist before it can begin operation
+ updates to fee modules and extension identities can only be performed by mutual upgrade, except under emergency conditions

Protections are enforced in the *BaseManager* `interactManager` function by requiring that any calls forwarded to the dedicated fee module are made by an authorized fee extension.

This arrangement is the same as the one implemented in the original *ICManager* contract, with small changes to preserve modular design and upgradeability features introduced by the new Manager system.

**Risks**

The operator retains the ability to remove the streaming fee module from the SetToken and change fee arrangements with new extensions and modules.

We can mitigate this risk for the methodologist by making the fee module and extensions impossible to remove unilaterally but this exposes us to significant operational dangers if either contract has bugs.

To address that problem we'd need to define emergency removal methods that let the operator de-activate problematic contracts. To make sure these methods weren't misused as a backdoor to alter fee arrangements we'd likely need to freeze further upgrades to the BaseManager at the same time. For security reasons, the operator would retain the ability to remove modules and extensions while upgrades were frozen.

The freeze could be resolved with a mechanism that lets the methodologist re-enable upgrades while adding and authorizing new fee contracts.

**Fee Contract & BaseManager State Conditions**

| Stage | Fee Contracts | Operator | Methodologist | Outcome |
| ----  | ----- | ---- | ---- | ---- |
| deployment | inactive | deploys, add modules and extensions | n/a | *interactManager* frozen |
| deployed | inactive | n/a | does nothing | *interactManager* frozen |
| deployed | inactive | n/a | authorizes | *interactManager* enabled / fee contracts can be actived by operator |
| active | active | upgrades fee contract(s) | does nothing | no change |
| active | active | upgrades fee contract(s) | authorizes | new fee contracts active |
| active | active | emergency removes fee contract(s) | n/a | fee contracts inactive / upgrades blocked |
| upgrades blocked | inactive | emergency upgrades fee contract(s) | does nothing | fee contracts inactive / upgrades blocked |
| upgrades blocked | inactive | emergency upgrades fee contract(s) | authorizes | new fee contracts active / upgrades re-enabled |


#### Delegation of Authority

-------

Another approach is to give the methodologist broad authority to manage which modules are protected and which extensions are allowed to interact with them. In this model, the methodologist would be able to veto any functional changes to the Index they disagree with by:

+ restricting the permissions of modules it has concerns about
+ requiring that extensions be authorized to access protected modules

To satisfy the engineering requirement that the operator have durable guarantees about which modules and extensions are enabled, the methodologist's ability to restrict extensions would be limited to a defined period of review.

**Risks**

The methodologist may:
+ try to class all new modules as protected making operation of the index difficult to manage.
+ abuse their power to protect modules to gain leverage in negotiations with the community
+ extort concessions from the operator whenever extension upgrades benefit the operator.

For example, suppose the operator shoulders execution costs for some aspect of the system and discovers a way to reduce these by upgrading the manager's module contracts. The methodologist could demand that fees be renegotiated in their favor before enabling the new contracts to capture some of these gains (despite having done nothing to earn them).

These risks could be mitigated by designing a mechanism that lets the community vote to revoke a methodologist's protection of a given module. Ultimately, this option returns the methodologist to square one with respect to whether their fees can be altered without their consent - e.g they can, by popular vote.

**Protected State Conditions**

The table below shows the possible state conditions under this model (without voting).

| Review period | Module State | Operator | Methodologist | Outcome |
| ---- | ----- | ---- | ---- | ---- |
| active | n/a | Adds module | protects module | all future extensions must be authorized |
| active | n/a | Adds module | does nothing | all extensions enabled |
| active | n/a | Adds module and extension | protects module | current extension disabled until authorized, future extensions must be authorized |
| active | protected | n/a | un-protects module | all extensions enabled, methodologist may still protect module while review window is open  |
| elapsed | protected | n/a | un-protects module | all extensions enabled |
| elapsed | protected | Adds extension | approves extension | extension enabled |
| elapsed | protected | Adds extension | does nothing | extension disabled |
| elapsed | unprotected | Adds extension |  n/a | extension enabled |

**Summary**
+ Modules are unprotected by default
+ Methodologists can protect a module within a designated review period which begins when the module is added by the operator.
+ Methodologists must approve extensions for protected modules to enable calls between extension and module.
+ Methodologists cannot remove authorization from an extension once granted

## Timeline

- Spec and review:  1 - 2 days
- Implementation: 1 day
- Internal review: 1 day
- Deployment scripts / execution docs: 1 day
- Deploy to testnet / testing: 1 day

## Checkpoint 1

**Reviewer**:

## Proposed Permission and Architecture Changes


## Requirements

**Reviewer**:

Reviewer: []

## Specification

- Pseudo code

**Reviewer**:

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()
[Link to Deploy outputs PR]()
