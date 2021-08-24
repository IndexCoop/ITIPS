# ITIP-002
## Abstract
Several upcoming IC products include some form of intrinsic productivity (IP) in their methodologies. In order to safely add these features, new tools must be built to manage these different yield positions.
## Motivation
Adding IP to products cannot be done at just the smart contract level. We must also describe new processes for rebalancing and issuance/redemption. These features and processes will be required by the operators of IC products to set up an maintain products that include IP.

## Background Information
There are multiple kinds of ways we may handle yield for a Set.
- Wrapped positions with an exchange rate (position units are fixed)
- Wrapped position with a rebasing mechanism (position units not fixed)
- Wrapped positions with a claim function (some different token is claimed and must have its position added after)

The wrap module does not allow slippage checks. This can potentially cause problems during wrapping into positions that can experience slippage such as curve LP positions. This can be mitigated by using the TradeModule for these positions instead.

## Open Questions
- How do we handle rebalancing curve LP positions?
    - potential solution involves trading WETH -> stablecoin -> 3Pool LP position -> curve metapool LP position
    - easier solution (but not necessarily lowest slippage) is WETH -> stablecoin -> curve metapool LP position
    - Yearn lets you zap into their vaults. Should we hook into their zap functionality for rebalancing?
- When do we need to sync positions?
    - At minimum before issuance, redemption, and after claiming COMP and AAVE rewards
    - Do we also need to sync positions before each use of the TradeModule or WrapModule?
    - How can we prevent users from arbing our calls to claim tokens such as AAVE and COMP?
- Do we expect negative rebases on the tokens? `AirdropModule` will fail if the token negatively rebases.

## Feasibility Analysis
Add a IntrinsicProductivityExtension for managing the wrapping, unwrapping, and claiming processes
- Support interacting with the wrap module
- Support interacting with the ClaimModule and AirdropModule for claiming AAVE and COMP rewards

Syncing positions before issuance and redemption
- Add new `AirdropModule` pre and post issuance hooks to `DebtIssuanceModule` for syncing positions
    - Used before issuance, redemption, and after claiming AAVE and COMP tokens
- Could build a custom `SyncModule` which does a similar thing
    - This might have some benefits since we wouldn't need to worry about compatibility between `DebtIssuanceModule` and `AirdropModule`
- Could also just write a RebaseTokenIssuanceModule which syncs positions before issuance and redemption
    - Probably overkill and not very modular

Handling Curve
- Can use Zapper to zap into and out of yearn vaults (including curve yearn vaults) to/from WETH
    - No exact output zap in
    - No exact output zap out
- Can also just rebalance into any token in the curve pool
    - after rebalance use AMMModule module to enter curve LP
    - then use wrap module to enter yearn vault


## Timeline
finish spec: 8/13
implementation: 8/20
internal audit: 8/23
external audit: 8/30

## Checkpoint 1
**Reviewer**:

## Proposed Architecture Changes
To launch products with intrinsic productivity, we will focus on only handling issuance and redemption to start. It will likely be at least a month (if not longer) after launching until we need to handle rebalances. Handling rebalances only requires upgrading the `AirdropModule` to be act as a module issuance/redemption hook. First, on initialization of the `AirdropModule` it must be able to optionally register with the `DebtIssuanceModule` by calling `registerToIssuanceModule`. Additionally we must add a `moduleIssueHook` and `moduleRedeemHook` to `AirdropModule` which syncs all the the rebasing tokens.

## Requirements
- Only require changes to `AirdropModule`
- Issuance hooks should only sync positions for components that are both allowed airdrops and already components of the set
- When initializing the `AirdropModule` it should allow users to disable registering itself as a hook to the `DebtIssuanceModule`

## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

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
