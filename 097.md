Energetic Lavender Corgi

Medium

# More funds will be lost by borrower on partial Liquidation

### Summary

In the liquidation incentive implementation, the lender's fee is deducted twice.  seizedAsset  = asset + Fee (liq) + Fee(lenders), and lendersAsset is also calculated separately and returned from the ` auditor.calculateSeize` function



### Root Cause

In the file Market.sol https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Market.sol#L606 the calculateSeize function is called in the auditor contract.

```solidity
function calculateSeize(
    Market repayMarket,
    Market seizeMarket,
    address borrower,
    uint256 actualRepayAssets
  ) external view returns (uint256 lendersAssets, uint256 seizeAssets) {
    LiquidationIncentive memory memIncentive = liquidationIncentive;
    lendersAssets = actualRepayAssets.mulWadDown(memIncentive.lenders);

    // read prices for borrowed and collateral markets
    uint256 priceBorrowed = assetPrice(markets[repayMarket].priceFeed);
    uint256 priceCollateral = assetPrice(markets[seizeMarket].priceFeed);
    uint256 baseAmount = actualRepayAssets.mulDivUp(priceBorrowed, 10 ** markets[repayMarket].decimals);

    seizeAssets = Math.min(
      baseAmount.mulDivUp(10 ** markets[seizeMarket].decimals, priceCollateral).mulWadUp(
        1e18 + memIncentive.liquidator + memIncentive.lenders
      ),
      seizeMarket.maxWithdraw(borrower)
    );
  }
```

As we see `lendersAssets` is calculated with `memIncentive.lenders`, the `seizeAssets` is also a function of `memIncentive.lenders`, meaning Approx (actualRepayAssets+ Fee(liq) + 2Fee(lenders)) is removed from the borrower.

### Internal pre-conditions

Liquidator just needs to call the liquidate function to liquidate an asset

### External pre-conditions

_No response_

### Attack Path

_No response_

### Impact

The impact of this is loss of funds for borrower to pay more fees than required which is passed unto the liquidator

### PoC

```solidity
function calculateSeize(
    Market repayMarket,
    Market seizeMarket,
    address borrower,
    uint256 actualRepayAssets
  ) external view returns (uint256 lendersAssets, uint256 seizeAssets) {
    LiquidationIncentive memory memIncentive = liquidationIncentive;
    lendersAssets = actualRepayAssets.mulWadDown(memIncentive.lenders);

    // read prices for borrowed and collateral markets
    uint256 priceBorrowed = assetPrice(markets[repayMarket].priceFeed);
    uint256 priceCollateral = assetPrice(markets[seizeMarket].priceFeed);
    uint256 baseAmount = actualRepayAssets.mulDivUp(priceBorrowed, 10 ** markets[repayMarket].decimals);

    seizeAssets = Math.min(
      baseAmount.mulDivUp(10 ** markets[seizeMarket].decimals, priceCollateral).mulWadUp(
        1e18 + memIncentive.liquidator + memIncentive.lenders
      ),
      seizeMarket.maxWithdraw(borrower)
    );
    // @audit M-2 liquidation will always fail for maxWithdraw as most assets are here and if market fee < lenders fee, it will revert
  }
  
 
```

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Auditor.sol#L280-L292

### Mitigation

Remove the second fee in the `seizedAsset`