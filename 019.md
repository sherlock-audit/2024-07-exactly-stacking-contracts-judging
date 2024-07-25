Fit Boysenberry Stallion

Medium

# DoS on deposits when a low savings amount is harvested

### Summary

The missing check in `StakedEXA::harvest` will cause a DoS for new depositors as the function will revert whenever the amount deposited back into the market (savings) is low enough to not mint any share. 

### Root Cause

In `StakedEXA:358` there is a missing check that will cause a DoS for new depositors whenever the amount deposited back into the market (savings) is low enough to not mint any share. 

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L358

### Internal pre-conditions

1. The [`save`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L357) amount calculated in `harvest` must be low enough to not mint any share when [deposited back](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L358) into the market. When the market has been active for some time, it's usual that the exchange rate is quite high, causing a deposit of a few wei not to mint any share at all. 

### External pre-conditions

1. The amount that we'll withdraw and deposit back on `harvest` directly depends on the treasury fees accrued on the underlying market. For this DoS to be persistent, the `market` specified in `StakedEXA` must have low activity so that the treasury fees accrued are a low amount. 

### Attack Path

1. Any user calls `deposit` or `mint` to stake EXA tokens. 
2. The `deposit` or `mint` function calls the internal `_update` function before minting the actual shares.
3. The `_update` function calls `harvest` to distribute the dividends from the provider market.
4. The `harvest` function [withdraws](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L356) from the market all funds from the treasury address (`provider`).
5. The `save` amount is calculated based on the `providerRatio` and that amount of tokens are [deposited back](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L358) into the market. 
    - When the `save` amount is low enough, the shares minted in the market will be 0, causing a revert on [this line](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC4626.sol#L48).

As long as the provider market is quite inactive and the treasury fees accrued are a low amount, the `harvest` function will revert every time causing a DoS on all deposits in `StakedEXA`. If the activity in the market stays low for quite some time, the DoS can be extended to more than 7 days, making this issue valid given that staking is a core protocol functionality. 

### Impact

All stakers will experience a DoS on new deposits within the `StakedEXA` contract. 

### PoC

_No response_

### Mitigation

To mitigate this issue is recommended to check within the `harvest` function if the amount to deposit back into the market will mint any share at all:

```diff
  function harvest() public whenNotPaused {
    // ...

    memMarket.withdraw(assets, address(this), memProvider);
    uint256 save = assets - amount;
+   uint256 sharesMinted = market.previewDeposit(save);
+   if (save != 0 && sharesMinted != 0) memMarket.deposit(save, savings);
-   if (save != 0) memMarket.deposit(save, savings);

    // ...
  }
```