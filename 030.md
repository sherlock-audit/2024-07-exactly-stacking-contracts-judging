Deep Concrete Goose

Medium

# Redeeming Shares To A Receiver Address Is Implemented Incorrectly

## Summary

Underlying assets are transferred incorrectly when redeeming shares to a `receiver` who is not the original shareholder.

## Vulnerability Detail

As a [4626](https://ethereum.org/en/developers/docs/standards/tokens/erc-4626/)-compliant implementation, the underlying value of redeemed [`StakedEXA`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol) may be sent to a custom `receiver` address:

```solidity
function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
```

However, the `receiver` address is not respected.

In the following proof of concept, we'll compare the results of an optimal staking interaction which conditionally determines whether account `c0ffee` decides to redeem either to themselves or to another account, `babe`:

### StakedEXA.t.sol

```solidity
/// @notice The `receiver` address is not respected when redeeming shares.
/// @param redeemToBabe If `true`, stakeholder `c0ffee` will opt to redeem shares to `babe`, else themselves.
function testSherlockRewardsSentToIncorrectRecipient(
  bool redeemToBabe
) external {

  /// @notice Here we'll configure two equally-valued staking
  /// @notice positions; one for `c0ffee` and one for `deadbeef`.
  /// @notice Although the activity of `c0ffee` is of interest,
  /// @notice we will be using `deadbeef` as a control.
  /// @notice Both positions are equal in value.
  uint256 depositAmount = 1 ether;
  address c0ffee = address(0xc0ffee);
  address deadbeef = address(0xdeadbeef);

  exa.mint(c0ffee, depositAmount);
  exa.mint(deadbeef, depositAmount);
  vm.startPrank(deadbeef);
    exa.approve(address(stEXA), type(uint256).max);
    stEXA.deposit(depositAmount, deadbeef);
  vm.stopPrank();
  vm.startPrank(c0ffee);
    exa.approve(address(stEXA), type(uint256).max);
    stEXA.deposit(depositAmount, c0ffee);
  vm.stopPrank();

  /// @notice Let's imagine that during staking, rewards denominated
  /// @notice in the underlying token ($EXA) are accrued:
  uint256 reward = 1 ether;
  skip(duration / 2);
  exa.mint(address(stEXA), reward);
  stEXA.notifyRewardAmount(exa, reward);

  /// @notice Fast forward to the end of an optimal stake for both
  /// @notice `c0ffee` and `deadbeef`.
  skip(duration / 2);

  /// @notice Now the optimal stake duration has elapsed, let's
  /// @notice have each of the stakeholders redeem their positions,
  /// @notice starting with `deadbeef`, the control account:
  vm.startPrank(deadbeef);
    stEXA.redeem(stEXA.balanceOf(deadbeef), deadbeef, deadbeef);
  vm.stopPrank();

  /// @notice Here we declare the `babe` account. For clarity,
  /// @notice we demonstrate that this account is in possession
  /// @notice of no assets prior to `c0ffee`'s conditional
  /// @notice actions.
  address babe = address(0xbabe);
  assertEq(exa.balanceOf(babe), 0);
  assertEq(stEXA.balanceOf(babe), 0);
  assertEq(rA.balanceOf(babe), 0);

  /// @notice Based upon the value of `redeemToBabe`, `c0ffee` can choose
  /// @notice whether to redeem their stake to themselves or to `babe`:
  vm.startPrank(c0ffee);
    stEXA.redeem(stEXA.balanceOf(c0ffee), redeemToBabe ? babe : c0ffee, c0ffee);
  vm.stopPrank();

  /// @notice Results of unstaking for each accounts in $EXA:
  assertEq(exa.balanceOf(c0ffee),   redeemToBabe ? 375249999999992544000 : 376249999999992544000); /// @audit c0ffee_receives_majority_stake
  assertEq(exa.balanceOf(babe),     redeemToBabe ?   1000000000000000000 :                     0); /// @audit babe_receives_native_reward
  assertEq(exa.balanceOf(deadbeef), 376249999999992544000 /* control */);

  /// @notice Results of unstaking for each accounts in $rA:
  assertEq(rA.balanceOf(c0ffee),    499999999999994726400 /* unconditional */); /// @audit c0ffee_always_receives_rA_rewards
  assertEq(rA.balanceOf(babe),      0); /// @audit babe_never_receives_rA_rewards
  assertEq(rA.balanceOf(deadbeef),  499999999999994726400 /* control */);

}
```

Okay, now let's analyze the outcomes. First, when `c0ffee` redeems to themselves:

| Account  | $EXA Balance            | $rA Balance             |
|----------|-------------------------|-------------------------|
| `c0ffee` | `376249999999992544000` | `499999999999994726400` |
| `babe`   | `0`                     | `0`                     |

Contrast against when `c0ffee` redeems to `babe`:

| Account  | $EXA Balance            | $rA Balance             |
|----------|-------------------------|-------------------------|
| `c0ffee` | `375249999999992544000` | `499999999999994726400` |
| `babe`   | `1000000000000000000`   | `0`                     |

Even though `c0ffee` explicitly redeemed the entirety of their shares to `babe`, `c0ffee` continues to hold the supermajority of underlying value (`babe` only received the initial deposit amount).

This error fundamentally arises from the assumptions made whilst burning shares during asset redemption:

```solidity
 else if (to == address(0)) {
    for (uint256 i = 0; i < rewardsTokens.length; ++i) {
      IERC20 reward = rewardsTokens[i];
      updateIndex(reward);
@>    claimWithdraw(reward, from, amount);
    }
  } else revert Untransferable(); 
```

Notice here, that the [`claimWithdraw(IERC20,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L182C11-L182C73) function has no concept of withdrawing to a `recipient` (such as when redeeming to a `receiver`):

```solidity
/// @notice Claims unclaimed rewards when withdrawing an amount of assets.
/// @param reward The reward token to claim.
@> /// @param account The account receiving the withdrawn assets and rewards. /// @audit account_is_implicitly_source_of_redemption_and_transfer_destination
/// @param amount The amount of assets being withdrawn.
@> function claimWithdraw(IERC20 reward, address account, uint256 amount) internal { /// @audit account_is_always_transfer_recipient_even_when_redeeming_to_different_address
  uint256 balance = balanceOf(account);
  uint256 numerator = claimed[account][reward] * amount;
  uint256 claimedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
  claimed[account][reward] -= claimedAmount;

  numerator = saved[account][reward] * amount;
  uint256 savedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
  saved[account][reward] -= savedAmount;

  uint256 claimableAmount = Math.max(rawClaimable(reward, account, amount), claimedAmount); // due to excess exposure
  uint256 claimAmount = claimableAmount - claimedAmount;
  if (claimAmount != 0) {
    reward.transfer(account, claimAmount);
    emit RewardPaid(reward, account, claimAmount);
  }

  uint256 rawEarned = earned(reward, account, amount);
  // due to rounding
  uint256 saveAmount = rawEarned <= claimableAmount + savedAmount ? 0 : rawEarned - claimableAmount - savedAmount;
  if (saveAmount != 0) reward.transfer(savings, saveAmount);
}
``` 

## Impact

Underlying assets for redeemed shares are not correctly transferred to the intended recipient.

When burning shares to a specified recipient address, the underlying value of these shares must be redeemed inclusive of accrued rewards, since shares fundamentally represent a claim to the representative totality of share value and not just the initially deposited amount of underlying token.

In all, this fundamentally violates the expectations of 4626 share redemption and should be treated as medium severity.

## Code Snippet

```solidity
/// @notice Hook to handle updates during token transfer.
/// @param from The address transferring the tokens.
/// @param to The address receiving the tokens.
/// @param amount The amount of tokens being transferred.
function _update(address from, address to, uint256 amount) internal override whenNotPaused {
  if (amount == 0) revert ZeroAmount();
  if (from == address(0)) {
    uint256 start = avgStart[to];
    uint256 time = start != 0 ? block.timestamp * 1e18 - start : 0;
    uint256 memRefTime = refTime * 1e18;
    uint256 balance = balanceOf(to);
    uint256 total = amount + balance;

    for (uint256 i = 0; i < rewardsTokens.length; ++i) {
      IERC20 reward = rewardsTokens[i];
      updateIndex(reward);

      if (time > memRefTime) {
        if (balance != 0) claimWithdraw(reward, to, balance);
        avgIndexes[to][reward] = rewards[reward].index;
      } else {
        if (balance != 0) claim_(reward);
        uint256 numerator = avgIndexes[to][reward] * balance + rewards[reward].index * amount;
        avgIndexes[to][reward] = numerator == 0 ? 0 : (numerator - 1) / total + 1;
      }
    }

    if (time > memRefTime) avgStart[to] = block.timestamp * 1e18;
    else {
      uint256 numerator = start * balance + block.timestamp * 1e18 * amount;
      avgStart[to] = numerator == 0 ? 0 : (numerator - 1) / total + 1;
    }
    harvest();
  } else if (to == address(0)) {
    for (uint256 i = 0; i < rewardsTokens.length; ++i) {
      IERC20 reward = rewardsTokens[i];
      updateIndex(reward);
      claimWithdraw(reward, from, amount);
    }
  } else revert Untransferable();

  super._update(from, to, amount);
}
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L125C3-L166C4

## Tool used

Manual Review

## Recommendation

It is our opinion that using the [`_update(address,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L129C12-L129C61) to handle rewards side effects is too low-level to have an appropriate awareness of the contextually-correct behaviour for the outcome of a burn operation.

If we were to expand the implementation of [`claimWithdraw(IERC20,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L182C11-L182C73) to support the required `receiver` address (i.e. `claimWithdraw(IERC20,address,address,uint256)`), the starved input parameter to the invocation inside of [`_update(address,address,uint256)`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L129C12-L129C61) should emphasize this architectural concern during burning.

```diff
/// @notice Claims unclaimed rewards when withdrawing an amount of assets.
/// @param reward The reward token to claim.
/// @param account The account receiving the withdrawn assets and rewards.
/// @param amount The amount of assets being withdrawn.
- function claimWithdraw(IERC20 reward, address account, uint256 amount) internal {
+ function claimWithdraw(IERC20 reward, address account, address receiver, uint256 amount) internal {
  uint256 balance = balanceOf(account);
  uint256 numerator = claimed[account][reward] * amount;
  uint256 claimedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
  claimed[account][reward] -= claimedAmount;

  numerator = saved[account][reward] * amount;
  uint256 savedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
  saved[account][reward] -= savedAmount;

  uint256 claimableAmount = Math.max(rawClaimable(reward, account, amount), claimedAmount); // due to excess exposure
  uint256 claimAmount = claimableAmount - claimedAmount;
  if (claimAmount != 0) {
-    reward.transfer(account, claimAmount);
+    reward.transfer(receiver, claimAmount);
    emit RewardPaid(reward, account, claimAmount);
  }

  uint256 rawEarned = earned(reward, account, amount);
  // due to rounding
  uint256 saveAmount = rawEarned <= claimableAmount + savedAmount ? 0 : rawEarned - claimableAmount - savedAmount;
  if (saveAmount != 0) reward.transfer(savings, saveAmount);
}
```
