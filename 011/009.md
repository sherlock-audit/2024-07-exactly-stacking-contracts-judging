Curly Vanilla Barracuda

Medium

# Insecure Token Transfer Methods

## Summary
Ser Marry
## Vulnerability Detail
The contract contains several functions (`claimWithdraw`, `claim_`, and `finishDistribution`) that interact with ERC20 tokens. These functions use the `.transfer` method for sending tokens, However, the `.transfer` method does not throw an exception or revert the transaction if the transfer fails. Instead, it simply returns a boolean indicating success or failure. If the recipient of the transfer is a contract that does not handle ERC20 token transfers correctly (i.e., it does not have a fallback function that can accept ERC20 tokens), the `.transfer` call will succeed from the perspective of the calling contract, but the tokens will not be received by the intended recipient. This leads to a silent failure, where the contract believes the transfer was successful, but in reality, the tokens were not moved.
Example Code Snippet:
```solidity
function claimWithdraw(IERC20 reward, address account, uint256 amount) internal {
    uint256 balance = balanceOf(account); // @audit whenNotPaused
    uint256 numerator = claimed[account][reward] * amount;
    uint256 claimedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
    claimed[account][reward] -= claimedAmount;

    numerator = saved[account][reward] * amount;
    uint256 savedAmount = numerator == 0 ? 0 : (numerator - 1) / balance + 1;
    saved[account][reward] -= savedAmount;

    uint256 claimableAmount = Math.max(rawClaimable(reward, account, amount), claimedAmount); // due to excess exposure
    uint256 claimAmount = claimableAmount - claimedAmount;
    if (claimAmount != 0) {
      reward.transfer(account, claimAmount); << @audit-ok
      emit RewardPaid(reward, account, claimAmount);
    }

    uint256 rawEarned = earned(reward, account, amount);
    // due to rounding
    uint256 saveAmount = rawEarned <= claimableAmount + savedAmount ? 0 : rawEarned - claimableAmount - savedAmount;
    if (saveAmount != 0) reward.transfer(savings, saveAmount); << @audit-ok
  }
```
Also in `claim_` function:
```solidity
      if (saveAmount != 0) {
        saved[msg.sender][reward] = savedAmount + saveAmount;
        reward.transfer(savings, saveAmount); << @audit-ok
      }
    }
    if (claimAmount != 0) {
      reward.transfer(msg.sender, claimAmount); << @audit-ok
      emit RewardPaid(reward, msg.sender, claimAmount);
    }
```
And in the `finishDistribution` function:
```solidity
 function finishDistribution(IERC20 reward) public onlyRole(DEFAULT_ADMIN_ROLE) onlyReward(reward) {
    updateIndex(reward);

    if (block.timestamp < rewards[reward].finishAt) {
      uint256 finishAt = rewards[reward].finishAt;
      rewards[reward].finishAt = uint40(block.timestamp);
      reward.transfer(savings, (finishAt - block.timestamp) * rewards[reward].rate); << @audit-ok
    }

    emit DistributionFinished(reward, msg.sender);
  }
```
In the above snippet, the `claimWithdraw` function uses `reward.transfer(account, claimAmount)` to send tokens to an account. If account is a contract that does not handle ERC20 tokens correctly, this transfer will fail silently.
## Impact
The primary impact of this vulnerability is the potential loss of tokens. Users expecting to receive their rewards may find that their balances remain unchanged, leading to confusion and frustration. Additionally, this vulnerability could be exploited by attackers to drain tokens from the contract under certain conditions, such as if the contract is tricked into sending tokens to a malicious contract controlled by an attacker.
## Code Snippet
[StakedEXA.sol#L202](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L202)
[StakedEXA.sol#L195](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L195)
[StakedEXA.sol#L390](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L390)
[StakedEXA.sol#L394](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L394)
[StakedEXA.sol#L431](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L431)
## Tool used

Manual Review

## Recommendation
The contract should use the `SafeERC20` library's `safeTransfer` method instead of the native `.transfer` method. The `SafeERC20` library checks if the recipient is a contract and, if so, calls its fallback function instead of directly sending tokens. This ensures that the contract behaves correctly even when interacting with other contracts.