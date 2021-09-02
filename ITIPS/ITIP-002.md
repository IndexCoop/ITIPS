# ITIP-002
*Using template v0.1*
## Abstract
Several upcoming IC products include some form of intrinsic productivity (IP) in their methodologies. These products include yield bearing tokens which generally need to be wrapped and unwrapped via the wrap module. Some of these positions (such as aTokens) have balances that positively rebase as yield is accrued. Other components simply have a variable exchange rate between the underlying and wrapped token, allowing the wrapped token units to remain fixed. Finally, some components accrue yield in a secondary token, such as in the case of cTokens accruing COMP rewards. In order to safely add support for these kinds of positions, new tools must be built to manage these different yield positions.

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
**Reviewer**: @richardliang

## Proposed Architecture Changes
Add a new manager issuance hook called `AirdropIssuanceHook` which has a `invokePreIssueHook` function that uses the `AirdropModule` to sync all rebasing components. In order to add and remove new rebasing tokens, we need to add a new extension, `AirdropExtension` for interacting with the `AirdropModule`.

## Requirements
- Requires no changes to core set protocol contracts
- Issuance hooks should only sync positions for components that are both allowed airdrops and already components of the set
- Anyone can call the hooks to sync positions
- Has hooks for both issuance and redemption
    - Redemption hook not needed right now but future issuance modules may support it
- `AirdropExtension` allows adding and removing airdrop tokens
    - For completeness, add all manager function of `AirdropModule` to this

## Checkpoint 2
**Reviewer**: @richardliang

## Specification
### AirdropIssuanceHook
#### Inheritance
- IManagerIssuanceHook
#### Functions
> constructor(IAirdropModule _airdropModule) public
```solidity
function constructor(IAirdropModule _airdropModule) public {
    airdropModule = _airdropModule;
}
```

> invokePreIssueHook(SetToken _setToken, uint256 _quantity, address _sender, address _to) external
```solidity
function invokePreIssueHook(SetToken _setToken, uint256 _quantity, address _sender, address _to) external override {
    _sync(_setToken);
}
```

> invokePreIssueRedeem(SetToken _setToken, uint256 _quantity, address _sender, address _to) external
```solidity
function invokePreIssueHook(SetToken _setToken, uint256 _quantity, address _sender, address _to) external override {
    _sync(_setToken);
}
```

> _sync(ISetToken _setToken) internal
```solidity
function _sync(ISetToken _setToken) internal {
    address[] memory airdrops = airdropModule.getAirdrops(_setToken);
    airdropModule.batchAbsorb(_setToken, airdrops);
}
```

### AirdropExtension
#### Functions
> constructor(IBaseManager _manager, IAirdropModule _airdropModule) public
```solidity
function constructor(IBaseManager _manager, IAirdropModule _airdropModule) public {
    manager = _manager;
    airdropModule = _airdropModule;
}
```

> initializeAirdropModule(AirdropSettings memory _airdropSettings) external onlyOperator
```solidity
function initializeAirdropModule(AirdropSettings memory _airdropSettings) external onlyOperator {
    // use invokeManager to do this
    airdropModule.initialize(setToken, _airdropSettings);
}
```

> absorb(address _token) external onlyOperator
```solidity
function absorb(address _token) external onlyOperator {
    // use invokeManager to do this
    airdropModule.absorb(setToken, _token);
}
```

> batchAbsorb(address memory _tokens) external onlyOperator
```solidity
function batchAbsorb(address memory _tokens) external onlyOperator {
    // use invokeManager to do this
    airdropModule.batchAbsorb(setToken, _tokens);
}
```

> addAirdrop(address _token) external onlyOperator
```solidity
function addAirdrop(address _token) external onlyOperator {
    // use invokeManager to do this
    airdropModule.addAirdrop(setToken, _token);
}
```

> removeAirdrop(address _token) external onlyOperator
```solidity
function removeAirdrop(address _token) external onlyOperator {
    // use invokeManager to do this
    airdropModule.removeAirdrop(setToken, _token);
}
```

> updateAnyoneAbsorb(bool _anyoneAbsorb) external onlyOperator
```solidity
function updateAnyoneAbsorb(bool _anyoneAbsorb) external onlyOperator {
    // use invokeManager to do this
    airdropModule.updateAnyoneAbsorb(setToken, _anyoneAbsorb);
}
```

> updateFeeRecipient(address _newRecipient) external onlyOperator
```solidity
function updateFeeRecipient(address _newRecipient) external onlyOperator {
    // use invokeManager to do this
    airdropModule.updateFeeRecipient(setToken, _newRecipient);
}
```

> updateAirdropFee(uint256 _newFee) external onlyOperator
```solidity
function updateAirdropFee(uint256 _newFee) external onlyOperator {
    // use invokeManager to do this
    airdropModule.updateAirdropFee(setToken, _newFee);
}
```

## Checkpoint 3
Before we move onto the implementation phase we want to make sure that we are aligned on the spec. All contracts should be specced out, their state and external function signatures should be defined. For more complex contracts, internal function definition is preferred in order to align on proper abstractions. Reviewer should take care to make sure that all stake holders (product, app engineering) have their needs met in this stage.

**Reviewer**: @richardliang

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()
