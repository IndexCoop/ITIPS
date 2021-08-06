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
  + *Answer* The methodologist might abuse this power. They could withold consent for contract
  upgrades to gain leverage in negotiations over unrelated matters. Granting the methodologist more control over the Manager contract complicates the operational environment for engineers designing the system. It introduces unknowns about which extensions might be enabled for a token and when.

- [ ] Are there any extension contract operations for which the possibility of a methodologist veto poses unacceptable risks?
  + *Answer* We likely can't answer this with "no" and have a confidence interval in the high 90s, so the answer is effectively "yes". We should assume that unexpectedly disabling an extension could break assumptions in the system with negative consequences.

## Feasibility Analysis

**FeeSplitExtension Permissions**

Restoring the `mutualUpgrade` modifier to the relevant fee methods to FeeSplitExtensions is straightforward.

**BaseManager Permissions**

The BaseManager contract will need finer grained permissions for methods that route module calls. A simple way of achieving this is to add a `protectedModules` mapping to the Manager which allows the methodologist to register sensitive modules and define which extensions may call them. **New modules would be unprotected by default**.

**Fee Security**

At the moment, the `BaseManager` is the receipient of fees prior to distribution and fee accrual
can be initiated by anyone as an atomic step (via the `StreamingFee` module). This arrangement exposes the fees to extensions that might not conform to Index Community fee requirements. We should update `FeeExtension` contracts so fees accrue to directly to themselves - by definition they'll be appropriately permissioned.

**Delegation of Authority**

At the core of this upgrade is a dillema about finding the correct balance between protecting methodologist prerogatives and ensuring that Index Coop's engineering team can add features both safely and efficiently.

Unfortunately, there is no programmatic way of guaranteeing *before the fact* that an extension with arbitary permissions doesn't interact with a new module which might affect methodologists' fees.

One solution is to give the methodologist authority to manage which modules are protected and which extensions are allowed to interact with them. This would let the methodologist veto any functional changes to the Index they disagree with by:

+ restricting the permissions of modules it has concerns about
+ requiring that extensions be authorized to access protected modules

In the current system the operator is given broad latitude to improve the contracts and asks the methodologist to trust that it will not act in bad faith with respect to fees. We would swap this arrangement for one in which the methodologist has more authority and the operator must trust that the methodologist will strike a viable balance between engineering quality and direct operational control.

The methodologist's authority to restrict extensions must be constrained to a defined period of review. This is a basic critical systems engineering requirement. The operator cannot meet their obligation to build safe smart contracts if they are unable to make assumptions about which modules and extensions are operational.

It's also necessary that the operator be able to add and remove modules at will to address critical code vulnerabilities and bugs.

We propose that when the operator adds modules, the methodologist be able to classify them as
protected within a designated review period. Thereafter they will be required to explicitly enable extensions for the protected module. The methodologist is free to revoke a module's protected status at any time. The table below shows the possible state conditions under this model.

**Protected State Conditions**

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
+ Methodologists can only protect a module within a designated review period which begins when the module is added by the operator.
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
// Defines modules which can only be called by authorized extensions
mapping(address => bool) public protectedModules;

// Defines modules which can only be accessed by a specific list of extensions
mapping(address => address[]) public authorizedExtensions;

// **ONLY METHODOLOGIST, WITHIN MODULE ADDITION REVIEW PERIOD** Marks a module protected.
function protectModule(address _module)

// **ONLY METHODOLOGIST** Revokes a modules protected status and deletes the list of authorized
// extensions associated with it.
function unProtectModule(address _module)

// **ONLY METHODOLOGIST** Grants an extension permission to interact with protected module
function authorizeExtensionForProtectedModule(address _module, address _extension);

// Events
event ProtectedModuleExtensionAdded(address _module, address _extension)
event ProtectedModuleExtensionRemoved(address _module, address _extension)
event ModuleProtected(address _module)
event ModuleUnprotected(address _module)

// Getters
// **ANYONE CAN CALL** Retrieves the list of extensions authorized for `_module`
function getAuthorizedExtensions(address _module)
```

**Existing Methods Changes**

`interactManager` would be modified to check whether its `module` target was protected and whether the extension contract invoking it was allowed to call.


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
