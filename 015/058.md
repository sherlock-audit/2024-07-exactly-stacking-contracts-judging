Rare Shamrock Sidewinder

Medium

# FLAWED LOGIC IN REWARD RATE COMPUTATION IN notifyRewardAmount() FUNCTION LEAD TO UNFAIR REWARD DISTRIBUTION

### Summary

In the course of a reward distribution period, the contract's owner is allowed to top up reward tokens in the system, that is, supply additional reward tokens to the contract. The code then updates the reward rate to reflect the newly added tokens. However, the rate update is wrongly computed because the code fails to refactor in the remaining time left to a reward's stoppage time, leading to a higher reward rate.

### Root Cause

In StakedEXA.sol:216, the reward rate calculation does not consider the remaining time to a reward's distribution stoppage time. See below:

```solidity
     else {
      uint256 remainingRewards = (rewardData.finishAt - block.timestamp) * rewardData.rate;
      rewardData.rate = (amount + remainingRewards) / rewardData.duration;
  }
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L216

### Internal pre-conditions

The vulnerability happens whenever the ADMIN supplies additional reward tokens during the course of a reward distribution via **notifyRewardAmount()** function

### External pre-conditions

There are a number of functions that depends on the value of a reward's rate. Let's analyze a number of them:

1. `finishDistribution()` function:   This is a privileged function callable only by the ADMIN. With respect to the scenario I describe in my PoC section of this report, if ADMIN had earlier called **notifyRewardAmount()** during the course of a reward distribution, and the later call this function, due to the inflated reward rate, ADMIN will transfer more reward tokens to the savings address than necessary or required.  # See below:
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L431


2. `globalIndex()` function:     Now, this function makes computations that depends on the reward rate of a reward. This implies with a higher reward rate, the return value of this function will also be higher. 

```solidity
return
      rewardData.index +
      (rewardData.rate * (lastTimeRewardApplicable(rewardData.finishAt) - rewardData.updatedAt)).divWadDown(
        totalSupply()
      );
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L287-L291


3. `earned()` function:   This function calculates a user's earned rewards . The computation performed by this function is dependent on the return value of **globalIndex()** which in turn is dependent on the inflated reward rate updated by the ADMIN through supplying additional reward tokens during the course of distribution. #See code snippet below:

```solidity
function earned(IERC20 reward, address account, uint256 shares) public view returns (uint256) {
    return shares.mulWadDown(globalIndex(reward) - avgIndexes[account][reward]);
  }
```
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L307-L309

Now, due to this flawed logic, with no change in the ADMIN supply tokens and no change in a reward's duration during the course of a reward distribution, 2 separate users ( USER A and USER B ) with the same shares will receive different rewards, making the rewards distribution mechanism unfairly propagated.  


### Attack Path

For 2 separate accounts, ACCOUNT A and ACCOUNT B, with the same shares in the protocol, and with constant supply reward tokens and static reward duration set by ADMIN during the course of a distribution, should receive the same amount of reward tokens. However, due to the incorrect logic in the new rate calculation when ADMIN supply constant additional reward tokens and non-changed reward Duration, one account will receive a higher rewards than the other.  

If ACCOUNT B waits for the ADMIN to supply constant additional reward tokens, he will get higher rewards than ACCOUNT A even though they had the same shares, and the amount of reward tokens supplied and reward Duration was not changed by ADMIN. 


### Impact

There is unfair distribution of rewards even though there has not been any change

### PoC


With definitive values, I am gonna showcase the  logic error. 

1. let's assume the owner adds a certain reward token called token RA, and then sets the duration for this reward token RA to 24 weeks which is equivalent to (24 * 7 * 86400) 14_515_200 seconds.

2. let's assume also that the amount of token RA transferred by the owner is 7_257_600

3. let's assume also that the distribution for this reward token RA has stopped, implying that, block.timestamp >= reward[token_RA].finishAt . In this case, the reward rate is calculated as shown:

**reward rate = amount / duration**;


For the assumed values, 

**reward rate = 7_257_600 / 14_515_200 ;
reward rate = 0.5**

For this reward token_RA, the initial reward rate is 0.5 token_RA per second.

NOTE: I wanted to get an initial reward rate for token_RA since it's clear that for every reward token, there's an initial reward rate. The **notifyRewardAmount()** just updates the rate based on the new amount of reward tokens.

NOTE 2: For a mathematical evenly probability distribution, if the independent variables of a rate calculation does not change, then the dependent variable must remain constant. In the rate calculation, amount of tokens and duration are the independent variables whereas the reward rate is the dependent variable. 
In short, for 2 different phases, if amount of tokens and duration in phase 1 is the same as the amount of tokens and duration in phase 2, then reward rate in phase 1 must be equal to reward rate in phase 2. 


4. Assume, in a second phase,  the owner wants to add more of reward token token_RA at the time when 20 weeks of the reward's duration has passed. 

2ND PHASE:
amount of token_RA = 7_257_600
duration = 24 weeks. 

**@note: Because in this phase, same amount of tokens are being supplied with the same duration set, as in phase 1, the probability evenly distribution stipulates that the rate of reward should be constant throughout the period from phase 1 to phase 2.**

### Current Code Implementation For The Reward Rate Spread If Tokens Are Added During The Course Of A Distribution 

```solidity
} else {
      uint256 remainingRewards = (rewardData.finishAt - block.timestamp) * rewardData.rate;
      rewardData.rate = (amount + remainingRewards) / rewardData.duration;
    }
```

**let , weeks that has passed = 20 weeks = 12_096_000 seconds**;
 **let, duration = rewardData.finishAt = 24 weeks = 14_515_200 seconds**;


```solidity
remainingRewards = ( 14_515_200  -  12_096_000 ) * 0.5   =  1_209_600 ;

rewardData.rate = ( 7_257_600   +   1_209_600)  / 14_515_200   =  0.583
```

Now, with the current code implementation, it is seen that for the same token amount and same duration by the owner, the reward rate is not evenly distributed leading to unfair distribution of rewards. 




### My Suggested Logic For Fairness Of Reward Distribution

Just as in the update of the reward rate, the remaining rewards of the current phase is added to the newly to-be supplied amount, so must the remaining duration of the current phase be added to the newly to-be set duration. 

```solidity
} else {
      uint256 remainingDuration = rewardData.finishAt  -  block.timestamp;
      uint256 remainingRewards = remainingDuration * rewardData.rate;
      rewardData.rate = (amount + remainingRewards) / (rewardData.duration + remainingDuration);
    }
```
For the same values as in the 2ND PHASE:

**let , weeks that has passed = 20 weeks = 12_096_000 seconds**;

**let, duration = rewardData.finishAt = 24 weeks = 14_515_200 seconds**;



```solidity
    remainingDuration = 14_515_200   -    12_096_000 =    2_419_200   ;
    remainingRewards = 2_419_200 * 0.5 = 1_209_600 ;
    rewardData.rate  = ( 7_257_600  +  1_209_600  )  /  ( 14_515_200  + 2_419_200 ) =  0.5
``` 

### Mitigation

The intended recommendation to ensure fairness is to refactor the remaining time to a reward stoppage in the new rate calculation just as the remaining rewards was refactored into the total amount. 

Change 

from:

```solidity
 else {
      uint256 remainingRewards = (rewardData.finishAt - block.timestamp) * rewardData.rate;
      rewardData.rate = (amount + remainingRewards) / rewardData.duration;
    }
```



To this:

```solidity
 else {
      uint256 remainingDuration = rewardData.finishAt  -  block.timestamp;
      uint256 remainingRewards = remainingDuration * rewardData.rate;
      rewardData.rate = (amount + remainingRewards) / (rewardData.duration + remainingDuration);
    }
```