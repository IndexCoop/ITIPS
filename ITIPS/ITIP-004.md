# ITIP-4
*Using template v0.1*

## Abstract

The Index community wants to modify the existing Manager Contracts to remove the ability for Index Coop
(designated as the `operator` in the Manager contracts) to unilaterally change the:
+ fee
+ fee split
+ fee recipient amounts

They require that **both** the Index Coop and external methodologists approve any changes related to these 3 parameters.

This issue is being deliberated by the Index Community as [IIP-64][21]

[21]: https://gov.indexcoop.com/t/iip-64-methodologist-smart-contract-permissioning/2200

We want to implement a new permissions model for the manager that lets us configure which kinds of functionality
are strongly or weakly permissioned. We need to be able to restrict fee changes as requested while preserving
the operator's flexibility with regard to updating extensions and modules more generally.

## Background Information

The [original Index Manager Contract][1] defined fee management functions and gated them
with a [mutualUpgrade][2] modifier which required both operator and methodologist consent for any changes.

In early 2021 Index Coop developed a new Manager system which delegates most functionality to extension contracts.
Its architecture helps Index engineers adhere to common "separation of concerns" software design principles
and easily upgrade Index token features.

The [FeeSplitAdapter][6] and [StreamingFeeSplitExtension][3] contract developed for the new system restricts fee changing methods with the `onlyOperator` modifier. In July, the DefiPulse methodologist expressed concern about this change. They are declining to migrate to the new system unless their existing fee prerogatives are preserved.

The new system's `BaseManager` contract has a small set of methods that permit module and
extension additions and removals.

By design, the operator is free to add extensions and modules at will and
+ extensions can call any module
+ extensions can be arbitrarily permissioned
+ the `BaseManager` is blind to the purpose of the calls routed through it

In order to comply with the Index community's requirement that fee arrangements be firewalled from any possibility of unilateral change we will need to:

+ update the existing fee split contracts to restore `mutualUpgrade` permissions settings
+ make changes to the manager contract that allow the methodologist to intervene if the operator adds extensions that don't conform to the necessary fee permission requirements
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
- [ ] Are there any extension contract operations for which the possibility of a methodologist veto poses unacceptable
      risks?

## Feasibility Analysis

**FeeSplitExtension Permissions**

Restoring the `mutualUpgrade` modifier to the relevant fee methods to FeeSplitExtensions is straightforward.

**BaseManager Permissions**

The BaseManager contract will need finer grained permissions for methods that route module calls. A simple way of achieving this is to add a `protectedModules` mapping to the Manager which allows the methodologist to register sensitive
modules and define which extensions may call them. **New modules would be unprotected by default**.

**Fee Security**

At the moment, the `BaseManager` is the receipient of fees prior to distribution and fee accrual
can be initiated by anyone as an atomic step (via the `StreamingFee` module). This arrangement exposes the fees to
extensions that might not conform to Index Community fee requirements. We should update `FeeExtension`
contracts so fees accrue to directly to themselves - by definition they'll be appropriately permissioned.

**Delegation of Authority**

At the core of this upgrade is a dillema about finding the correct balance between protecting methodologist
prerogatives and ensuring that Index Coop's engineering team can add features efficiently.

Unfortunately, there is no programmatic way of guaranteeing *before the fact* that an extension with arbitary permissions doesn't interact with a new module which might affect methodologists' fees.

One solution is to give the methodologist sole authority to manage which modules are protected and which extensions are allowed to interact with them. This would let the methodologist veto any functional changes to the Index they
disagree with by:

+ restricting the permissions of modules it has concerns about
+ declining to grant unwelcome extensions access to protected modules

In the current system the operator is given broad latitude to improve the contracts and asks the methodologist to trust that it will not act in bad faith with respect to fees. We would swap this arrangement for one in which the methodologist has more authority and the operator must trust that the methodologist will strike a viable balance between engineering quality and direct operational control.

## Timeline

- Spec and review:  1 - 2 days
- Implementation: 1 day
- Internal review: 1 day
- Deployment scripts / execution docs: 1 day
- Deploy to testnet / testing: 1 day

## Checkpoint 1

**Reviewer**:

## Proposed Permission and Architecture Changes

**Code References**
+ [mutualUpgrade][8]
+ [onlyOperator][9]
+ [onlyMethodologist][10]

[8]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/b654195c7c0c4c50fb2568399c5d6e0f2659d508/contracts/lib/MutualUpgrade.sol#L25-L72
[9]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/837fa4218aee30711e10fd14feaebbb5337b82bf/contracts/manager/BaseManager.sol#L61-L64
[10]: https://github.com/SetProtocol/index-coop-smart-contracts/blob/837fa4218aee30711e10fd14feaebbb5337b82bf/contracts/manager/BaseManager.sol#L69-L72

### Permissions

The following existing methods would have their modifiers updated

**StreamingFeeSplitExtension**

| Method | Old | New |
| ---- | ---- | ---- |
| *updateStreamingFee* | onlyOperator | mutualUpgrade |
| *updateFeeRecipient* | onlyOperator | mutualUpgrade |
| *updateFeeSplit* | onlyOperator | mutualUpgrade |

**FeeSplitAdapter**

| Method | Old | New |
| ---- | ---- | ---- |
| *updateStreamingFee* | onlyOperator | mutualUpgrade |
| *updateIssueFee* | onlyOperator | mutualUpgrade |
| *updateRedeemFee* | onlyOperator | mutualUpgrade |
| *updateFeeRecipient* | onlyOperator | mutualUpgrade |
| *updateFeeSplit* | onlyOperator | mutualUpgrade |

**BaseManager**

| Method | Old | New |
| ---- | ---- | ---- |
| *setManager* | onlyOperator | mutualUpgrade |


### BaseManager contract changes

The `BaseManager` would be upgraded to include a new permissions data structure

```solidity
// Defines modules which can only be accessed by a specific list of extensions
mapping(address => address[]) public protectedModules;

// **ONLY METHODOLOGIST** Marks a module protected (if not already) and grants an extension
// permission to interact with it
function addExtensionForProtectedModule(address _module, address _extension);

// **ONLY METHODOLOGIST** Removes an extension's ability to interact with a module
function removeExtensionForProtectedModule(address _module, address _extension);

// **ONLY METHODOLOGIST** Remove a module from the protected modules mapping
function removeAllModuleProtections(address module)

// Events
event ProtectedModuleExtensionAdded(address module, address extension)
event ProtectedModuleExtensionRemoved(address module, address extension)
event ModuleProtectionsRemoved(address module)
```

**Existing Methods Changes**

`interactManager` would be modified to check whether its `module` target was
protected and whether the extension contract invoking it was allowed to call.


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
