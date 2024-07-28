Large Fuchsia Dove

Medium

# Liquidating maturies with unassigned earnings will not take into account floating assets increase leading to loss of funds

### Summary

`Market::liquidate()` calculates the amount to liquidate in `Auditor::checkLiquidation()`, which does not take into account floating assets accrual due to unassigned earnings in `Market::noTransferRepayAtMaturity()`. Thus, it will underestimate the collateral a certain account has and liquidate less than it should, either giving these funds to the liquidatee or in case the debt is bigger than the collateral, leave this extra untracked collateral in the account which will not enable `clearBadDebt()` to work.

### Root Cause

In `Auditor::checkLiquidation()`, the collateral is computed just by [looking](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Auditor.sol#L219) at the shares of an account, without taking into account maturities accruing unassigned earnings.
In `Market::noTransferRepayAtMaturity()`, maturities are repaid and floating assets are [increased](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Market.sol#L481-L483) due to backup earnings.
Thus, when it seizes the collateral, it will seize at most the collateral without taking into account the new amount due to the floating assets sudden increase, as `Auditor::checkLiquidation()` is called [prior](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Market.sol#L560) the floating assets increase.

### Internal pre-conditions

1. Pool has unassigned earnings and user is liquidated.

### External pre-conditions

None.

### Attack Path

1. Users borrow at maturity and generate unassigned earnings.
2. One of these users is liquidated but the collateral preview is incorrect and the liquidation is incomplete and the funds are lost.

### Impact

If the user has less debt than collateral but the debt would require most of its collateral, it would not be possible to seize it all. If the user has more debt than collateral a portion of the collateral will be leftover which will not allow bad debt to be cleared right away via `clearBadDebt()` and stays accruing.

### PoC

Add the following test to `Market.t.sol`, confirming that the user should have all its collateral seized given that debt > collateral but some collateral remains due to the increase in floating assets.
```solidity
function test_POC_FloatingAssetsIncrease() external {
  uint256 maxVal = type(uint256).max;

  market.deposit(10000 ether, address(this));

  vm.startPrank(BOB);
  market.deposit(100 ether, BOB);
  market.borrowAtMaturity(4 weeks, 60e18, maxVal, BOB, BOB);
  vm.stopPrank();

  skip(10 weeks);

  (uint256 coll, uint256 debt) = market.accountSnapshot(BOB);
  assertEq(coll, 100e18);
  assertEq(debt, 111.246904109483404820e18);

  vm.startPrank(ALICE);
  market.liquidate(BOB, 100_000 ether, market);
  vm.stopPrank();

  (coll, debt) = market.accountSnapshot(BOB);
  assertEq(coll, 4.557168045571680e15);
  assertEq(debt, 20.337813200392495730e18);
}
```

### Mitigation

`Auditor::checkLiquidation()` should compute the increase in collateral by previewing the floating assets increase due to accrued unassigned earnings in a fixed pool or instead of increasing floating assets these earnings could increase the earnings accumulator.