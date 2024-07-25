Ancient Ash Lemur

Medium

# Unassigned earnings from a fixed pool can still be stolen


## Summary

This is very similar to [this report](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/157#issuecomment-21186888) & [this](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/68#issuecomment-21186887), but here we have the fee as `1` instead of `0`.

As the linked report has proven, an attacker can borrow a dust amount from a fixed pool to round down the fee to zero, repeating this thousands of times the attacker will get a big fixed loan with 0 fees. If that loan is repaid early, the amount repaid will be lower than the borrowed amount, effectively stealing funds from the unassigned earnings of that fixed pool.

To fix this the protocol has swapped one of the `mulDivDown` with a `mulWadUp` , i.e

```diff
- fee = assets.mulWadDown(fixedRate.mulDivDown(maturity - block.timestamp, 365 days));
+ fee = assets.mulWadUp(fixedRate.mulDivDown(maturity - block.timestamp, 365 days));

```

Issue however is that since there is no "minimum fee" amount, an attacker can still make multiple dust borrows and have the fee as low as possible (could even be `1` wei), which wouldn't sum up to much after repayments.

## Vulnerability Detail

When a user takes a loan from a fixed pool, the resulting fee is rounded up, however there is no minimum value for this fee so an attacker can use this feature to borrow a dust amount from a fixed pool thousands of times to end up with a big fixed loan with minute fees.

The fee is calculated here https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L322-L323

```solidity
      fee = assets.mulWadUp(fixedRate.mulDivDown(maturity - block.timestamp, 365 days));

```

When the borrowed assets are really low, the resulting fee will potentially be `1`. This by itself is already an issue, but an attacker can use this loan to steal funds from the unassigned earnings.

When a fixed loan is repaid early, it can get a discount that is calculated as if the repaid amount was deposited into that fixed pool. The discount depends on the unassigned earnings of the pool and the proportion that the repaid amount represents in the total fixed debt backed by the floating pool.

When the attacker repays this loan with no interest, he's going to get a discount based on the current unassigned earnings of that pool. This discount will make the attacker repay less funds than he originally borrowed, and those funds will be subtracted from the unassigned earnings of that pool.

## Impact

An attacker can still steal the unassigned earnings from a fixed pool even after the round up, cause the fee that's paid could be as low as `1`.

## Code Snippet

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/Market.sol#L322-L323

## Tool used

Manual Review

## Recommendation

Apply the fix suggested in [this report](https://github.com/sherlock-audit/2024-04-interest-rate-model-judging/issues/157#issuecomment-21186888)

Which is to have a minimum fee/borrow amount, i.e

```diff
    fee = assets.mulWadDown(fixedRate.mulDivDown(maturity - block.timestamp, 365 days));
+   require(fee > minimumFee);
```
