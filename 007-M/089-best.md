Damaged Corduroy Sidewinder

Medium

# function `notifyRewardAmount` in stakedEXA.sol incorrectly checks reward token balance in contract when when staked asset is same as reward token.

## Summary
The function `notifyRewardAmount` ensures that the contract has sufficient reward tokens to cover the total rewards to be distributed over the specified duration. here;
```solidity
 if (
      rewardData.rate * rewardData.duration >
      reward.balanceOf(address(this)) - (address(reward) == asset() ? totalAssets() : 0)
    ) revert InsufficientBalance();
```
The logic contains an edge case where It accounts for the scenario where the reward token is the same as the staked asset, `(address(reward) == asset() ? totalAssets() : 0)` however the function` totalAssets() ` is defined in the contract as below;
```solidity
  function totalAssets() public view override returns (uint256) {
    return totalSupply();
  }
```
where totalsupply is the amount of minted shares for totalAsset staked and not the actual total amount of staked asset. this will result in flaw in checking the balance of staked assets, leading to incorrect validation during reward distribution.

## Vulnerability Detail
The current implementation checks if the reward rate multiplied by the duration exceeds the balance of the reward token held by the contract. However, it incorrectly uses totalAssets(), which represents the total supply of minted shares, instead of converting the total assets to the staked asset equivalent.

## Impact
Since totalsupply of minted share in not same as total assets staked. there contract has a wrong value for total assets this will ultimately cause incorrect validation preventing proper reward distribution. it may falsely indicate that there are insufficient reward tokens available, even when there are enough staked assets, causing the contract to revert and halt reward distribution.

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L222
## Tool used

Manual Review

## Recommendation
Modify the check to convert totalAssets() to the staked asset equivalent before performing the comparison:
```solidity
if (
  rewardData.rate * rewardData.duration >
  reward.balanceOf(address(this)) - (address(reward) == asset() ? convertToAssets(totalAssets()) : 0)
) revert InsufficientBalance();
```
This ensures the validation correctly accounts for the actual staked assets, preventing erroneous reverts and ensuring proper reward distribution.