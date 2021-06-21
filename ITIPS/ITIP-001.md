# ITIP-[xx]
*Using template v0.1*
## Abstract
We want to be able to increase the trade size for FLI products while maintaining a low target level of slippage. Increasing the trade size allows for FLI products to delever faster and take on greater amounts of capital while not compromising on the saefty of the product. Additionally, larger trade sizes cut down on the operating expenses of maintaining FLI products by reducing the amount of transactions necessary to rebalance.
## Motivation
This feature/upgrade looks to increase the max trade size for FLI products while maintaining the same slippage tolerance. This helps Index Cooperative achieve two main goals:
1. Safely scale FLI products to higher AUM levels
2. Reduce costs of maintaining FLI products, especially in volatile periods, by reducing the amount of transactions necessary to rebalance the product

Safety for FLI products can be defined by how fast a product is able to delever during periods of high volatility. One of the biggest factors determining the delever rate is how much of the AUM of the FLI can be traded within one transaction which can be defined as: `TS/AUM`, where `TS = max trade size` and `AUM = asset under management`. By increasing the max trade size we increase the delever rate thus providing a higher level of safety.  

Targeting max trade size has a follow on benefit of reducing maintenance costs. Another way of increasing safety could be to increase the frequency of rebalance transactions, however this incurs additional maintenance costs as more transaction are required for maintenance. By increasing the max trade size we can reduce the amount of required rebalance transactions while still achieving similar levels of safety.

This feature needs to be able to work for any instance a rebalance needs to be called for FLI products. That means it must be sufficient for day-to-day operations of FLI as well as during periods of high volatility. Additionally this feature must be easily parameterizable by the Index Cooperative and not be able to be exploited by malicious actors looking to use rebalance transactions for their own benefit. We want to allow keepers the ability to submit trades deemed acceptable (if not essentially the best) while eliminating the ability to execute bad trades.

## Background Information
The current iteration of the FLI rebalance strategy, as encoded in the [`FLIStrategyAdapter`](https://github.com/SetProtocol/index-coop-smart-contracts/blob/3b8e8af3391cd4d118baa81fc8920bbe14e76087/contracts/adapters/FlexibleLeverageStrategyAdapter.sol) allows for execution on only one exchange, which can be picked from available AMM-exchange adapters enabled by Set Protocol: `SushiswapExchangeAdapter`, `UniswapV2ExchangeAdapterV2`. There is additional work underway to implement two exchange adapters that may be of great use to FLI products, `UniswapV3ExchangeAdapter` ([PR](https://github.com/SetProtocol/set-protocol-v2/pull/96)) and a TradeSplitter contract ([STIP](https://github.com/SetProtocol/STIPS/pull/5)).

In general, there are two ways to increase the trade size for for a rebalance transaction:
1. Execute trades on an exchange that has deeper liquidity
2. Split trades across multiple exchanges

These new adapters in development provide the possibility of being able to do both, tapping into the the deeper liquidity pools of UniswapV3 and/or splitting trades across multiple exchanges. Digging into how these two adapters will work can help inform how we can build around them:

1. The `UniswapV3ExchangeAdapter` is fairly self-explanatory, it will interact with the UniswapV3 Router and execute trades across it's various liquidity pools. One difference of note is that UniswapV3 requires choosing a fee tier to execute in (since the same asset pair can have multiple pools just with different tiers), so when parameterizing UniswapV3 trades the fee tier will need to be included in the exchange data.
2. The `TradeSplitter` will split between `UniswapV2` and `Sushiswap`, so there will be exposure to liquidity across both of those exchanges. Since it is still WIP it's unclear any specific parameterization requirements the `TradeSplitter` will need.

It's important to note that exchanges perform quite differently depending on the environment. Anecdotally the various exchanges break down in the following way:

| Exchange  	| Stable Market   	| High Volatility 	| Trustlessness 	| Notes                                                                                                	|
|-----------	|-----------------	|-----------------	|---------------	|------------------------------------------------------------------------------------------------------	|
| UniswapV3 	| Highly liquid   	| Low/Med Liquid  	| High          	| During very volatile environments price moves away from liquidity and may not be adjusted            	|
| UniswapV2 	| Med Liquid      	| Med Liquid      	| High          	| Due to algorithm remains very stable regardless of environment. Most reliable.                       	|
| Sushiswap 	| Med Liquid      	| Med Liquid      	| High          	| Due to algorithm remains very stable regardless of environment. Most reliable.                       	|
| 0x        	| High/Med Liquid 	| Med Liquid      	| Med           	| Pulls from other exchanges so lower bound on liquidity. High vol means less liquidity on PMM network 	|

The ideal solution probably takes advantage of the strengths of each of these exchanges while limiting usage during times when they are not effective.

## Open Questions
- [ ] Are there any further guarantees we can give that the best trade will always be executed?
- [ ] How do we deter being sandwich attacked?
    - *Answer*
## Feasibility Analysis
Any solution needs to balance max trade size with reliability that the trade can succeed in all market types. The most resilient system needs to exhibit the following properties: 
1. Be able to have trades executed across any exchange so that a trade is always available no matter the market environment 
2. Not require batching of trades in one transaction in case one exchange reverts
3. Deter malicious actors by enforcing slippage checks in case a certain exchange's liquidity dries up in a given market environment

To that end, it makes sense to update rebalance logic in the FLIStrategyAdapter to allow any exchange to be used for execution, provided it has been parameterized by the Index Cooperative. This effectively operates as on-chain parameterization of trades but off-chain splitting and routing logic. Each exchange can be parameterized with it's own max trade size, slippage checks, and any other data necessary to interact with the exchange. Keepers can call the FLIStrategyAdapter passing in the exchange they wish execute the trade through. This should require only updates to the logic of the FLIStrategyAdapter and can be rolled out immediately. This strategy also has the effect of decreasing the time between rebalance transactions since we can put each exchange on it's own TWAP.

## Timeline
- Spec + review: 3 days
- Implementation: 4-5 days (comprehensive integration tests will be necessary)
- Internal review: 2 days
- Deployment scripts: 1 day
- Deploy to testnet: 1 day
- Testing: 3-5 days
- Write docs: 1-2 days

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**: @felix2feng and @richardliang on 10/11

## Proposed Architecture Changes
This update won't require re-architecting any part of our system though it will require some updates to logic, and a reintroduction of the FLIStrategyViewer contract which can be used to give keepers information on which exchange is the best to use. As a refresher this is the current architecture of the FLI Strategy system:

![Current Architecture](/assets/ITIP-001/current-architecture.png)

We don't currently use a FLIStrategyViewer since the Chainlink oracle upgrade, however here it makes sense to reintroduce it as a resource for keepers to receive information about the best on-chain quote.

Note: We will reintroduce it as a launch +1 item, as viewer contracts can be easily swapped out.

## Requirements
- Must allow for execution across any parameterized exchange, each exchangeName should be parameterized with the following info
    - Exchange name
    - Max trade size
    - Lever exchange data
    - Delever exchange data
    - Incentivized max trade size
- Each exchange should track it's own last trade timestamp
- During regular epoch rebalances, only one rebalance can be called
- Adding a new exchange to execute on should be as simple as adding parameters, deploying a new contract should not be necessary
- There should be some logic that is able to notify keepers of the exchange with the best on-chain quote
## User Flows
To start we will revisit the current rebalance flow:    
### Current Rebalance Flow
1. A keeper wants to rebalance the FLI which last rebalanced a day ago so calls `shouldRebalanceWithBounds` to know which function to call (in this case `rebalance`)
2. The keeper calls `rebalance` passing in no parameters
3. The current leverage ratio is calculated and all params are gathered for the rebalance including trade size
4. Validate leverage ratio isn't above ripcord and that enough time has elapsed since lastTradeTimestamp (unless outside bounds)
5. Validate not in TWAP
6. Calculate new leverage ratio
7. Calculate the total rebalance size and the chunk rebalance size, the chunk rebalance size is less than total rebalance size
8. Create calldata for invoking lever/delever on CLM
9. Log the last trade timestamp
10. Set the twapLeverageRatio so that we don't have to recalculate the target leverage ratio again for this TWAP

### Current Iterate Flow
1. A keeper wants to rebalance the FLI which kicked off a TWAP rebalance a minute ago so calls `shouldRebalanceWithBounds` to know which function to call (in this case `rebalance`)
2. The keeper calls `iterateRebalance` passing in no parameters
3. The current leverage ratio is calculated and all params are gathered for the rebalance including trade size
4. Validate leverage ratio isn't above ripcord and that enough time has elapsed since last lastTradeTimestamp 
5. Validate TWAP is underway
6. Calculate new leverage ratio
7. Check that prices haven't moved making continuing rebalance unnecessary
8. Calculate the total rebalance size and the chunk rebalance size, the chunk rebalance size equals total rebalance size
9. Create calldata for invoking lever/delever on CLM
10. Log the last trade timestamp
11. Delete twapLeverageRatio since the TWAP is over

### Current Ripcord Flow
1. A keeper wants to rebalance the FLI in the middle of a falling market and calls `shouldRebalanceWithBounds` to know which function to call (in this case `ripcord`)
2. The keeper calls `ripcord` passing in no parameters
3. The current leverage ratio is calculated and all params are gathered for the rebalance including trade size
4. Validate that leverage ratio outside bounds and that incentivized cool down period has elapsed from lastTradeTimestamp
5. Calculate the notional amount of the chunk rebalance size
6. Create calldata for invoking delever on CLM
7. Log the last trade timestamp
8. Delete twapLeverageRatio
9. Transfer ether to keeper

### Revised Rebalance Flow
1. A keeper wants to rebalance the FLI which last rebalanced a day ago so calls a function on the FLIViewer that tells it to rebalance _and_ to route the trade through UniswapV3. Note: for initial release, we will not have the FLIViewer contract. It will purely be determined by an offchain algorithm (e.g. naive use Uniswap V3 during normal conditions, use both exchanges when > 2.3x leverage)
2. The keeper calls `rebalance` on the FLIStrategyAdapter passing in `UniswapV3ExchangeAdapter`
3. The _exchange's_ max trade size is grabbed from storage
4. Repeat steps 3-9 from original flow
5. Log the _exchange_ last trade timestamp
6. Set the twapLeverageRatio so that we don't have to recalculate the target leverage ratio again for this TWAP

### Revised Iterate Flow
1. A keeper wants to rebalance the FLI which kicked off a TWAP rebalance one block ago so calls a function on the FLIViewer that tells it to iterate _and_ to route the trade through Sushiswap since the cool down period hasn't elapsed for UniswapV3
2. The keeper calls `iterateRebalance` passing in `SushiswapExchangeAdapter`
3. The _exchange's_ max trade size and last trade timestamp are grabbed from storage
4. Validate leverage ratio isn't above ripcord and that enough time has elapsed since _exchange's_ lastTradeTimestamp 
4. Repeat steps 4-10 from original flow
5. Log the _exchange_ last trade timestamp
6. Delete twapLeverageRatio since the TWAP is over\

### Revised Ripcord Flow
1. A keeper wants to rebalance the FLI in the middle of a falling market and calls a function on the FLIViewer that tells it to ripcord _and_ to route the trade through `Sushiswap`
2. The keeper calls `ripcord` passing in SushiswapExchangeAdapter`
3. The _exchange's_ incentivized max trade size and last trade timestamp are grabbed from storage
3. The current leverage ratio is calculated and all params are gathered for the rebalance including _exchange's_ incentivized trade size
4. Validate that leverage ratio outside bounds and that incentivized cool down period has elapsed from _exchange's_ lastTradeTimestamp
5. Repeat steps 5-7
6. Log the _exchange_ last trade timestamp
7. Repeat steps 8-9

### High-Level Data Structure Update
- Mapping from exchangeName to ExchangeSettings. We will use this data structure even though strings cost more gas. This prevents cases when we can use different exchange settings for the same exchange. This also requires us to store an array of exchanges that are enabled. We are willing to make this tradeoff
- Params removed from ExecutionSettings and put in ExchangeSettings
    - twapMaxTradeSize
    - exchangeName
    - leverExchangeData
    - deleverExchangeData
- Params removed from IncentiveSettings and put in ExchangeSettings
    - incentivizedTwapMaxTradeSize
- Replace all lastTradeTimestamps with globalLastTradeTimestamp for additional clarity
- Iterate rebalance and ripcord rebalances that rely on cooldown periods should use the exchange specific lastTradeTimestamp
- Update both exchange and global last trade timestamp to keep state consistent

#### Table of Should Rebalance Conditions
- Global timestamp will always equal one exchange's last trade timestamp. There will be no case where global timestamp is strictly less than any exchange's last trade timestamp
    - In other words, if global timestamp elapsed is a YES then every exchange timestamp elapsed should be YES
    - If an exchange timestamp is updated from any action, global will always be updated with the same value

| Function     | Has elapsed since global last trade timestamp | Has elapsed since exchange 1 last trade timestamp | Has elapsed since exchange 2 last trade timestamp | Should rebalance |
|-----------    |-----------------  |-----------------  |--------------- |--------------- |
| Rebalance     | Yes     | Yes    | Yes    | Yes |
| Rebalance     | No     | No    | Yes    | No |
| Rebalance     | No     | No    | No    | No |
| IterateRebalance     | Yes     | Yes    | Yes    | Yes |
| IterateRebalance     | No     | No    | Yes    | Yes |
| IterateRebalance     | No     | No    | No    | No |
| Ripcord     | Yes     | Yes    | Yes    | Yes |
| Ripcord     | No     | No    | Yes    | Yes |
| Ripcord     | No     | No    | No    | No |

### Other functions changed or added
- `engage()`
- `disengage()`
- `shouldRebalance`
    - Returns ShouldRebalance enum and exchanges that can be used
- `getTotalRebalanceNotional`
    - Helper to get the expected total notional rebalance size
- `setExchangeSettings`
    - Added to contract
- `_shouldRebalance`
- `getExchangeSettings`

### FLI Viewer (_Post Launch Release_)
1. A keeper wants to know the FLIStrategy function to call and the exchange to use so they call `getFLIFunctionAndExchange`
2. Viewer calls shouldRebalance on FLIStrategyAdapter and receives function to call and list of exchanges that can be used
3. FLIViewer iterates over each exchange doing following:
    - Get parameterization for exchange from FLIStrategyAdapter
    - Get size of rebalance trade
    - Get quote from exchange

### Public Function Interface Changes
- `shouldRebalance() returns ENUM` => `shouldRebalance() returns (string[], ENUM[])`
- `shouldRebalanceWithBounds(uint256 minLeverage, uint256 maxLeverage) returns ENUM` => `shouldRebalanceWithBounds(uint256 minLeverage, uint256 maxLeverage) returns (string[], ENUM[])`
- `rebalance()` => `rebalance(string _exchangeName)`
- `iterateRebalance()` => - `iterateRebalance(string _exchangeName)`
- `ripcord()` => `ripcord(string _exchangeName)`
- `engage()` => `engage(string _exchangeName)`
- `disengage()` => `disengage(string _exchangeName)`

## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: @richardliang

## Specification

### `FlexibleLeverageStrategyAdapter`
#### Inheritance
- `BaseAdapter`
#### Struct
ExchangeSettings

| Type  | Name  | Description  |
|------ |------ |------------- |
| uint256 |twapMaxTradeSize|TWAP Max trade size|
| bytes |leverExchangeData|Arbitrary exchange data for lever|
| bytes |deleverExchangeData|Arbitrary exchange data for delever|
| uint256 |incentivizedTwapMaxTradeSize|Ripcord TWAP Max trade size|
| uint256 |exchangeLastTradeTimestamp|Exchange last trade timestamp which is separate from global variable|

LeverageInfo (Updated)

| Type  | Name  | Description   |
|------ |------ |-------------  |
| string |exchangeName|Store the exchange string|

#### Public Variables
| Type  | Name  | Description   |
|------ |------ |-------------  |
|mapping(string => ExchangeSettings) |exchangeSettings|Mapping of exchange name to settings struct|
|string[]|enabledExchanges|Array of enabled exchange names|
|uint256|globalLastTradeTimestamp (Updated from lastTradeTimestamp)| Last rebalance timestamp|

#### Updated Functions

> function rebalance()
- exchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function rebalance(string memory _exchangeName) external onlyEOA onlyAllowedCaller(msg.sender) {

    LeverageInfo memory leverageInfo = _getAndValidateLeveragedInfo(execution.slippageTolerance, exchangeSettings[_exchangeName].twapMaxTradeSize, _exchangeName);

    // We limit rebalance to only be called once if within bounds so we use the global trade timestamp to prevent multiple exchanges from calling once each
    _validateNormalRebalance(leverageInfo, methodology.rebalanceInterval, globalLastTradeTimestamp);
    
    ...


}
```

> function iterateRebalance()
- exchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function iterateRebalance(string memory _exchangeName) external onlyEOA onlyAllowedCaller(msg.sender) {

    LeverageInfo memory leverageInfo = _getAndValidateLeveragedInfo(execution.slippageTolerance, exchangeSettings[_exchangeName].twapMaxTradeSize, _exchangeName);
    
    // We can use any exchange to iterate rebalances so we use the exchange specific last trade timestamp here
    _validateNormalRebalance(leverageInfo, execution.twapCooldownPeriod, executionSettings[_exchangeName].exchangeLastTradeTimestamp);

    ...

    
}
```

> function ripcord()
- exchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function ripcord(string memory _exchangeName) external onlyEOA onlyAllowedCaller(msg.sender) {
    LeverageInfo memory leverageInfo = _getAndValidateLeveragedInfo(
        incentive.incentivizedSlippageTolerance, 
        exchangeSettings[_exchangeName].incentivizedTwapMaxTradeSize,
        _exchangeName
    );

    // We can use any exchange to ripcord so we use the exchange specific last trade timestamp here
    _validateRipcord(leverageInfo, executionSettings[_exchangeName].exchangeLastTradeTimestamp);

    ...
  
}
```

> function disengage()
- exchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function disengage(string memory _exchangeName) external onlyOperator {
    LeverageInfo memory leverageInfo = _getAndValidateLeveragedInfo(execution.slippageTolerance, exchangeSettings[_exchangeName].twapMaxTradeSize, _exchangeName);

    ...

}
```

> function engage()
- exchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function engage(string memory _exchangeName) external onlyOperator {
    LeverageInfo memory leverageInfo = LeverageInfo({
        action: engageInfo,
        currentLeverageRatio: PreciseUnitMath.preciseUnit(), // 1x leverage in precise units
        slippageTolerance: execution.slippageTolerance,
        twapMaxTradeSize: execution.twapMaxTradeSize,
        exchangeName: _exchangeName
    });

    ...

    
}
```

> function _getAndValidateLeveragedInfo()
- slippageTolerance
- maxTradeSize
- exchangeName
```solidity
function _getAndValidateLeveragedInfo(uint256 _slippageTolerance, uint256 _maxTradeSize, string memory _exchangeName) internal view returns(LeverageInfo memory) {
    // Check the max trade size is >0 to see if exchange is enabled on the adapter
    require(_maxTradeSize > 0, "Must be valid exchange");
    
    ...

    return LeverageInfo({
        action: actionInfo,
        currentLeverageRatio: currentLeverageRatio,
        slippageTolerance: _slippageTolerance,
        twapMaxTradeSize: _maxTradeSize,
        exchangeName: _exchangeName
    });
}
```

> function _validateNormalRebalance()
- leverageInfo
- twapCooldownPeriod
- lastTradeTimestamp
```solidity
function _validateNormalRebalance(LeverageInfo memory _leverageInfo, uint256 _coolDown, uint256 _lastTradeTimestamp) internal view {
    ...

    require(
        block.timestamp.sub(_lastTradeTimestamp) > _coolDown
        || _leverageInfo.currentLeverageRatio > methodology.maxLeverageRatio
        || _leverageInfo.currentLeverageRatio < methodology.minLeverageRatio,
        "Cooldown not elapsed or not valid leverage ratio"
    );
}
```

> function _validateRipcord()
- leverageInfo
- lastTradeTimestamp
```solidity
function _validateRipcord(LeverageInfo memory _leverageInfo, uint256 _lastTradeTimestamp) internal view {
    ...

    // If currently in the midst of a TWAP rebalance, ensure that the cooldown period has elapsed
    require(_lastTradeTimestamp.add(incentive.incentivizedTwapCooldownPeriod) < block.timestamp, "TWAP cooldown must have elapsed");

    // There are cases where incentivized max trade size is set to 0 to disable ripcord for certain exchanges
    require (
        exchangeSettings[_leverageInfo.exchangeName].incentivizedTwapMaxTradeSize > 0,
        "Incentivized TWAP max trade size must be greater than 0"
    );
}
```

> function _delever()
- leverageInfo
- chunkRebalanceNotional

```solidity
function _delever(LeverageInfo memory _leverageInfo, uint256 _chunkRebalanceNotional) internal {
    
    ...

    bytes memory deleverCallData = abi.encodeWithSignature(
        "delever(address,address,address,uint256,uint256,string,bytes)",
        address(strategy.setToken),
        strategy.collateralAsset,
        strategy.borrowAsset,
        collateralRebalanceUnits,
        minRepayUnits,
        _leverageInfo.exchangeName,
        _leverageInfo.deleverExchangeData
    );
}
```

> function _lever()
- leverageInfo
- chunkRebalanceNotional

```solidity
function _lever(LeverageInfo memory _leverageInfo, uint256 _chunkRebalanceNotional) internal {
    
    ...

    bytes memory leverCallData = abi.encodeWithSignature(
        "lever(address,address,address,uint256,uint256,string,bytes)",
        address(strategy.setToken),
        strategy.borrowAsset,
        strategy.collateralAsset,
        borrowUnits,
        minReceiveCollateralUnits,
        _leverageInfo.exchangeName,
        _leverageInfo.leverExchangeData
    );
}
```

> function _deleverToZeroBorrowBalance()
- leverageInfo
- chunkRebalanceNotional

```solidity
function _deleverToZeroBorrowBalance(LeverageInfo memory _leverageInfo, uint256 _chunkRebalanceNotional) internal {
    
    ...

    bytes memory deleverToZeroBorrowBalanceCallData = abi.encodeWithSignature(
        "deleverToZeroBorrowBalance(address,address,address,uint256,string,bytes)",
        address(strategy.setToken),
        strategy.collateralAsset,
        strategy.borrowAsset,
        maxCollateralRebalanceUnits,
        _leverageInfo.exchangeName,
        _leverageInfo.deleverExchangeData
    );
}
```

> function _updateRebalanceState()
- chunkRebalanceNotional
- totalRebalanceNotional
- newLeverageRatio
- exchangeName

```solidity
function _updateRebalanceState(uint256 _chunkRebalanceNotional, uint256 _totalRebalanceNotional, uint256 _newLeverageRatio, string memory _exchangeName) internal {
    globalLastTradeTimestamp = block.timestamp;

    executionSettings[_exchangeName].exchangeLastTradeTimestamp = block.timestamp;
    
    ...

}
```

> function _updateIterateState()
- chunkRebalanceNotional
- totalRebalanceNotional
- exchangeName

```solidity
function _updateIterateState(uint256 _chunkRebalanceNotional, uint256 _totalRebalanceNotional, string memory _exchangeName) internal {
    globalLastTradeTimestamp = block.timestamp;

    executionSettings[_exchangeName].exchangeLastTradeTimestamp = block.timestamp;
    
    ...

}
```

> function _updateRipcordState()
- exchangeName

```solidity
function _updateRipcordState(string memory _exchangeName) internal {
    globalLastTradeTimestamp = block.timestamp;

    executionSettings[_exchangeName].exchangeLastTradeTimestamp = block.timestamp;
    
    ...

}
```

> function _validateSettings()
- methodology
- execution
- incentive

```solidity
function _validateSettings(MethodologySettings memory _methodology, ExecutionSettings memory _execution, IncentiveSettings memory _incentive) internal pure {

    // Remove below as this. We may disable ripcord functionality for certain exchanges, and we do that by setting incentivizedTwapMaxTradeSize to 0. As a result, we need to ensure incentivizedTwapMaxTradeSize > 0 in the validateRipcord internal function
    require (
        _execution.twapMaxTradeSize <= _incentive.incentivizedTwapMaxTradeSize,
        "TWAP max trade size must be less than incentivized TWAP max trade size"
    );

}
```

> function _shouldRebalance()
- currentLeverageRatio
- minLeverageRatio
- maxLeverageRatio

```solidity
function _shouldRebalance(uint256 _currentLeverageRatio, uint256 _minLeverageRatio, uint256 _maxLeverageRatio) internal view returns (string[] memory, ShouldRebalance[] memory) {
    ShouldRebalance[] shouldRebalanceEnums;

    for (uint256 i = 0; i < enabledExchanges.length; i++) {
        // If above ripcord threshold, then check if incentivized cooldown period has elapsed for the enabled exchange
        if (_currentLeverageRatio >= incentive.incentivizedLeverageRatio) {
            if (exchangeSettings[enabledExchanges[i]].exchangeLastTradeTimestamp.add(incentive.incentivizedTwapCooldownPeriod) < block.timestamp) {
                shouldRebalanceEnums.push(ShouldRebalance.RIPCORD);
            }
        } else {
            // If TWAP, then check if the cooldown period has elapsed for the enabled exchange
            if (twapLeverageRatio > 0) {
                if (exchangeSettings[enabledExchanges[i]].exchangeLastTradeTimestamp.add(execution.twapCooldownPeriod) < block.timestamp) {
                    shouldRebalanceEnums.push(ShouldRebalance.ITERATE_REBALANCE);
                }
            } else {
                // If not TWAP, then check if the rebalance interval has elapsed OR current leverage is above max leverage OR current leverage is below
                // min leverage for the GLOBAL timestamp to prevent multiple exchanges from calling when within bounds
                if (
                    block.timestamp.sub(globalLastTradeTimestamp) > methodology.rebalanceInterval
                    || _currentLeverageRatio > _maxLeverageRatio
                    || _currentLeverageRatio < _minLeverageRatio
                ) {
                    shouldRebalanceEnums.push(ShouldRebalance.REBALANCE);
                }
            }
        }

         // If none of the above conditions are satisfied, then should not rebalance
        shouldRebalanceEnums.push(ShouldRebalance.NONE);
    }

    return (enabledExchanges, shouldRebalanceEnums)
}
```

> function addEnabledExchange()
- exchangeName
- exchangeSettings

```solidity
function addEnabledExchange(string memory _exchangeName, ExchangeSettings memory _newExchangeSettings) external onlyOperator noRebalanceInProgress {
    exchangeSettings[_exchangeName] = _newExchangeSettings;

    enabledExchanges.push(_exchangeName);
}
```

> function removeExchange()
- exchangeName

```solidity
function removeExchange(string memory _exchangeName) external onlyOperator noRebalanceInProgress {
    delete exchangeSettings[_exchangeName];

    enabledExchanges.remove(_exchangeName);
}
```

> function getTotalRebalanceNotional()

```solidity
function getTotalRebalanceNotional() external view returns (uint256) {
    // Either use existing internal functions or recreate logic that reduces unnecessary calls
}
```

### `FLIRebalanceViewer` [WIP]

FLI Rebalance viewer that compares prices between Uniswap V3 and Uniswap V2 (or router with matching interface) and returns the exchangeName with better prices and the rebalance action in the FlexibleLeverageStrategyAdapter. Note: this is only limited to comparing one V3 and one V2-like exchange (Uniswap V2, Sushiswap, TradeSplitter)

#### Enums
FLIRebalanceAction

| Name  | Description  |
|------ |------------- |
|NONE|Indicates no rebalance action can be taken|
|REBALANCE|Indicates rebalance() function can be successfully called|
|ITERATE_REBALANCE|Indicates iterateRebalance() function can be successfully called|
|RIPCORD|Indicates ripcord() function can be successfully called|

#### Public Variables
| Type  | Name  | Description   |
|------ |------ |-------------  |
|IFLIStrategyAdapter|strategyAdapter|FLI Strategy adapter that uses multiple exchanges|
|IUniswapV2Router |uniswapV2Router|Uniswap V2 router or router with matching interface|
|IUniswapV3Quoter|uniswapV3Quoter|Uniswap V3 get quote contract |
|string|uniswapV3ExchangeName|Human readable string of the V3 exchange in the Set system|
|string|uniswapV2ExchangeName|Human readable string of the V2 exchange (Sushi, Uni V2, TradeSplitter) that conforms to the same get quote interface in the Set system|

#### Functions
> function constructor()
- fliStrategyAdapter: Address of FLI adapter contract
- uniswapV3Quoter: Uniswap V3 get quote contract
- uniswapV2Router: Uniswap V2 or router with matching interface
- uniswapV3ExchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule
- uniswapV2ExchangeName: name of exchange registered in the IntegrationRegistry for the CompoundLeverageModule

```solidity
function constructor(
    IFLIStrategyAdapter _fliStrategyAdapter,
    IUniswapV3Quoter _uniswapV3Quoter,
    IUniswapV2Router _uniswapV2Router,
    string memory _uniswapV3ExchangeName,
    string memory _uniswapV2ExchangeName
)
    public
{

    // Set public variables

}
```

> function shouldRebalanceWithBounds()
- customMinLeverageRatio
- customMaxLeverageRatio

```solidity
// Note: we return 1 exchange and 1 FLIRebalanceAction in these arrays, but we match the interface in the FLI strategy adapter to minimize keeper bot logic
function shouldRebalanceWithBounds(
    uint256 _customMinLeverageRatio,
    uint256 _customMaxLeverageRatio
)
    external
    view
    returns(string memory [], FLIRebalanceAction[])
{
    // Get total notional to rebalance from FLI adapter

    // Get quote from Uniswap V3 SwapRouter

    // Get quote from Uniswap V2 Router

    // Loop through 
}
```




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
