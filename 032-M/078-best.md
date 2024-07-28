Clumsy Coal Salamander

High

# Anyone could call `harvest()` to extend the period finish time

## Summary
Anyone could extend the finish time of `market.asset()`, potentially resulting in users receiving fewer rewards than expected within the same time period.

## Vulnerability Detail
The [StakedEXA.harvest()](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L344-L361) function can be called by anyone allowing it to be exploited to extend the reward finish time at little cost.
This could result in loss of rewards.
For example, if `market.asset()` is `WETH` and there are 10 WETH rewards within a 10-day period, a malicious user could extend the finish time on `day 5`, extending the finish time to the 15th day.
Participants would only receive around 7.5 WETH by the 10th day.

```solidity
File: contracts\StakedEXA.sol
344:   function harvest() public whenNotPaused { 
         [...]
360:     notifyRewardAmount(providerAsset, amount, address(this));
361:   }

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

## Impact
Anyonce could extend the finish time of `market.asset()` and the users may receive less rewards than expected during the same time period.
## Tool used

Manual Review

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L344-L361

## Recommendation
Only specific users are allowed to call function `harvest()`.