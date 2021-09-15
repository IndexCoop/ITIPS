# ITIP-003
*Using template v0.1*
## Abstract
Several upcoming IC products include some form of intrinsic productivity (IP) in their methodologies. IP include yield bearing tokens which take several different forms. While ITIP-002 details how to enable issuance and redemption for these kinds of products, this ITIP will describe the rebalancing process.

## Motivation
In order to deliver products with IP, it must be possible to perform rebalances. Since wrapped tokens generally have poor DEX liquidity, rebalances will require the wrapping and unwrapping of tokens. This requires writing new manager extensions. This feature will be used exclusively by Sets utilizing the `BaseManagerV2` contracts.

## Background Information
For more information on handling issuance and redemption of IP products, refer to https://github.com/SetProtocol/ITIPS/pull/8

IP Rebalance Process:
1. Unwrap all components
2. Perform trades
3. Wrap all components

## Open Questions
- How much structure should the extensions provide for the manager?
- Will wrapping and unwrapping effect arbitrage bots?
- Should we consider interactions with `AmmModule` (needed for PAY)?
- Do we want to allow for partial wrapping/unwrapping (not wrapping/unwrapping full balance)?

## Feasibility Analysis
Intrinsic Productivity Tokens

|Token|Protocol|Module|Adapter|Notes|
|-----|--------|------|-------|-----|
|aTokens|Aave|WrapModuleV2|AaveV2WrapV2Adapter|rebasing|
|cTokens|Compound|WrapModuleV2|CompoundWrapV2Adapter||
|yearn vaults|Yearn|WrapModuleV2|YearnWrapV2Adapter||
|imUSD|mStable|WrapModuleV2|MStableWrapV2Adapter|not built yet|
|curve LP tokens|Curve|AmmModule|CurveAmmAdapter|not built yet, might be able to avoid using the deposit module by zapping in|

Note: tokens with lockups will not be supported

### Option 1: Direct interface for interacting with the `WrapModuleV2`
- Simple to implement
- Requires many multisig transactions to execute rebalances

### Option 2: Direct interface with `WrapModuleV2` with extra batching functions
- Simple to implement
- Only requires one multisig transaction
- If one wrap/unwrap action fails, whole transaction reverts
- Might still need more than one multisig transaction if we run into gas limit issues

### Option 3: Single extension for wrapping/unwrapping and trading (`IPExtension`)
- Extra complexity
- Only requires one multisig transaction
- Better abstraction
- Actual wrapping and unwrapping action can be permissionless (similar to how `GeneralIndexModule` works)

Since wrapped components all have an exchange rate, this system can be built by just specifying the target units of the underlying components. In order to fetch exchange rates, a helper contract of interface`IWrapOracle` can be added to have a `getExchangeRate` function. To handle cases where both an underlying component can correlate to two different wrapped components (such as a set that contains aUSDC and cUSDC) or cases where a set contains both a wrapped and raw asset, we must also specify the percentage of the target units that will be wrapped.

Below is the outline for executing a rebalance through this process:

1. Ensure the extension knows how to wrap and unwrap each component
    - This would only need to be done when a new wrapped component is added. After that it will be saved between rebalances
    - Stores the wrapped component, underlying component, and wrap adapter name
2. Parametrize the rebalance
    - Pass in the target underlying units as well as the percentage of each underlying unit that should be rewrapped into each corresponding wrapped unit
        - If a components target units are being set to 0, store MAX_UINT_256 as the amount to unwrap. This allows us to enforce that 100% of a token is being unwrapped in step 3 (which might otherwise not be the case if a token positively rebases between steps 2 and 3).
    - By using the exchange rates, we can calculate the approximate amount of underlying components that the set contains. From there, we can calculate how much of each wrapped component to unwrap. Need to call `AirdropModule` here to handle rebasing tokens.
    - Using simple algebra, we can calculate the target units for the unwrapped components, and use that to parametrize the trading portion of the rebalance similar to how the `GIMExtension` works.
    - When wrapping components after the trading portion has completed, we use a percentage based system to handle the cases where both a wrapped unit and its underlying should be components of the final set or when the final set contains two wrapped components with the same underlying. In sets where this is not the case, the operator will provide 100% as the amount to wrap.
3. Execute unwrap
    - Call `executeUnwrap` on `IPExtension`
    - Goes through the list of components to unwrap calculated in the previous step and unwraps one per call
    - Can be marked `onlyAllowedCaller`
4. Execute trades
    - Call `startRebalance` on `IPExtension`
        - does not require any parameters since everything has been parametrized in step 2
    - Perform trades through `GeneralIndexModule` exactly as they are done in a normal simple index rebalance
5. Execute wrap
    - Call `executeWrap` on `IPExtension`
    - Goes though the list of components to wrap and calculates the amount to rewrap using the percentages supplied in step 2.
    - Can be marked `onlyAllowedCaller`

Notes:
- Logic for lossy wrapping/unwrapping
    - since rebalancing trades are lossy anyway, not receiving exactly the correct unwrapped units won't cause any major problems
    - since we provide percentages to re-wrap rather than absolute amounts, we do not need to worry about not receiving an exact amount of wrapped units back
    - in the case of tokens that should not be wrapped or unwrapped at certain times, we can add `shouldWrap` and `shouldUnwrap` functions to `IWrapOracle` that can prevent the execution of a suboptimal wrap or unwrap.
- Adding and removing components can be handled during steps 1 and 2.
    - Adding a new component
        - Add the adapter to use in step 1
        - Perform step 2 as normally with the new component and the percentage to wrap in the component list
    - Removing a component
        - Set target units to 0 in step 2

Changes to rebalancing utilities:
- Methodologists will provide percentage based weights.
- Since the contract requires units be given in the underlying components, calculating units should be easy by fetching underlying component prices.
- Additional logic is to handle the cases where the percentage to re-wrap is not 100%
    - Can be handled programmatically
    - Methodologist should specify the underlying units as well as what it is being wrapped into
    - In the case where an underlying unit is being wrapped into multiple wrapped components (or only partially wrapped), calculate the correct re-wrap percentage to supply

#### Example 1: Rebalance between wrapped components
|Start Component|Start Weight|End Component|End Weight|
|---------------|------------|-------------|----------|
|aWBTC|50%|aWBTC|30%|
|aDAO|50%|aDAI|70%|

a. Pass in the target underlying units and amounts to be wrapped for each (assume BTC=$50k, DAI=$1, SET=$100)
|Underlying|Underlying Target Units|Percentage Wrapped|
|----------|-----------------------|------------------|
|wBTC|0.3 * 100 / 50000 * 10^8 = 0.0006 * 10^8|100%|
|DAI|0.7 * 100 / 1 * 10^18 = 70 * 10^18|100%|

b. Calculate the amount to unwrap by utilizing the exchange rate from unwrapped to wrapped tokens (aToken exchange rate is 1 to 1)  
- underlying is calculated by: exchangeRate * currentWrappedUnits  
- amount to unwrap is calculated by: max(0, currentUnderlying - targetUnderlying)  

|Wrapped|Underlying|Exchange Rate|Underlying Amount| Amount to Unwrap|
|-------|----------|-------------|-----------------|-----------------|
|aWBTC|wBTC|1|1 * (0.5 * 100 / 50000 * 10^8) = 0.001 * 10^8|max(0, 0.001 - 0.0006) = 0.0004|
|aDAI|DAI|1|1 * (0.5 * 100 / 1 * 10^18) = 50 * 10^18|max(0, 50 - 70) = 0|

c. Calculate target units for the GIM rebalance  
- if it is a wrapped component, target units are always equal to the amount of wrapped units that will remain after unwrap step
- if it is an underlying component, target units are: (1/exchangeRate) * finalUnderlyingUnits - startingUnderlyingUnits  

|Component|Target Units|
|---------|------------|
|wBTC|max(0, (1/1) * 0.0006 - 0.0006 = 0|
|DAI|(1/1) * 70 - 50 = 20|
|aWBTC|0.0006|
|aDAI|50|

d. Calculate amount to rewrap
- since executing the trades can cause slippage, this step should be done after the rebalance via GIM.
- amount to wrap will just be equal to the total underlying amount since percentage for each component is 100%

|Component|Amount to Wrap|
|---------|--------------|
|wBTC|0|
|DAI|20|



## Timeline
TBD

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
