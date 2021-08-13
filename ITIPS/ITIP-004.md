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

At the moment, the `BaseManager` is the receipient of fees prior to distribution and fee accrual occurs in the StreamingFeeModule without immediate distribution when the streaming fee is updates. This arrangement exposes the fees to extensions that might not conform to Index Community fee requirements. We should update `FeeExtension` contracts so fees are always distributed when accrued.

### Module and Extension Updates

At the core of this ITIP is a dillema about finding the right balance between protecting methodologist prerogatives and ensuring that Index Coop's engineering team can operate the system securely.

A solution has to take the following operator requirements into consideration:
1. the operator must be able to add and remove modules and extensions at will to mitigate vulnerabilities and fix bugs
2. to design system interactions safely, the operator must have durable guarantees about which modules and extensions are enabled
3. the modularity and upgradeability properties of the new Manager system are preserved

Below we sketch a strategy to balance these requirements and evaluate their risks for both methodologists and operators.

#### Protected Modules with Permissioned Manager Initialization

--------

The permissions model proposed for *BaseManager* has the following properties:
+ supports a registry of protected modules and defines which extensions may call them
+ protected modules are initialized on deployment in the manager contract constructor
+ the methodologist must enable the manager contract before it can begin operation
+ updates to protected module arrangements can only be performed by mutual upgrade, except under emergency conditions
+ protected modules cannot be unilaterally removed, except under emergency circumstances
+ extensions authorized for protected modules cannot be unilaterally removed, except under emergency conditions
+ the operator can protect new modules at their discretion, on behalf of the methodologist
+ the methodologist can un-protect modules at their discretion, on behalf of the operator

Protections are enforced in the *BaseManager* `interactManager` function by requiring that any calls forwarded to a protected module are made by an authorized extension.

The operator may develop new modules that have fee splitting features that a methodologist might want protections for. The operator is free to extend protected module status to any new module.

This arrangement has the same permissions as the one implemented in the original *ICManager* contract, with changes to preserve modular design and upgradeability features introduced by the new Manager system.

**Risks**

The methodologist needs guarantees that protected modules cannot be unilaterally swapped out. This requirement has to be implemented in a way that lets the operator intervene immediately if vulnerabilities or bugs are discovered in any contracts.

To accomplish this we propose adding emergency removal methods that let the operator de-activate protected modules. These methods would also freeze any further upgrades to the BaseManager, ensuring that they can't be misused as a backdoor to alter fee arrangements. (For security reasons, the operator would retain the ability to remove other modules and extensions while upgrades are frozen.)

To resolve the freeze, the methodologist would need to approve an emergency replacement of the protected modules at issue by mutual upgrade or unilaterally resolve the emergency without replacing anything.

In sum, the operator's power to execute an emergency removal is counterbalanced by the methodologist's power to prevent normal upgrades until they authorize a new protected module arrangement.

**State Transition Tables**

> **Deployment and initialization**

`interactManager` is locked until the methodologist authorizes initialization. Anything
else can be called normally.

| initialized | Action | Target | Caller | Success |
| ----        | -----  | ----   | ----   | ----    |
| false | constructor(...empty) | -| deployer | :white_check_mark: |
| false | constructor(...protections) | -| deployer | :white_check_mark: |
| false | interactManager | regular module | operator/extension | :x: |
| false | interactManager | protected module | operator/extension | :x: |
| false | authorizeInitialization | -| operator | :x: |
| false | authorizeInitialization | -| methodologist | :white_check_mark: |
| true  | interactManager | regular module | operator/extension | :white_check_mark: |
| true  | interactManager | protected module | operator/extension | if authorized |

-----

> **Standard module and extension protections**

|  Action | Target | Caller | Success |
| ----    | -----  | ----   | ----    |
| interactManager | regular module | operator/extension | :white_check_mark: |
| interactManager | protected module | operator/extension | if authorized |
| removeModule | regular module  | operator | :white_check_mark: |
| removeModule | protected module  | operator | :x: |
| removeExtension | regular extension  | operator | :white_check_mark: |
| removeExtension | extension when authorized for any module  | operator | :x: |

-----

> **Protected module status state transitions**

+ "regular" means a module has been added to the manager but isn't protected
+ "not added" means the module has not been added to the manager yet

|  Action | Caller | Initial State | Final State | Success |
| ----    | -----  | ----          | ----        | ----    |
| constructor | operator | -| added and protected | :white_check_mark: |
| protectModule | operator | regular | protected | :white_check_mark: |
| protectModule | operator | not added | not added | :x:|
| protectModule | operator | protected | protected | :x:|
| unProtectModule | methodologist | protected | regular | :white_check_mark: |
| unProtectModule | methodologist | not added | not added | :x:|
| unProtectModule | methodologist | regular | regular | :x:|
| emergencyRemoveProtected | operator | protected | not added | :white_check_mark: |
| emergencyRemoveProtected | operator | not added | not added | :x:|
| emergencyRemoveProtected | operator | regular | regular | :x:|

**...when replacing**:

Replacement methods call SetToken's `addModule` and/or `removeModule`. Modules to replace with should
*not* be added to the manager before calling these methods.

|  Action | Caller | Current Init | Current Final | Replacement Init | Replacement Final | Success |
| ----    | -----  | ----         | ----          | ----             | ----              | ----    |
| replaceProtected | mutual | protected | not-added | not added | protected | :white_check_mark: |
| replaceProtected | mutual | protected | protected | regular | regular | :x: |
| replaceProtected | mutual | regular | regular | -| -| :x:|
| replaceProtected | mutual | not added | not added | -| -| :x:|
| emergencyReplaceProtected | mutual | -| -| not added | protected | :white_check_mark: |
| emergencyReplaceProtected | mutual | -| -| protected | protected | :x:|
| emergencyReplaceProtected | mutual | -| -| regular | regular | :x:|

-----

> **Authorized extension status state transitions**

+ "regular" means an extension has been added to the manager but isn't authorized for any module
+ "not added" means the extension has not been added to the manager yet
+ authorization is granted for a specific module. Extensions can be authorized for more than one.

|  Action | Caller | Module State | Initial Ext. State | Final Ext. State | Success |
| ----    | -----  | ----         | ----               | ----             | ----    |
| authorizeExtension | mutual | protected |regular | authorized | :white_check_mark: |
| authorizeExtension | mutual | protected |authorized | authorized | :x:|
| authorizeExtension | mutual | protected |not added | not added | :x:|
| authorizeExtension | mutual | regular   | regular | regular | :x: |
| revokeExtensionAuth | mutual | protected |authorized | regular | :white_check_mark: |
| revokeExtensionAuth | mutual | protected | regular | regular | :x:|
| revokeExtensionAuth | mutual | protected | not added | not added | :x:|
| revokeExtensionAuth | mutual | regular | regular | regular | :x:|
| protectModule(..extensions) | operator | regular | not added | added and authorized | :white_check_mark: |
| protectModule(..extensions) | operator | regular | regular | authorized | :white_check_mark: |
| unProtectModule | methodologist | - | authorized | regular | :white_check_mark: |
| emergencyRemoveProtected | operator | - | authorized | regular | :white_check_mark: |

**...when replacing**

|  Action | Caller | Current Init. | Current Final | Replacement Init. | Replacement Final | Success |
| ----    | -----  | ----          | ----          | ----              | ----              | ----    |
| replaceProtect(..ext) | mutual | authorized | regular | regular/not-added | authorized | :white_check_mark: |
| emergencyReplace(..ext) | mutual | -| -| regular/not-added | authorized | :white_check_mark: |

----

> **Emergencies: state transitions**

+ Because the operator can emergency remove more than one module, emergencies are tracked with
a counter

|Init Emerg # |  Action | Caller | Initial Mod | Final Mod | Final Emerg # | Success |
| --------    | ----    | -----  | ----        | ----      | ----          | ----    |
| 0          | emergencyRemoveProtected | operator | protected | not added | 1 | :white_check_mark: |
| 1          | emergencyRemoveProtected | operator | protected | not added | 2 | :white_check_mark: |
| 1          | emergencyReplaceProtected | mutual | - | protected | 0 | :white_check_mark: |
| 2          | emergencyReplaceProtected | mutual | - | protected | 1 | :white_check_mark: |
| 0          | emergencyReplaceProtected | mutual | - | - | 0 | :x:|
| 1          | resolveEmergency | methodologist | - | - | 0 | :white_check_mark: |
| 2          | resolveEmergency | methodologist | - | - | 1 | :white_check_mark: |
| 0          | resolveEmergency | methodologist | - | - | 0 | :x: |
| 1          | addModule | operator | not added | not added | 1 | :x:|
| 1          | addExtension | operator | not added | not added | 1 | :x:|
| 1          | protectModule| operator | regular | regular | 1 | :x:|


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

| Type  | Name  | Description  |
|------ |------ |------------- |
| boolean | isProtected | true if module can only be called by authorized extensions |
| address[] | authorizedExtensionsList | array of extensions allowed to call this module |
| mapping(address => bool) | authorizedExtensions | map of adapters authorized to call module |

#### Modifiers

> upgradesPermitted

```solidity
/**
  Modifier requiring that contract is not in an "emergency state" following a unilateral operator
  removal of a protected module. In this state, the operator cannot add modules or extensions
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
|uint256 | emergencies | counter that tracks operator's "emergency" removals protected modules. There may be more than one emergency removal happening at any given time |

#### New Methods

> authorizeInitialization:

```solidity
/**
 * ONLY METHODOLOGIST : Called by the methodologist to enable contract. All `interactManager`
 * calls revert until this is invoked. Lets methodologist review and authorize initial protected
 * module settings.
 */
function authorizeInitialization()
  external
  onlyMethodologist
```

> addAuthorizedExtension
- _protectedModule: protected module to add extension for
- _extension: extension which is allowed access to module

```solidity
/**
 * MUTUAL UPGRADE**: Authorizes an extension for a protected module
 */
function authorizeExtension(address _protectedModule, address _extension)
  external
  mutualUpgrade(operator, methodologist);
```

> removeAuthorizedExtension
- _protectedModule: protected module to remove extension for
- _extension: extension whose access to module is revoked

```solidity
/**
 * MUTUAL UPGRADE: revokes an extension's authorization for a protected module
 */
function revokeExtensionAuthorization(address _protectedModule, address _extension)
  external
  mutualUpgrade(operator, methodologist);
```

> protectModule
- _module: Module to protect
- _extensions: Array of adapters to authorize for protected module

```solidity
/**
 * OPERATOR ONLY: The operator uses this when they're adding new functionality and want to
 * assure the methodologist that new features won't be unilaterally changed in the
 * future. Cannot be called during an emergency because methodologist needs to explicitly
 * approve protection arrangements under those conditions.
 *
 * Marks an existing module as protected and authorizes existing adapters for
 * it. Adds module to the protected modules list
 */
function protectModule(address _module, address[] memory _adapters)
  external
  upgradesPermitted
  onlyOperator
```

> unProtectModule
- _module: Module to remove protections for

```soldiity
/**
 * METHODOLOGIST ONLY: Called by the methodologist when they want to cede control over a
 * protected module without triggering an emergency (for example, to remove it because its dead).
 *
 * Marks a currently protected module as unprotected and deletes its authorized adapter registries.
 * Removes old module from the protected modules list.
 *
 * @param  _module          Module to revoke protections for
 */
function unProtectModule(address _module) external onlyMethodologist
```

> emergencyRemoveProtectedModule
- address _module: Module to remove protections for

```solidity
/**
 * OPERATOR ONLY: Called by operator when a module must be removed immediately for security
 * reasons and it's unsafe to wait for a `mutualUpgrade` process to play out.
 *
 * Marks a currently protected module as unprotected and deletes its authorized adapter registries.
 * Removes module from the SetToken. Increments the `emergencies` counter, prohibiting any further
 * operator-only module or extension additions until `emergencyReplaceProtectedModule` decrements
 * `emergencies` back to zero or the methodologist resolves the emergency unilaterally.
 */
function emergencyRemoveProtectedModule(address _module)
  external
  onlyOperator
```

> emergencyReplaceProtectedModule
- _module: Module to protect
- _extensions: List of extensions authorized to call the protected module

```solidity
/**
 * MUTUAL UPGRADE: Replaces a module the operator has removed with `emergencyRemoveProtectedModule`.
 *
 * Adds new module to SetToken. Marks `_newModule` as protected and addes/authorizes new adapters for it.
 * Adds `_newModule` to protectedModules list. Decrements the emergencies counter, restoring
 * operator's ability to add module or adapters unilaterally (if this is the only emergency.)
 */
function emergencyReplaceProtectedModule(address _module, address[] _extensions)
  external
  mutualUpgrade(operator, methodologist)
```

> replaceProtectedModule
- _oldModule: address of module to replace
- _newModule: address of module to add in place of
- _extensions: address array of extensions to authorize for new module

```solidity
/**
 * MUTUAL UPGRADE: Used when methodologists wants to guarantee that an existing protection
 * arrangement is replaced with a suitable substitute (ex: upgrading a StreamingFeeSplitExtension).
 *
 * Marks a currently protected module as unprotected and deletes its authorized extension
 * registries. Removes `_oldModule` from the  `protectedModulesList`. Removes old module from
 * SetToken. Adds new module to SetToken. Marks `_newModule` as protected and adds/authorizes new
 * extensions for it. Adds `_newModule` module to protectedModules list.
 */
function replaceProtectedModule(address _oldModule, address _newModule, address[] _extensions)
  external
  mutualUpgrade(operator, methodologist);
```

> resolveEmergency

```solidity
/**
 * METHODOLOGIST ONLY: Allows a methodologist to exit a state of emergency without replacing a
 * protected module that was unilaterally removed. This could happen if the module has no viable
 * substitute or operator and methodologist agree that restoring normal operations is the
 * best way forward.
 */
function resolveEmergency() external onlyMethodologist {
    require(emergencies > 0, "Not in emergency");
    emergencies -= 1;
}
```

#### Interfaces changes for existing functions

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
 * MUTUAL UPGRADE Update the SetToken manager address
 */
function setManager(address _newManager)
  external
  mutualUpgrade(operator, methodologist)
```

#### New Getters

> getAuthorizedExtensions
- _module: protected module

Returns a list of all extensions authorized for module

```solidity
function getAuthorizedExtensions(address _module) external view returns (address[] memory)
```

> isAuthorizedExtension
- _module: protected module
- _extension: authorized extension

Returns true if extension is authorized for a module

```solidity
function isAuthorizedExtension(address _module, address _extension) external view returns (bool)
```

> getProtectedModules

Returns a list of all protected modules for the manager

```solidity
function getProtectedModules() external view returns (address[] memory)
```

#### Logic changes in existing functions

> interactManager
- module
- data

Method should revert if contract initialization has not been authorized by methodologist

> addModule
- _module

Method should revert if contract is in an emergency state

> addExtension
- _extension

Method should revert if contract is in an emergency state

> removeModule
- _module

Method should revert if a module is protected

> removeExtension
- _extension

Method should revert if extension is authorized by any module


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
