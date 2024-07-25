Ancient Ash Lemur

Medium

# `clearBadDebt` function still does not accrue earnings from each maturity

## Summary

During the operation of a maturity, `Pool.accrueEarnings()` is triggered to transfer the `backupEarnings` from `unassignedEarnings` of the maturity to the floating pool, which is dripped over time. However, the `clearBadDebt()` function does not trigger `Pool.accrueEarnings()` to collect earnings for the backup supply. This results in a loss of the remaining earnings if a maturity ends with a `clearBadDebt()` call.

## Vulnerability Detail

In the FixedLib library, the `accrueEarnings()` function is used to collect backup earnings from a specific maturity (fixed pool) to the floating pool. These earnings are dripped from the `unassignedEarnings` of the maturity over time. `Pool.accrueEarnings()` is called whenever an operation of the maturity (such as deposit, withdrawal, borrowing, or repayment) occurs see https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/utils/FixedLib.sol#L84-L99

```solidity
  function accrueEarnings(Pool storage pool, uint256 maturity) internal returns (uint256 backupEarnings) {
    uint256 lastAccrual = pool.lastAccrual;

    if (block.timestamp < maturity) {
      uint256 unassignedEarnings = pool.unassignedEarnings;
      pool.lastAccrual = block.timestamp;
      backupEarnings = unassignedEarnings.mulDivDown(block.timestamp - lastAccrual, maturity - lastAccrual);
      pool.unassignedEarnings = unassignedEarnings - backupEarnings;
    } else if (lastAccrual == maturity) {
      backupEarnings = 0;
    } else {
      pool.lastAccrual = maturity;
      backupEarnings = pool.unassignedEarnings;
      pool.unassignedEarnings = 0;
    }
  }
```

Now, `Market.clearBadDebt()` is a function [called by `Auditor.handleBadDebt()`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Auditor.sol#L315) to clear all the debt of a borrower when this borrower has no collateral. It clears all debt from each maturity that this account has borrowed from. However, it does not trigger `accrueEarnings()` for each fixed pool, see `Market.clearBadDebt()` https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L637-L660

```solidity
function clearBadDebt(address borrower) external {

    while (packedMaturities != 0) {
      if (packedMaturities & 1 != 0) {
        FixedLib.Position storage position = fixedBorrowPositions[maturity][borrower];
        uint256 badDebt = position.principal + position.fee;
        if (accumulator >= badDebt) {
          RewardsController memRewardsController = rewardsController;
          if (address(memRewardsController) != address(0)) memRewardsController.handleBorrow(borrower);
          accumulator -= badDebt;
          totalBadDebt += badDebt;
          floatingBackupBorrowed -= fixedPools[maturity].repay(position.principal);
          delete fixedBorrowPositions[maturity][borrower];
          account.fixedBorrows = account.fixedBorrows.clearMaturity(maturity);

          emit RepayAtMaturity(maturity, msg.sender, borrower, badDebt, badDebt);

          if (fixedPools[maturity].borrowed == position.principal) {
            earningsAccumulator += fixedPools[maturity].unassignedEarnings;
            fixedPools[maturity].unassignedEarnings = 0;
          }
        }
      }
      packedMaturities >>= 1;
      maturity += FixedLib.INTERVAL;
    }
```

An issue will occur when a maturity ends and `clearBadDebt()` is the last operation of this maturity, but the` unassignedEarnings()` of that maturity have not been fully accrued. This means that although the earnings have been completely dripped because the maturity has ended, they will never be collected into the floating pool because `clearBadDebt()` does not trigger `Pool.accrueEarnings()` for that maturity.
Therefore, in this case, floating assets will incur a loss of earnings from that maturity, since the remaining `unassignedEarnings` of this maturity will still be greater than 0 but will never be accrued. There is no mitigation to claim it since there is no debt remaining in this maturity.

See POC [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/162#issuecomment-2128348729)

> NB: From the previous contest it was duplicated to [this `will fix` tagged report](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/130#issuecomment-212834829), since the logic was similar around accruing the unassigned earnings, however these issues are distinct in the sense that as hinted in this report and the POC in the case where this is the last action of a maturity then the floating pool will lose funds, but after the fix applied to [this report](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/130#issuecomment-212834829) we now correctly have the funds tracked for the external accumulator as seen here: https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L652-L655, but since the `unassignedEarnings` in this maturity (i.e the one in which `clearBadDebt()` is queried last on) will never be accrued the floating pool will lose significant funds accrued from maturities.

## Impact

As hinted under _Vulnerability Details_ & [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/162#issuecomment-2128348729), when `clearBadDebt()` is the last operation of a maturity after it ends, the remaining `unassignedEarnings` in this maturity will never be accrued. Therefore, the floating pool will lose significant funds accrued from maturities

## Code Snippet

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L637-L660

## Tool used

Manual Review

## Recommendation

Apply the fix suggested [here](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/162#issuecomment-2128348729), i.e the `clearBadDebt()` function should trigger `Pool.accrueEarnings()` for each maturity when clearing debt as follows:

```solidity
..
while (packedMaturities != 0) {
    if (packedMaturities & 1 != 0) {
        FixedLib.Pool storage pool = fixedPools[maturity];
        floatingAssets += pool.accrueEarnings(maturity);

        ...
    }
    ...
}

```
