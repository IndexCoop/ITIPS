# ITIP-002
*Using template v0.1*
## Abstract
Several upcoming IC products include some form of intrinsic productivity (IP) in their methodologies. In order to safely add these features, new tools must be built to manage these different yield positions.

## Motivation
Adding IP to products cannot be done at just the smart contract level. We must also describe new processes for rebalancing and issuance/redemption. These features and processes will be required by the operators of IC products to set up an maintain products that include IP.

## Background Information
There are multiple kinds of ways we may handle yield for a Set.
- Wrapped positions with an exchange rate (position units are fixed)
- Wrapped position with a rebasing mechanism (position units not fixed)
- Wrapped positions with a claim function (some different token is claimed and must have its position added after)

Under normal operation (not during rebalances), the only type of wrapped position that is not compatible with default sets is rebasing tokens. Before issuance and redemption, these tokens must have their balances updated.

## Open Questions
- Do we want to handle negative rebases? `AirdropModule` will fail when attempting to sync in this case.
- Do we want to automatically sync positions before redemptions
    - Not strictly needed if a token never negatively rebases since at the worst we will slightly short change redeemers.
    - If we make the sync function public arbitrage bots can call it before redeeming if they want to receive the maximum returns.

## Feasibility Analysis
- Add module hooks to `AirdropModule` for pre issuance and redemption to sync positions
    - Requires changes to the core protocol
    - Can cause compatibility problems between different modules
    - Needs to use `DebtIssuanceModule`
- Write a RebaseTokenIssuanceModule which syncs positions before issuance and redemption
    - Probably overkill and not modular
- Write a custom manager issuance hook which syncs positions before issuance
    - Compatible with `BasicIssuanceModule`
    - Modular
    - No complex compatibility interactions with other modules

## Timeline
8/25: specification  
8/27: implementation  
8/30: internal audit
9/2: deployment scripts

## Checkpoint 1
**Reviewer**:

## Proposed Architecture Changes
Add a new manager issuance hook called `SyncIssuanceHook` which has a `invokePreIssueHook` function that uses the `AirdropModule` to sync all rebasing components.

## Requirements
- Requires no changes to core set protocol contracts
- Issuance hooks should only sync positions for components that are both allowed airdrops and already components of the set

## Checkpoint 2
**Reviewer**:

Reviewer: []
## Specification
### [Contract Name]
#### Inheritance
- List inherited contracts
#### Structs
| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|address|manager|Address of the manager|
|uint256|iterations|Number of times manager has called contract|  
#### Constants
| Type 	| Name 	| Description 	| Value 	|
|------	|------	|-------------	|-------	|
|uint256|ONE    | The number one| 1       	|
#### Public Variables
| Type 	| Name 	| Description 	|
|------	|------	|-------------	|
|uint256|hodlers|Number of holders of this token|
#### Modifiers
> onlyManager(SetToken _setToken)
#### Functions
> issue(SetToken _setToken, uint256 quantity) external
- Pseudo code
## Checkpoint 3
Before we move onto the implementation phase we want to make sure that we are aligned on the spec. All contracts should be specced out, their state and external function signatures should be defined. For more complex contracts, internal function definition is preferred in order to align on proper abstractions. Reviewer should take care to make sure that all stake holders (product, app engineering) have their needs met in this stage.

**Reviewer**:

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()
