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

At the core of this ITIP is a dillema about finding the right balance between protecting methodologist prerogatives and ensuring that Index Coop's engineering team can operate the system securely.

A solution has to take the following constraints into consideration:
1. the operator must be able to add and remove modules and extensions at will to mitigate vulnerabilities and fix bugs
2. to design system interactions safely, the operator must have durable guarantees about which modules and extensions are enabled

Below we sketch a strategy to balance these requirements and evaluate their risks for both methodologists and operators.

#### Protected Modules with Permissioned Manager Initialization

--------

The permissions model proposed for *BaseManager* has the following properties:
+ supports a registry of protected modules and defines which extensions may call them
+ protected modules are initialized on deployment in the manager contract constructor
+ the manager contract must be enabled by the methodologist before it can begin operation
+ updates to protected modules can only be performed by mutual upgrade, except under emergency conditions
+ extensions authorized for protected modules cannot be unilateraly removed, except under emergency conditions

Protections are enforced in the *BaseManager* `interactManager` function by requiring that any calls forwarded to a protected module are made by an authorized extension.

This arrangement has the same permissions as the one implemented in the original *ICManager* contract, with small changes to preserve modular design and upgradeability features introduced by the new Manager system.

**Risks**

The methodologist needs guarantees that protected modules cannot be unilaterally swapped out. This requirement has to be implemented in a way that lets the operator intervene immediately if vulnerabilities or bugs are discovered in any contracts.

To accomplish this we propose adding emergency removal methods that let the operator de-activate protected modules. These methods would freeze any further upgrades to the BaseManager, ensuring that they can't be misused as a backdoor to alter fee arrangements. (For security reasons, the operator would retain the ability to remove other modules and extensions while upgrades are frozen.)

To resolve the freeze, the methodologist would need to approve an emergency replacement of the protected modules at issue.

In sum, the operator's power to execute an emergency removal is counterbalanced by the methodologists power to prevent normal upgrades until they authorize a new protected module arrangement.

**State Transition Table: Overview**

| Stage | Protected Module | Operator | Methodologist | Outcome |
| ----  | ----- | ---- | ---- | ---- |
| deployment | inactive | deploys, registers permissioned modules and their authorized extensions | n/a | *interactManager* frozen, regular modules and extensions can be added |
| deployed | inactive | n/a | does nothing | *interactManager* frozen |
| deployed | inactive | n/a | authorizes contract initialization | *interactManager* enabled / all modules can be activated by operator |
| active | active | replaces protected module (*mutual upgrade*)| does nothing | no change |
| active | active | replaces protected module (*mutual upgrade*)| confirms | new module active |
| active | active | adds authorized extension to protected module (*mutual upgrade*) | does nothing | no change |
| active | active | adds authorized extension to protected module (*mutual upgrade*) | confirms | extension activated for protected module |
| active | active | removes extension for protected module (*mutual upgrade*) | does nothing | no change |
| active | active | removes extension for protected module (*mutual upgrade*) | confirms | extension de-activated for protected module |
| active | active | calls regular `removeExtension` on extension authorized for protected module | n/a | fails |
| active | active | calls regular `removeModule` on protected module | n/a | fails |
| active | active | emergency removes protected module | n/a | protected module de-activated / all upgrades blocked |
| emergency | inactive | calls `removeExtension` on regular extension | n/a | regular extension removed |
| emergency | inactive | calls `removeModule` on regular module | n/a | regular module removed |
| emergency | inactive | adds regular module | n/a | fails |
| emergency | inactive | adds regular extension | n/a | fails |
| emergency | inactive | emergency upgrades protected module | does nothing | new protected modules inactive / upgrades blocked |
| emergency | inactive | emergency upgrades protected module | confirms | new protected module active / upgrades re-enabled |


## Timeline

- Spec and review:  1 - 2 days
- Implementation: 1 day
- Internal review: 1 day
- Deployment scripts / execution docs: 1 day
- Deploy to testnet / testing: 1 day

## Checkpoint 1

**Reviewer**:

## Proposed Permission and Architecture Changes

### BaseManager Interface Changes

#### Struct
> ProtectedModule

| Type  | Name  | Description   |
|------ |------ |-------------  |
| boolean | isProtected | true if module can only be called by authorized extensions |
| address[] | authorizedExtensions | array of extensions allowed to call this module |

#### Modifiers

> isInitialized:

```solidity
/**
  Modifier requiring that methodologist has authorized the BaseManager's initial protected modules
  configuration. These are set in the constructor on deployment
 */
modifier isInitialized()
```

> upgradesPermitted

```solidity
/**
  Modifier requiring that contract is not in an "emergency state" following a unilateral operator
  removal of a protected module. In such conditions, the operator cannot add modules or extensions
  without the consent of the methodologist
 */
modifier upgradesPermitted()
```

#### Public Variables

| Type  | Name  | Description   |
|------ |------ |-------------  |
|mapping(address => ProtectedModule) |protectedModules| Defines protected modules. These cannot be called, or removed except by mutual upgrade. Extensions associated with a protected module cannot be unilaterally removed. |
|address[]| protectedModulesList| A list for iterating over the set of protected modules. This is used to verify that an extension removal by the operator does not require methodologist consent |
|boolean | initialized|Flag set by methodologist that must be true for BaseManager to begin routing calls via `interactManager`|
|boolean | upgradesPaused | Flag set to true when the operator "emergency removes" a protected module. |

#### New Methods

> authorizeInitialization:

```solidity
/*
  Called by the methodologist to enable contract. All `interactManager` calls revert until this
  is invoked. Lets methodologist review and authorize initial protected module settings.
 */
function authorizeInitialization()
  external
  onlyMethodologist
```

> replaceProtectedModule
- _oldModule: address of module to replace
- _newModule: address of module to add in place of

```solidity
/**
 **MUTUAL UPGRADE**: Marks a currently protected module as unprotected and deletes its authorized
 extensions array. Marks a new module as protected and initializes it with an empty
 `authorizedExtensions` array. Removes `_oldModule` from the `protectedModulesList`
 */
function replaceProtectedModule(address _oldModule, address _newModule)
  external
  mutualUpgrade(operator, methodologist);
```

> addAuthorizedExtension
- _protectedModule: protected module to add extension for
- _extension: extension which is allowed access to module

```solidity
/**
  **MUTUAL UPGRADE**: Authorizes an extension for a protected module
 */
function addAuthorizedExtension(address _protectedModule, address _extension)
  external
  mutualUpgrade(operator, methodologist);
```

> removeAuthorizedExtension
- _protectedModule: protected module to remove extension for
- _extension: extension whose access to module is revoked

```solidity
/**
  **MUTUAL UPGRADE**: De-authorizes an extension for a protected module
 */
function removeAuthorizedExtension(address _protectedModule, address _extension)
  external
  mutualUpgrade(operator, methodologist);
```

> emergencyRemoveProtectedModules
- address _protectedModule: Module to remove protections for

```solidity
/**
  **ONLY OPERATOR**:
  + Marks a currently protected module as unprotected and deletes its authorized extensions array.

  + Removes `protectedModule` from the `protectedModulesList`.

  + Removes module from the SetToken.

  + Sets the `upgradesPaused`flag to true, prohibiting any further operator-only module or extension
    additions until `emergencyReplaceProtectedModule` is successfully executed.
 */
function emergencyRemoveProtectedModule(address _protectedModule)
  external
  onlyOperator
```

> emergencyReplaceProtectedModule
- _protectedModule: Module to protect
- _extensions: List of extensions authorized to call the protected module

```solidity
/**
  **MUTUAL UPGRADE**: Adds a module (with extensions defined) to replace a protected module
  unilaterally removed by the operator during an emergency. This method unsets `upgradesPaused` and
  restores the operators upgradeability privileges
 */
function emergencyReplaceProtectedModule(address _protectedModule, address[] _extensions)
  external
  mutualUpgrade(operator, methodologist)
```

#### Updated Functions

> constructor
- setTokenAddress,
- operator,
- methodologist,
- [ moduleAddress, ... ]
- [ [extensionAddressForModule, ....], ... ]

```solidity
/*
  Protected modules and their extensions are initialized in the constructor
 */
constructor(
  ISetToken _setToken,
  address _operator,
  address _methodologist
  address[] protectedModules,      // List of modules to initialize as protected
  address[][] authorizedExtensions // Authorized extensions for each protected module
)
```

> setManager
- manager

Change permissions from `onlyOperator` to `mutualUpgrade`, ensuring that methodologist approves
any new manager.

```solidity
/**
  **MUTUAL UPGRADE** Update the SetToken manager address
 */
function setManager(address _newManager)
  external
  mutualUpgrade(operator, methodologist)
```

> interactManager
- module
- data

Add `isInitialized` modifier, preventing calls until methodologist approves initial BaseManager protected
module settings

```solidity
/**
  **ONLY EXTENSION / ONLY WHEN INITIALIZED**: Routes calls from extensions to modules after verifying that
  a module can be called by the extension.
 */
function interactManager(address _module, bytes calldata _data)
  external
  isInitialized
  onlyAdapter
```

### FeeSplitAdapter Interface Changes

All fee changing methods are changed from `onlyOperator` to `mutualUpgrade`

| Method | Old | New |
| ---- | ---- | ---- |
| *updateStreamingFee* | onlyOperator | mutualUpgrade |
| *updateIssueFee* | onlyOperator | mutualUpgrade |
| *updateRedeemFee* | onlyOperator | mutualUpgrade |
| *updateFeeRecipient* | onlyOperator | mutualUpgrade |
| *updateFeeSplit* | onlyOperator | mutualUpgrade |

### FeeSplitAdapter Logic Changes

> updateStreamingFee

Ensure that accrueFeesAndDistribute is called during this operation so that the manager contract
isn't accidentlaly left with a fee balance that could be taken by an unauthorized extension

### StreamingFeeSplitExtension Interface Changes

All fee changing methods are changed from `onlyOperator` to `mutualUpgrade`

| Method | Old | New |
| ---- | ---- | ---- |
| *updateStreamingFee* | onlyOperator | mutualUpgrade |
| *updateFeeRecipient* | onlyOperator | mutualUpgrade |
| *updateFeeSplit* | onlyOperator | mutualUpgrade |

### StreamingFeeSplitExtension Logic Changes

> updateStreamingFee

Ensure that accrueFeesAndDistribute is called during this operation so that the manager contract
isn't accidentlaly left with a fee balance that could be taken by an unauthorized extension


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
