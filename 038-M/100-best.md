Wonderful Goldenrod Mole

High

# Users can avoid penalty near refTime

## Summary
Users can avoid penalties when they claim some time before refTime, depending on penaltyGrowth value.

## Vulnerability Detail

Below you can see the discountFactor function to calculate the penalty factor for early exits, after minTime and before refTime.

```solidity
function discountFactor(uint256 time) internal view returns (uint256) {
    uint256 memMinTime = minTime * 1e18;
    if (time <= memMinTime) return 0;
    uint256 memRefTime = refTime * 1e18;
    if (time >= memRefTime) {
      uint256 memExcessFactor = excessFactor;
      return (1e18 - memExcessFactor).mulWadDown((memRefTime * 1e18) / time) + memExcessFactor;
    }

    uint256 timeRatio = ((time - memMinTime) * 1e18) / (memRefTime - memMinTime);
    if (timeRatio == 0) return 0;

    uint256 penalties = uint256(((int256(penaltyGrowth) * int256(timeRatio).lnWad()) / 1e18).expWad());

    uint256 memPenaltyThreshold = penaltyThreshold;
    return Math.min((1e18 - memPenaltyThreshold).mulWadDown(penalties) + memPenaltyThreshold, 1e18);
  }

``` 
timeRatio is always between 0 and 1. If timeRatio is too close to 1 (exponantially) `penalties` will be 0. Even penaltyGrowth is in range as setPenaltyGrowth sets, this condition is met when penaltyGrowth is relatively small value. 
discountFactor will be 1e18 so full penalty discount will be applied even if the staker exits before refTime.

***A user can claim 2-3 seconds(1 block in Optimism) before refTime, without a penalty cut. I can provide PoC.***

Below you can see the penalty growth range. This value being small leads to discountFactor being 1e18(max).

```solidity

function setPenaltyGrowth(uint256 penaltyGrowth_) public onlyRole(DEFAULT_ADMIN_ROLE) {
    if (penaltyGrowth_ < 0.1e18 || penaltyGrowth_ > 100e18) revert InvalidRange();
    penaltyGrowth = penaltyGrowth_;
    emit PenaltyGrowthSet(penaltyGrowth_, msg.sender);
  }

```

After discountFactor is calculated, rawClaimable function calls it to calculate claimable amount as below.

```solidity
function rawClaimable(IERC20 reward, address account, uint256 shares) public view returns (uint256) {
    uint256 start = avgStart[account];
    if (start == 0) return 0;
    return earned(reward, account, shares).mulWadDown(discountFactor(block.timestamp * 1e18 - start));
  }
 
```

When full discount applied, penalties will be avoided.

## Impact

System logic bypassed, penalty avoided by users

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L234-L250


## Tool used

Manual Review

## Recommendation
