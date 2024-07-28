Large Fuchsia Dove

Medium

# `StakedEXA::notifyRewardAmount()` will increase rewards without actually having tokens causing insolvency or lost rewards

### Summary

`StakedEXA::notifyRewardAmount()` reverts if the rewards to emit are smaller than the current balance of rewards (minus staked assets if `reward == asset`), but does not take into account that users may have not yet withdrawn their rewards.

### Root Cause

In `StakedEXA.sol::220`, in `notifyRewardAmount()`, it [checks](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L220-L223) the balance of rewards as:
```solidity
if (
  rewardData.rate * rewardData.duration >
  reward.balanceOf(address(this)) - (address(reward) == asset() ? totalAssets() : 0)
) revert InsufficientBalance();
```
Thus, it fails to account for rewards that have been emitted but were not yet claimed, still contributing to the balance. This means some users will not be able to withdraw their rewards or if the reward is the same as the asset, users will withdraw staked funds instead of rewards, which will DoS withdrawals.

### Internal pre-conditions

1. Admin calls `StakedEXA.sol::notifyRewardAmount()` without forwarding the tokens. This is a possibility because the admin assumes the code is well behaved and reverts if there are not enough tokens. Thus, it may simulate that it's possible to notify new rewards as the code check is incorrect, and call the function without sending tokens, halting withdrawals or reward claims.

### External pre-conditions

None.

### Attack Path

1. Admin calls `StakedEXA::notifyRewardAmount()` without forwarding tokens.
2. Users withdraw their stakes and/or rewards.
3. Last users will not be able to withdraw their stake or claim rewards because the others claimed more than they should.

### Impact

DoSed withdrawals or rewards claim.

### PoC

The following poc shows how it's possible to notify new rewards without forwarding tokens.
```solidity
function test_POC_IncorrectBalanceCheck_NotifyRewards() public {
  skip(duration);
  stEXA.notifyRewardAmount(exa, initialAmount);
}
```

### Mitigation

Track the total claimed rewards and subtract from the available reward's balance.