Teeny Marmalade Elk

High

# Unclaimed reward may be used in calculating reward ratios.

## Summary
Due to incorrect validation in the `StakedEXA.sol#notifyRewardAmount()` function, an unclaimed reward may be used as the reward amount.
## Vulnerability Detail
The `StakedEXA.sol#notifyRewardAmount()` is a function that notifies the contract about a reward amount.

```solidity
  function notifyRewardAmount(IERC20 reward, uint256 amount, address notifier) internal onlyReward(reward) {
    updateIndex(reward);
    RewardData storage rewardData = rewards[reward];
    if (block.timestamp >= rewardData.finishAt) {
      rewardData.rate = amount / rewardData.duration;
    } else {
      uint256 remainingRewards = (rewardData.finishAt - block.timestamp) * rewardData.rate;
216:  rewardData.rate = (amount + remainingRewards) / rewardData.duration;
    }

    if (rewardData.rate == 0) revert ZeroRate();
220:if (
221:  rewardData.rate * rewardData.duration >
222:  reward.balanceOf(address(this)) - (address(reward) == asset() ? totalAssets() : 0)
223:) revert InsufficientBalance();

    rewardData.finishAt = uint40(block.timestamp) + rewardData.duration;
    rewardData.updatedAt = uint40(block.timestamp);

    emit RewardAmountNotified(reward, notifier, amount);
  }
```
In #L220~#L223, a check is performed to confirm whether the asset corresponding to the reward amount has been transferred to the contract.

In a contract, if the user does not call the `claim_()/claimWithdraw()` function, the reward will remain in the contract.
However, as seen in #L222, reward that has not yet been claimed and is locked in the contract is not taken into account.
As a result, the attacker can unfairly increase the reward rate without transferring assets corresponding to the reward amount to the contract in #L216.

On the other hand, the `notifyRewardAmount()` function can be called by `DEFAULT_ADMIN_ROLE`.
```solidity
  function notifyRewardAmount(IERC20 reward, uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) {
    updateIndex(reward);
    notifyRewardAmount(reward, amount, msg.sender);
  }
```
Considering that the phenomenon of private key explorit occurring frequently, if the private key of `DEFAULT_ADMIN_ROLE` is explorited, the attacker can freeze all assets of the protocol by repeatedly executing this attack.
## Impact
All assets of the protocol may be freezed permanently.
## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L209-L229
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L453-L456
## Tool used

Manual Review

## Recommendation
Accurately implement validation of reward amount considering unclaimed reward.