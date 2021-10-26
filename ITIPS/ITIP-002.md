# ITIP-002
*Using template v0.1*
## Abstract
Our suite of trustless leverage products based on Compound, ETH2x-FLI and WBTC2x-FLI,
have exceeded our expectations and are our most profitable products. However, Compound does not support other high volume ERC20 tokens such as LINK, YFI, DPI etc. Also, the cost of maintenance of FLI products is quite high on the Ethereum blockchain, and we need to launch products on layer 2 scaling solutions, to balance those costs.

## Motivation

#### Feature
A strategy extension that encodes the FLI methodology and is usable with Aave, another lending protocol similar to Compound, would allow us to launch more FLIs (Chainlink)

#### Why is this feature necessary?
1. To launch the LINK 2x FLI which is only available on Aave. LINK has over $1B in liquidity and is the most traded ERC20 token higher than UNI, MATIC etc.
2. To prepare to deploy on Polygon

#### Who is this feature for and how is it intended to be used?
This feature is to enable Index Coop to launch more FLI products on Aave and Polygon

## Background Information

#### Differences between Compound and Aave FLI
In the FlexibleLeverageStrategyExtension for Compound, we need to call the Comptroller directly to fetch the collateralFactor (LTV) to calculate maxBorrow. This appears to be the only change required to support Aave.

```solidity
( , uint256 collateralFactorMantissa, ) = strategy.comptroller.markets(address(strategy.targetCollateralCToken));
```

In Aave, there are 2 %s to be aware of:
- Max LTV: the loan to value of the asset (collateralFactor in Compound). This means we can only borrow up to this amount when levering
- liquidationThreshold: the liquidation threshold, which is a buffer between the max LTV and the % we actually get liquidated. Even though the max LTV restricts our borrow amount, we are still able to redeem assets up to the liquidation threshold. This may happen when delevering. E.g. MATIC LTV is 50% (max 2x leverage) but liquidation threshold is 65% (max 2.85x leverage)

We must change the logic to fetch the LTV for lever and liquidationThreshold for delever in the `_calculateMaxBorrowCollateral` function

#### Similarities between Compound and Aave FLI
The AaveLeverageModule will have the same function interfaces as the CompoundLeverageModule

Both Aave and Compound use Chainlink oracles

## Open Questions
- [ ] What admin functions are there that can break our implementation?

    - *Answer*

## Feasibility Analysis
The interfaces for the AaveLeverageModule will be exactly the same as the CompoundLeverageModule per requirements of the module. However, we must account for the collateralFactor (Compound) vs LTV / liquidation threshold (Aave).

## Timeline
- Spec + review: 2 days
- Implementation: 4-5 days (comprehensive integration tests will be necessary)
- Internal review: 2 days
- Deployment scripts: 1 day
- Deploy to testnet: 1 day
- Testing: 3-5 days
- Write docs: 1-2 days

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**: 

## Proposed Architecture Changes
This update won't require re-architecting any part of our system though it will require some updates to logic to account for using the `ltv` to calculate `maxBorrow` for lever (borrow variable debt) and `liquidationThreshold` to calculate `maxBorrow` for delever (withdraw aToken)

![Current Architecture](/assets/ITIP-001/current-architecture.png)

We will keep interfaces the same so our keepers and `FLIRebalanceViewer` contract can be used.

## Requirements
- Same interfaces / functions as FLI strategy extension
- Logic to use liquidation threshold for delever and ltv for lever

## User Flows
The flows are the same as the CompoundLeverageModule

### Rebalance Flow
1. A keeper wants to rebalance the FLI which last rebalanced a day ago so calls a function on the FLIViewer that tells it to rebalance _and_ to route the trade through UniswapV3. Note: for initial release, we will not have the FLIViewer contract. It will purely be determined by an offchain algorithm (e.g. naive use Uniswap V3 during normal conditions, use both exchanges when > 2.3x leverage)
2. The keeper calls `rebalance` on the FLIStrategyAdapter passing in `UniswapV3ExchangeAdapter`
3. The _exchange's_ max trade size is grabbed from storage
4. The current leverage ratio is calculated and all params are gathered for the rebalance including trade size
5. Validate leverage ratio isn't above ripcord and that enough time has elapsed since lastTradeTimestamp (unless outside bounds)
6. Validate not in TWAP
7. Calculate new leverage ratio
8. Calculate the total rebalance size and the chunk rebalance size, the chunk rebalance size is less than total rebalance size
9. Create calldata for invoking lever/delever on CLM
10. Log the last trade timestamp
11. Log the _exchange_ last trade timestamp
12. Set the twapLeverageRatio so that we don't have to recalculate the target leverage ratio again for this TWAP

### Iterate Flow
1. A keeper wants to rebalance the FLI which kicked off a TWAP rebalance one block ago so calls a function on the FLIViewer that tells it to iterate _and_ to route the trade through Sushiswap since the cool down period hasn't elapsed for UniswapV3
2. The keeper calls `iterateRebalance` passing in `SushiswapExchangeAdapter`
3. The _exchange's_ max trade size and last trade timestamp are grabbed from storage
4. Validate leverage ratio isn't above ripcord and that enough time has elapsed since _exchange's_ lastTradeTimestamp 
5. Validate leverage ratio isn't above ripcord and that enough time has elapsed since last lastTradeTimestamp 
6. Validate TWAP is underway
7. Calculate new leverage ratio
8. Check that prices haven't moved making continuing rebalance unnecessary
9. Calculate the total rebalance size and the chunk rebalance size, the chunk rebalance size equals total rebalance size
10. Create calldata for invoking lever/delever on CLM
11. Log the last trade timestamp
12. Log the _exchange_ last trade timestamp
13. Delete twapLeverageRatio since the TWAP is over\

### Ripcord Flow
1. A keeper wants to rebalance the FLI in the middle of a falling market and calls a function on the FLIViewer that tells it to ripcord _and_ to route the trade through `Sushiswap`
2. The keeper calls `ripcord` passing in SushiswapExchangeAdapter`
3. The _exchange's_ incentivized max trade size and last trade timestamp are grabbed from storage
4. The current leverage ratio is calculated and all params are gathered for the rebalance including _exchange's_ incentivized trade size
5. Validate that leverage ratio outside bounds and that incentivized cool down period has elapsed from _exchange's_ lastTradeTimestamp
6. Calculate the notional amount of the chunk rebalance size
7. Create calldata for invoking delever on CLM
8. Log the last trade timestamp
9. Log the _exchange_ last trade timestamp
10. Delete twapLeverageRatio
11. Transfer ether to keeper

### High-Level Data Structure Update
- Update getting the collateralFactor from the Comptroller, we call `getReserveConfigurationData` in Aave's `ProtocolDataProvider` contract to which returns the LTV and liquidationThreshold for a given asset
- Use the LTV for if `_isLever` is true and liquidationThreshold if `isLever` is false

### Functions changed or added
- `constructor()`
- `_calculateMaxBorrowCollateral()`

## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: 

## Specification

### `AaveFLIStrategyExtension`
#### Inheritance
- `BaseAdapter`
#### Struct

ContractSettings (Updated)

| Type  | Name  | Description   |
|------ |------ |-------------  |
| IAaveLeverageModule |leverageModule|Address of Aave leverage module|
| IAaveProtocolDataProvider |aaveProtocolDataProvider|Address of Aave data provider|

#### Updated Functions

> function _calculateMaxBorrowCollateral()
- ActionInfo memory actionInfo
- bool isLever

```solidity
function _calculateMaxBorrowCollateral(ActionInfo memory _actionInfo, bool _isLever) internal view returns(uint256) {
    // Retrieve collateral factor which is the % increase in borrow limit in precise units (75% = 75 * 1e16)
    ( , uint256 maxLtv, uint256 liquidationThreshold ) = strategy.aaveProtocolDataProvider.getReserveConfigurationData(address(strategy.collateralAsset));

    if (_isLever) {
        uint256 netBorrowLimit = _actionInfo.collateralValue
            .preciseMul(maxLtv)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

        return netBorrowLimit
            .sub(_actionInfo.borrowValue)
            .preciseDiv(_actionInfo.collateralPrice);
    } else {
         uint256 netRedeemLimit = _actionInfo.collateralValue
            .preciseMul(liquidationThreshold)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

        return _actionInfo.collateralBalance
            .preciseMul(netBorrowLimit.sub(_actionInfo.borrowValue))
            .preciseDiv(netBorrowLimit);
    }
}
```


## Checkpoint 3
Before we move onto the implementation phase we want to make sure that we are aligned on the spec. All contracts should be specced out, their state and external function signatures should be defined. For more complex contracts, internal function definition is preferred in order to align on proper abstractions. Reviewer should take care to make sure that all stake holders (product, app engineering) have their needs met in this stage.

**Reviewer**: 

## Implementation
[Link to implementation PR](https://github.com/SetProtocol/index-coop-smart-contracts/commit/653284ed8cb52b97c2f099e598c4cc659255fc4d#diff-877652a4ec96d853d63bbbe0258a8d36c6b3d9557d6d29d4d20d8eb5e01353d6)
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR](https://github.com/SetProtocol/index-deployments/pull/80)
