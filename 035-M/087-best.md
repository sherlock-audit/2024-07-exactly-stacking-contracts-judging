Electric Holographic Albatross

Medium

# Stakers will receive excessive rewards for durations beyond the reference staking period

### Summary

For users staking longer than the reference period, the contract **incorrectly calculates rewards** using the **current global index** instead of the index when the user's staking duration reaches the reference staking period. This leads to overpayment of rewards for the excess duration.

### Root Cause

The `earned` function uses the current `globalIndex(reward)` value regardless of staking duration

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L308

This approach doesn't align with the intended behavior described in the documentation:
_"In such a case, the idea is to pay in full for the maximum predetermined period and to apply a discount for any excess over this maximum."_

### Internal pre-conditions

1. User has staked for longer than `refTime`
2. `globalIndex(reward)` continues to increase after `refTime`

### External pre-conditions

1. Rewards continue to be added to the contract after the reference period

### Attack Path

1. User stakes tokens
2. User waits for a duration longer than `refTime`, and new rewards are added to the contract.
3. User withdraws, receiving excess rewards due to the increased global index.

### Impact

1. Overpayment of rewards to long-term stakers
2. Potential depletion of reward tokens faster than intended
3. Unfair advantage for users who stake for extended periods

### PoC

_No response_

### Mitigation

1. Implement a mechanism to track the global index at the reference time for each user:

```solidity
mapping(address => mapping(IERC20 => uint256)) public userRefTimeIndex;

function updateUserRefTimeIndex(address user, IERC20 reward) internal {
    if (block.timestamp >= avgStart[user] + refTime && userRefTimeIndex[user][reward] == 0) {
        userRefTimeIndex[user][reward] = globalIndex(reward);
    }
}

```
2. Modify the earned function to use the reference time index for calculations beyond `refTime`:

```solidity
function earned(IERC20 reward, address account, uint256 shares) public view returns (uint256) {
    uint256 stakingDuration = block.timestamp - avgStart[account];
    if (stakingDuration <= refTime) {
        return shares.mulWadDown(globalIndex(reward) - avgIndexes[account][reward]);
    } else {
        uint256 refTimeEarnings = shares.mulWadDown(userRefTimeIndex[account][reward] - avgIndexes[account][reward]);
        uint256 excessEarnings = shares.mulWadDown(globalIndex(reward) - userRefTimeIndex[account][reward]);
        return refTimeEarnings + excessEarnings.mulWadDown(excessFactor);
    }
}
```

3. Update the `userRefTimeIndex` in relevant functions (e.g., `_update`, `claim_`).

These changes ensure that rewards are calculated correctly for staking durations beyond the reference period, aligning with the intended behavior described in the documentation.