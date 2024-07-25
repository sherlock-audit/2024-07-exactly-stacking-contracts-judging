Fit Boysenberry Stallion

Medium

# Admins can cause a DoS by enabling too many reward tokens

### Summary

Too many reward tokens within the same `StakedEXA` contract will cause a DoS on deposits and withdrawals for all stakers, which will lock all staked funds inside the contract. 

### Root Cause

In `StakedEXA:418` there is a missing check to ensure that not too many reward tokens are enabled within the same `StakedEXA` contract.  

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L418

### Internal pre-conditions

1. Admin needs to call `enableReward` many times to distribute a large variety of reward tokens.

### External pre-conditions

None

### Attack Path

1. Admin calls `enableReward` many times to distribute a variety of reward tokens to all stakers.
2. When stakers try to deposit or withdraw from the contract, the `_update` function will be called, which will loop over all the reward tokens to update the indexes and more. 
    - If the `rewardsTokens` array has too many items, all transactions will be reverted due to an OOG error, meaning the total consumed gas will be higher than the block gas limit. 

### Impact

All stakers will experience a DoS on all deposits and withdrawals within the `StakedEXA` contract. This means that all staked funds will be locked in the contract forever because it won't be possible to withdraw funds. 

Moreover, the admins don't have any way of reducing the `rewardsTokens` array so they won't be able to unstuck all the staked funds. 

According to the README, there are no admin restrictions regarding the length of arrays so I guess that makes this issue valid. 

### PoC

_No response_

### Mitigation

To mitigate this issue is recommended to check the array length on `enableReward` before adding a new reward token to prevent adding too many items and causing a DoS. 

```diff
  function enableReward(IERC20 reward) public onlyRole(DEFAULT_ADMIN_ROLE) {
    if (rewards[reward].finishAt != 0) revert AlreadyEnabled();
+   if (rewardsTokens.length >= maxRewardTokens) revert();

    rewards[reward].finishAt = uint40(block.timestamp);
    rewardsTokens.push(reward);

    emit RewardListed(reward, msg.sender);
  }
```

Also, it'd be a good idea to add a new function to deprecate a reward token so that the admins can remove some reward tokens to add new ones. 