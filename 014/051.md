Rare Shamrock Sidewinder

High

# Due to missing update of a user's average starting time for staked tokens, user can reclaim already claimed rewards within the same distribution period.

### Summary

The missing update of a user's **avgStart[user]** within the **else if** block of `StakedEXA.sol::_update()` function after the for loop on L162 will lead a malicious user to doubly claim all reward tokens  over again within the same distribution cycle. #See code snippet below:
```solidity
} else if (to == address(0)) {
      for (uint256 i = 0; i < rewardsTokens.length; ++i) {
        IERC20 reward = rewardsTokens[i];
        updateIndex(reward);
        claimWithdraw(reward, from, amount);
      }
```

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L157-L162

### Root Cause

Inside the `_update()` function of the stakedEXA.sol contract are 3 conditional arms that handles the various token hook transfers. The first conditional arm handles minting transfers, whereas the 2nd conditional arm handles the burning transfers. The code revert on the 3rd arm effectively implying non-handling of token transfers between accounts. 


2. If we look at the first arm that handles the minting transfers when the reference staked period has elapsed, after the call to `claimWithdraw()` inside the **for** loop, it goes on to update the average starting time for a user's staked token right after the execution of the **for** loop . See below:

```solidity
if (time > memRefTime) avgStart[to] = block.timestamp * 1e18;
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L151

 + Still in the first arm where minting token transfers are handled when the reference staked duration is not yet reached, after the call to `claim()` inside the for loop, it also goes on to update the average starting time for a user's staked token right after the execution of this for loop. # See below:

```solidity
else {
        uint256 numerator = start * balance + block.timestamp * 1e18 * amount;
        avgStart[to] = numerator == 0 ? 0 : (numerator - 1) / total + 1;
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L152-L154


3. The said vulnerability lies in the 2nd conditional arm that handles the token burning transfers. Let's analyse this arm:
 +  The code iterates through all the available reward tokens, and for each reward token, it calls `updateIndex(reward)` and then `claimWithdraw(reward)`  function afterwards. # See below:

```solidity
        updateIndex(reward);
        claimWithdraw(reward, from, amount);
```

 + However, unlike in the first conditional arm where minting token transfer logic is handled, this 2nd conditional arm where burning logic is handled fails to update the user's average starting time for staked token. **That is, there is no resetting of avgStart[from] to any value.**  What this means is that, a user's avgStart[user] is still in the past, and hence he or she can reclaim the tokens again (as though he or she hasnot already receive the claimed rewards) through the `claim()` function. 
 This is the root cause of this vulnerability: failure to update **avgStart[from]** after the for loop execution . 

### Internal pre-conditions

Internal conditions acting as precurssor to this attack is:

1. `StakedEXA.sol:: _update(address from, address to, uint256 amount)` function should be called first with malicious user's address set as the from address, the to address set to the zero address and the amount parameter set to a non-zero value.

2. Malicious user can then call the `claim(reward)` function afterwards. 

### External pre-conditions

_No response_

### Attack Path


1. `StakedEXA.sol:: _update(address from, address to, uint256 amount)` function should be called first with malicious user's address set as the from address, the to address set to the zero address and the amount parameter set to a non-zero value.

2. Malicious user can then call the `claim(reward)` function afterwards. 

### Impact

Let's analyse the `claim()` function to see if it's possible for malicious user to claim rewards for the second time.

```solidity
function claim_(IERC20 reward) internal whenNotPaused {
    uint256 time = block.timestamp * 1e18 - avgStart[msg.sender];
    if (time <= minTime * 1e18) return;

    uint256 claimedAmount = claimed[msg.sender][reward];
    // due to excess exposure
    uint256 claimableAmount = Math.max(rawClaimable(reward, msg.sender, balanceOf(msg.sender)), claimedAmount);
    uint256 claimAmount = claimableAmount - claimedAmount;

    if (claimAmount != 0) claimed[msg.sender][reward] = claimedAmount + claimAmount;

    if (time > refTime * 1e18) 
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

+ Because attacker's  **avgStart[attacker]**  wasnot updated in the burning logic **(else if block)**  of `_update()` function, **avgStart[attacker]** is still in the past, and hence time computation in the above code will be a higher value. 

+ Note that, during the call to `claimWithdraw()`, **claimed[attacker][reward]** was decremented. #See below:
`    claimed[account][reward] -= claimedAmount;`
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L186

+ Note also that, `rawClaimable` is dependent on `discountFactor()` which is also dependent on **avgStart[attacker]**. And since **avgStart[attacker]** hasnot been updated in the burning logic, this implies a stale value for `rawClaimable()`. #see below:
```solidity
uint256 start = avgStart[account];
    if (start == 0) return 0;
    return earned(reward, account, shares).mulWadDown(discountFactor(block.timestamp * 1e18 - start));
  }
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L316-L320

+ What this means is that, **claimableAmount** variable in the `claim()` function will be equal to the returned value of the `rawClaimable()` function. Since **claimableAmount** is a stale value, whereas **claimed[attacker][reward]** was decremented inside `claimWithdraw()`,  the value of **claimAmount** in the above `_claim()` function will be higher.

The code then increments claimed[attacker][reward] by claimableAmount. 


+ The reference staking period is irrelevant to this attack, therefore i skip the analysis for the `if time > refTime * 1e18` block


+ The code then transfers the new higher **claimAmount** to the attacker again. This is the 2nd transfer of reward tokens to the Attacker.# see below:
```solidity
if (claimAmount != 0) {
      reward.transfer(msg.sender, claimAmount);
      emit RewardPaid(reward, msg.sender, claimAmount);
``` 
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L393-L395

+ Note that, the first transfer of reward tokens occurred in the `claimWithdraw()` function call in the **else if block** of  `_update()`. 
 See below:
```solidity
if (claimAmount != 0) {
      reward.transfer(account, claimAmount);
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L194-L195


# IMPACT
Malicious user claims reward tokens twice, which effectively is a  direct loss of funds to the protocol.

### PoC

_No response_

### Mitigation

update the average starting time for user's staked token in the else if block of the `update()` function as below;

Change from:

```solidity
else if (to == address(0)) {
      for (uint256 i = 0; i < rewardsTokens.length; ++i) {
        IERC20 reward = rewardsTokens[i];
        updateIndex(reward);
        claimWithdraw(reward, from, amount);
      }
    }
```


To this:

```solidity
else if (to == address(0)) {
      for (uint256 i = 0; i < rewardsTokens.length; ++i) {
        IERC20 reward = rewardsTokens[i];
        updateIndex(reward);
        claimWithdraw(reward, from, amount);
      }
     +  avgStart[from] = block.timestamp * 1e18
    }
```