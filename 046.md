Large Fuchsia Dove

Medium

# Anyone will DoS setting a new rewards duration which harms the protocol/users as they will receive too much or too little rewards

### Summary

Rewards are destributed via `StakedEXA::notifyRewardAmount()` over a certain duration and amount, which originates a rate. The protocol may intend to change this rate by altering the duration, making users receive more or less rewards depending on intent. The duration of the rewards is set in `StakedEXA::setRewardsDuration()`, but it will be near impossible to set as there is an automatic mechanism to harvest rewards from the provider.

### Root Cause

In `StakedEXA::443`, it's [not possible](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L443) to set a new duration if the current rewards are still not finished. This will always be the case as the automated provider harvesting will keep notifying new rewards and resetting the finish time of the rewards.

### Internal pre-conditions

None.

### External pre-conditions

None.

### Attack Path

1. Provider harvests new rewards.
2. `StakedEXA::harvest()` is called, which notifies new rewards and updates the finish time.
3. `StakedEXA::setRewardsDuration()` reverts as `block.timestamp` has not reached `rewardData.finishAt`.

### Impact

The ability to change the duration of rewards is DoSed which will impact the amount of rewards users get over time.

### PoC

The function `StakedEXA::setRewardsDistribution()` shows that it's not possible to update the duration if the current reward period is not finished:
```solidity
function setRewardsDuration(IERC20 reward, uint40 duration) public onlyRole(DEFAULT_ADMIN_ROLE) {
  RewardData storage rewardData = rewards[reward];
  if (rewardData.finishAt > block.timestamp) revert NotFinished(); //@audit DoS this

  rewardData.duration = duration;

  emit RewardsDurationSet(reward, msg.sender, duration);
}
```
`StakedEXA::harvest()` notifies new rewards, which updates the current reward period finish, making it impossible to change the duration.

### Mitigation

The duration can be set by carefully adjusting the current reward rate to reflect the new duration. One example solution is doing:
```solidity
function setRewardsDuration(uint256 _rewardsDuration) external onlyRole(DEFAULT_ADMIN_ROLE) {
    uint256 periodFinish_ = periodFinish;
    if (block.timestamp < periodFinish_) {
        uint256 leftover = (periodFinish_ - block.timestamp) * rewardRate;
        rewardRate = leftover / _rewardsDuration;
        periodFinish = block.timestamp + _rewardsDuration;
    }

    rewardsDuration = _rewardsDuration;
    emit RewardsDurationUpdated(rewardsDuration);
}
```