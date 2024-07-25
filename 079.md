Clumsy Coal Salamander

Medium

# In periods where `totalSupply = 0`, rewards will be locked forever in `StakeEXA` contract

## Summary
The `StakeEXA` contract is supposed to distribute rewards to stakers, but if in some period, `totalSupply()` was equal to 0, then for that time period, rewards will not added to `rewardData.index` and those period rewards would not distribute to any address and those rewards will stuck in contract forever.

## Vulnerability Detail
In the `notifyRewardAmount()` function, `rewardData.rate` is calculated by dividing `amount`(or `amount + remainingRewards`) by  `rewardData.duration`.
```solidity
File: contracts\StakedEXA.sol
209:   function notifyRewardAmount(IERC20 reward, uint256 amount, address notifier) internal onlyReward(reward) { 
         [...]
212:     if (block.timestamp >= rewardData.finishAt) {
213:       rewardData.rate = amount / rewardData.duration;
214:     } else {
215:       uint256 remainingRewards = (rewardData.finishAt - block.timestamp) * rewardData.rate;
216:       rewardData.rate = (amount + remainingRewards) / rewardData.duration;
217:     }
         [...]
229:   }
```
The value of `rewardData.rate` has been set to the division of available reward to duration. So if we distribute `rewardData.rate` amount in every second between stakers, then all rewards will be used by contract. Contract uses the `updateIndex()` function to update `rewardData.index` (this variable keeps track of distributed tokens) from L121.
```solidity
File: contracts\StakedEXA.sol
119:   function updateIndex(IERC20 reward) internal { 
120:     RewardData storage rewardData = rewards[reward];
121:     rewardData.index = globalIndex(reward);
122:     rewardData.updatedAt = uint40(lastTimeRewardApplicable(rewardData.finishAt));
123:   }
```
`rewardData.index` is updated by the `globalIndex()` function.
```solidity
File: contracts\StakedEXA.sol
283:   function globalIndex(IERC20 reward) public view returns (uint256) {
284:     RewardData storage rewardData = rewards[reward];
285:     if (totalSupply() == 0) return rewardData.index;
286: 
287:     return
288:       rewardData.index +
289:       (rewardData.rate * (lastTimeRewardApplicable(rewardData.finishAt) - rewardData.updatedAt)).divWadDown(
290:         totalSupply()
291:       );
292:   }
```
If for some period `totalSupply()` was 0, then contract won't increase `rewardData.index`. And in those periods, reward stuck in contract forever, because there is no mechanism to calculate them and withdraw them in contract.

For example, right after protocol team deploy and initialize the contract immediately before others having a chance of staking their tokens, a malicious attacker can call the `harvest()` function to push some rewards from a market, then the rewards for early period will be locked forever.

## Impact
Forever lost rewards. The amount depends on the amount of time `totalSupply` was `0`, which is not easy to predict, but is likely to happen in the beginning of every new staking period, as stakers have less incentive to be staking without pending rewards.

## Tool used

Manual Review

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L119-L123
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L283-L292
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L209-L229

## Recommendation
When `totalSupply == 0`, capture the rewards that were emitted and add them later to leftover.