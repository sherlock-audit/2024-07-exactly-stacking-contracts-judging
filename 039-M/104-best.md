Wonderful Goldenrod Mole

Medium

# Untracked reward penalty cuts in StakedEXA.sol

## Summary

Exactly protocol has developed a new system in their synthetix staking rewards contract as penalty logic.
However, the penalty cuts are untracked in the contract.

## Vulnerability Detail

According to the contract, if a user exits between minTime(earliest exit with reward) and refTime (optimal), user will be given a penalty from his reward. This penalty cut decreases when time get near to refTime, and when arrived refTime it's 0 penalty.

```solidity
function rawClaimable(IERC20 reward, address account, uint256 shares) public view returns (uint256) {
    uint256 start = avgStart[account];
    if (start == 0) return 0;
    return earned(reward, account, shares).mulWadDown(discountFactor(block.timestamp * 1e18 - start)); //@audit penalty cut from earned() not tracked
  }

```

## Impact

Untracked funds in the contract, can affect reward calculations and harvest flow.

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L234-L250
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L316-L320

## Tool used

Manual Review

## Recommendation

Add functions to track the reward cut by penalty, use them paying other user's rewards.