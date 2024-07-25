Ancient Ash Lemur

High

# There will still be some funds untracked in the market after a borrower's liquidation


## Summary

In the `market`, all `funds` should be tracked accurately, whether they are currently held, `borrowed` by `borrowers`, or repaid in the future. To ensure this, the `market` has a sophisticated tracking system that functions effectively.
However, there will be some funds left untracked in the `market` when the `borrower` of the `maturity pool` is liquidated before `maturity`.

## Vulnerability Detail

Users have the option to deposit into the `market` directly or into specific `fixed rate pools`.
When `borrowers` `borrow` funds from the `fixed rate pool`, they are backed by the `fixed deposits` first.
If there is a shortfall in funds, the remaining `debt` is supported by `floating assets`.
The movement of funds between `fixed borrowers` and `fixed depositors` is straightforward outside of the `tracking system`.
The `tracking system` within the `market` primarily monitors funds within the `variable pool` itself.
To simplify the scenario, let's assume there are no `fixed depositors` involved.

First, there are `extraordinary earnings`, including `variable backup fees`, `late fixed repayment penalties`, etc.
The `earnings accumulator` is responsible for collecting these earnings from `extraordinary` sources and subsequently distributing them gradually and smoothly.
For this purpose, there is a `earningsAccumulator` variable. So when users deposit funds into the `variable pool`, the `floatingAssets` increase by the deposited amounts as well as any additional earnings from the `earnings accumulator` https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L723-L730

```solidity
  function afterDeposit(uint256 assets, uint256) internal override whenNotPaused whenNotFrozen {
    updateFloatingAssetsAverage();
    uint256 treasuryFee = updateFloatingDebt();
    uint256 earnings = accrueAccumulatedEarnings();
    floatingAssets += earnings + assets;
    depositToTreasury(treasuryFee);
    emitMarketUpdate();
  }
```

Funds borrowed by `variable rate borrowers` are tracked using the `floatingDebt` variable, while funds borrowed by `fixed rate borrowers` are tracked using the `floatingBackupBorrowed` variable.
Additionally, there is an `unassignedEarnings` variable for each `maturity pool`, which represents upcoming `fees` from `borrowers`.
These earnings are added to the `floatingAssets` whenever there are changes in the `market`, such as `borrowers` repaying their `debt` , depositors withdrawing their funds etc.

```solidity
function depositAtMaturity() external whenNotPaused whenNotFrozen returns (uint256 positionAssets) {
  uint256 backupEarnings = pool.accrueEarnings(maturity); // @audit, here
  floatingAssets += backupEarnings;
}
```

While this variable is important, it is not directly involved in the `tracking system`.

A step by step proof of this vulnerability can be seen [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-211757541)

To explain a bit further, in the `noTransferRepayAtMaturity` function, the `liquidator` repays the `full debt` of this `borrower` even before `maturity`.
It's important to note that the funds equivalent to the `unassignedEarnings` of this `maturity pool` are not added to the `tracking system` and they should be included in the `floatingAssets` as usual, but in this liquidation, we skipped this, see `noTransferRepayAtMaturity` here: https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L470-L543

So if there are other `borrowers` or `depositors` in this `maturity pool`, the `unassignedEarnings` would be added to the `floatingAssets` when their state changes and if this is the last user of this `maturity pool`, they won't be any chance to update this `pool`'s state.
Consequently, these funds remain untracked in the `market`, even though they are actually deposited in the `market`.

**After `liquidation`, the tracked balance and the actual balance are different and the difference is equal to the `unassignedEarnings` of that `maturity pool`.** see digits [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-211757541)

And if this user is the last `borrower` of that `maturity pool`, there is no way to convert these `unassignedEarnings` into `floatingAssets`. Consequently, some funds become untracked in the `market`. Or if this user is not the last user of this `maturity pool`, these untracked `unassignedEarnings` can be allocated to late `fixed depositors`.

End 2 end foundry test case is also available [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-211757541)

## Impact

As hinted [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-211757541), although the description of this vulnerability may seem complex, it can occur under normal circumstances and its impact is significant.
Therefore, it's crucial to prevent this issue.

## Code Snippet

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L470-L543

## Tool used

- Manual Review
- Previous finding, that was unfixed due to being a duplicate, see [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-211757541)

## Recommendation

As suggested [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/119#issuecomment-2117575412), apply these changes

```solidity
function noTransferRepayAtMaturity(
  uint256 maturity,
  uint256 positionAssets,
  uint256 maxAssets,
  address borrower,
  bool canDiscount
) internal returns (uint256 actualRepayAssets) {
..snip
    if (canDiscount) {
      (uint256 discountFee, uint256 backupFee) = pool.calculateDeposit(principalCovered, backupFeeRate);
      pool.unassignedEarnings -= discountFee + backupFee; // As we can see, the unassignedEarnings is not 0
      earningsAccumulator += backupFee;
      actualRepayAssets = debtCovered - discountFee;
    } else {
      actualRepayAssets = debtCovered;

+      if (principalCovered == pool.borrowed) {
+        earningsAccumulator += pool.unassignedEarnings;
+        pool.unassignedEarnings = 0;
+      }
    }
  } else {
    actualRepayAssets = debtCovered + debtCovered.mulWadDown((block.timestamp - maturity) * penaltyRate);

    earningsAccumulator += actualRepayAssets - debtCovered;
  }
}
```

The above suggestion is similar to another finding that was tagged `will fix`, but unrelated to liquidations, this was for bad debts clearance, see [issue#130](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/130#issuecomment-212834872)
