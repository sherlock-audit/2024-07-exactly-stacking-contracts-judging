Electric Holographic Albatross

Medium

# Incorrect Discount Application will Reduce Rewards for Long-Term Stakers

### Summary

For users that have a staking duration longer than the reference staking period, the current implementation of the StakedEXA contract **incorrectly applies the discount factor to the entire reward amount**, including rewards earned during the reference staking period. This **contradicts the intended design** as described in the documentation, which states that the _discount should only be applied to rewards earned after the maximum predetermined period (reference staking period)._

### Root Cause

The discrepancy lies in the `rawClaimable()` function, which calculates the claimable rewards for a user. The function applies the discount factor to the entire earned amount, instead of only to the portion earned after the reference period.
Current implementation in `rawClaimable()`:
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L319

### Internal pre-conditions

1. The contract has been initialized with a valid `refTime` (reference staking period).
2. Users have staked tokens for a duration longer than `refTime`.

### External pre-conditions

_No response_

### Attack Path

This is not an attack vector, but rather a discrepancy between the intended behavior and the actual implementation.

### Impact

Long-term stakers (those staking for longer than the reference period) will receive fewer rewards than intended. The impact increases with the duration of staking beyond the reference period, potentially discouraging long-term staking and reducing the protocol's ability to retain stakers.

### PoC

_No response_

### Mitigation

Modify the `rawClaimable()` function to apply the discount factor only to rewards earned after the reference period. Here's a suggested approach:

1. Calculate rewards earned up to the reference period.
2. Calculate rewards earned after the reference period.
3. Apply the discount factor only to the rewards earned after the reference period.
4. Sum the two amounts to get the total claimable rewards.

```solidity
function rawClaimable(IERC20 reward, address account, uint256 shares) public view returns (uint256) {
    uint256 start = avgStart[account];
    if (start == 0) return 0;
    
    uint256 currentTime = block.timestamp * 1e18;
    uint256 stakingDuration = currentTime - start;
    uint256 refTimestamp = start + refTime * 1e18;
    
    if (stakingDuration <= refTime * 1e18) {
        // If staking duration is less than or equal to refTime, use original implementation
        return earned(reward, account, shares).mulWadDown(discountFactor(stakingDuration));
    } else {
        // Calculate total earned rewards
        uint256 totalEarned = earned(reward, account, shares);
        
        // Calculate the fraction of rewards earned after refTime
        uint256 excessFraction = (currentTime - refTimestamp).divWadDown(stakingDuration);
        
        // Apply discount only to the excess amount
        uint256 discountedExcess = totalEarned.mulWadDown(excessFraction).mulWadDown(discountFactor(stakingDuration));
        uint256 undiscountedPortion = totalEarned.mulWadDown(1e18 - excessFraction);
        
        return undiscountedPortion + discountedExcess;
    }
}
```