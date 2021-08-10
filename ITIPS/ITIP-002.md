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
    - Yearn lets you zap into their vaults. Should we hook into their zap functionality for rebalancing?
- When do we need to sync positions?
    - At minimum before issuance, redemption, and after claiming COMP and AAVE rewards
    - Do we also need to sync positions before each use of the TradeModule or WrapModule?
## Feasibility Analysis
Add a IntrinsicProductivityExtension for managing the wrapping, unwrapping, and claiming processes
- Support interacting with the wrap module
- Support interacting with the ClaimModule and AirdropModule for claiming AAVE and COMP rewards

Add a SyncPositionsHook for syncing positions
- Used before issuance, redemption, and after claiming AAVE and COMP tokens

Add YearnZapIndexExchangeAdapter
- Allows us to zap into and out of yearn vaults to/from WETH
- No need to use a zap to a curve LP and a wrap to the yearn vault. Can all be both be done simultaneously while rebalancing

## Timeline
finish spec: 8/13
implementation: 8/20
internal audit: 8/23
external audit: 8/30

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

## Proposed Architecture Changes
A diagram would be helpful here to see where new feature slot into the system. Additionally a brief description of any new contracts is helpful.
## Requirements
These should be a distillation of the previous two sections taking into account the decided upon high-level implementation. Each flow should have high level requirements taking into account the needs of participants in the flow (users, managers, market makers, app devs, etc) 
## User Flows
- Highlight *each* external flow enabled by this feature. It's helpful to use diagrams (add them to the `assets` folder). Examples can be very helpful, make sure to highlight *who* is initiating this flow, *when* and *why*. A reviewer should be able to pick out what requirements are being covered by this flow.
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
