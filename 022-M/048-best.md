Large Fuchsia Dove

Medium

# Having no deposits in `StakedEXA` will lead to stuck rewards when harvesting

### Summary

The provider in `StakedEXA` is free to distributed rewards as it pleases and there is an automated way to distribute via `StakedEXA::harvest()`. The harvest setup may be started but no actual deposits have been made to `StakedEXA` or users may withdraw from `StakedEXA` while automated ongoing rewards are happening, which will lead to stuck rewards inside `StakedEXA`.

### Root Cause

In `StakedEXA::globalIndex()` it returns a stale index if the `totalSupply()` is [null](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L285) (no deposits), but it should instead calculate the amount (`rate x deltaTime`) and send it to savings. This way rewards will never be stuck.

### Internal pre-conditions

1. No deposits are present in `StakedEXA`.

### External pre-conditions

None.

### Attack Path

1. Deposits were not yet made and rewards are harvested or users withdraw all deposits while rewards are rolling out.

### Impact

Stuck rewards as the index is not increased but `rewardData.updatedAt` increases. 

### PoC

In `StakedEXA::updateIndex()`, the index is updated to `globalIndex(reward);` and `rewardData.updatedAt` to the current `block.timestamp` or `rewardData.finishAt`.
```solidity
function updateIndex(IERC20 reward) internal {
  RewardData storage rewardData = rewards[reward];
  rewardData.index = globalIndex(reward);
  rewardData.updatedAt = uint40(lastTimeRewardApplicable(rewardData.finishAt));
}
```
In `StakedEXA::globalIndex()`, the index is not increased if the total supply is null:
```solidity
function globalIndex(IERC20 reward) public view returns (uint256) {
  RewardData storage rewardData = rewards[reward];
  if (totalSupply() == 0) return rewardData.index;

  return
    rewardData.index +
    (rewardData.rate * (lastTimeRewardApplicable(rewardData.finishAt) - rewardData.updatedAt)).divWadDown(
      totalSupply()
    );
}
```
Thus, as can be seen, the index will not be updated by `rewardData.updatedAt` is increased, losing these rewards forever.

### Mitigation

If the total supply is null the amount not distributed can be calculated by doing `rate x deltaTime` and send to savings.