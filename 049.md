Rare Shamrock Sidewinder

High

# MATH ERROR IN StakedEXA::discountFactor() function will lead to loss of protocol and user's savings funds.

### Summary

+ Users who have elapsed the 24 weeks reference stake time will claim far greater rewards and lose their savings as protocol too  loses its saving funds due to the **refTime** being doubly multiplied by 1e18 in discountFactor() function. 



### Root Cause

+ In StakedEXA.sol:240, within the discountFactor() function,  the **refTime** is doubly multiplied by 1e18. From L237 of the said contract,   `uint256 memRefTime = refTime * 1e18` .           Now, at L240, the memRefTime is further multiplied by 1e18 in the return value, ie, that is , there is:
                  ====>>>>           `memRefTime * 1e18` which literally equals to `refTime * 1e18 * 1e18` . See the code snipet below:

`       return (1e18 - memExcessFactor).mulWadDown((memRefTime * 1e18) / time) + memExcessFactor; `
            https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L240


### Internal pre-conditions

The only internal pre-condition to this logic error is that user's staked time should exceed the 24 weeks reference stake period, that is, user must have elapsed the default 24 weeks reference stake period.

### External pre-conditions

Although the **discountFactor()** function in question is an internal type, there are a lot of public and other internal functions that calls to this function, and the execution of these other functions is dependent on the return value of this vulnerable function. Some include:

1. `rawClaimable()` function : The return value of this public function is dependent on the return value of **discountFactor()** function. #See L319: ` return earned(reward, account, shares).mulWadDown(discountFactor(block.timestamp * 1e18 - start));`
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L319


Now, there are other functions that indirectly depends on the return value of the **discountFactor()** function. Some are:

2. `claimWithdraw()` function: This function calls **rawClaimable()** function which in turn is dependent on the return value of the vulnerable **discountFactor()** function. _#See StakedEXA:L192 :_ 
`   uint256 claimableAmount = Math.max(rawClaimable(reward, account, amount), claimedAmount); `
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L192

3. `claimable()` function: This function calls **rawClaimable()** function which in turn is dependent on the return value of the vulnerable **discountFactor()** function. #See StakedEXA.sol:L331
`    uint256 rawClaimable_ = rawClaimable(reward, account, shares);`
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L331
         + In fact, the return value from this function is solely dependent on the return value from the **rawClaimable()** function call. And a flawed **rawClaimable(**) return value means the returned value of this **claimable()** function is also flawed. #See L337:
`    return rawClaimable_ > claimedAmountProportion ? rawClaimable_ - claimedAmountProportion : 0;`
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L337

4. `_claim()` function: This internal function too calls the **rawClaimable()** function whose return value is in turn dependent on the return value of the vulnerable **discountFactor()** function. #See StakedEXA.sol:L377
`    uint256 claimableAmount = Math.max(rawClaimable(reward, msg.sender, balanceOf(msg.sender)), claimedAmount);`
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L377

- [ ] Now, inside this `_claim()` function, a flawed **claimableAmount** means a flawed **claimAmount{L378}**, a flawed **maxClaim{L385}** , and consequently a flawed **saveAmount {386}** . Now, this means a wrong amount transfer to savings.


### Attack Path

_No response_

### Impact

1. + Now, the core protocol invariant that the discountFactor should always be between 0 and 1e18 will be broken as this return value will indeed far exceed 1e18.



2. Now, I will take this moment to showcase the impact of this vulnerability on the `claimWithdraw` function.
Below is the **claimWithdraw()** function for analysis:
```solidity
function claimWithdraw(IERC20 reward, address account, uint256 amount) internal {
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
      reward.transfer(account, claimAmount);//
      emit RewardPaid(reward, account, claimAmount);
    }

    uint256 rawEarned = earned(reward, account, amount);
    // due to rounding
    uint256 saveAmount = rawEarned <= claimableAmount + savedAmount ? 0 : rawEarned - claimableAmount - savedAmount;
    if (saveAmount != 0) reward.transfer(savings, saveAmount);
  }
```

 +  Now since the calculations are done in 1e18 decimals, this is acting as a base. I will assume that 1e18 is equivalent to 100%. And due to the math error, since the discountFactor() return value will far exceed 1e18, it means it will exceed 100% too.


 + Now, in rawClaimable() function, the computation that is returned, in simplest terms, is: 
  ` rawClaimable() -> return value  ===>>>>>    Amount earned * discountFactor`
And since, discountFactor > 100%, it implies the returned valued will also be far greater than Amount earned. 

For instance, if discountFactor in percentage terms is 150%, that means
>>>>>       rawClaimable()-> return value ====== 1.5 * Amount earned


+ Now, within the `_claimWithdraw()` function, it implies user will get a higher claimAmount because of the increased claimableAmount. 


+ At the same time, because claimableAmount is far greater than the earned amount (In the instance above, it is 50% greater than the rawEarned), saveAmount = 0, and the savings account receives nothing even though user's saved amount has already been decremented beforehand [`    saved[account][reward] -= savedAmount;`]


+ Now, the implication is that, the protocol will lose the saving funds of all users who claim after the 24 weeks reference stake period has elapsed. 

### PoC

 In this PoC section, I am gonna use protocol defined values to demonstrate that the discountFactor can indeed exceeds 1e18. This is the computation:
```solidity
uint256 memRefTime = refTime * 1e18;
    if (time >= memRefTime) {
      uint256 memExcessFactor = excessFactor;
      return (1e18 - memExcessFactor).mulWadDown((memRefTime * 1e18) / time) + memExcessFactor;
    }
```

I am taking these values based on the `stakedEXA.t.sol` file. From this test file,

1. refTime = 24 weeks = 14_515_200 seconds ;
2. excessFactor = 0.5e18

+ Now, the time parameter in this function refers to the difference of the current block.timestamp and a user's average starting time for staked tokens, all in 1e18 decimals base. # See 
```solidity
  uint256 start = avgStart[account];
    if (start == 0) return 0;
    return earned(reward, account, shares).mulWadDown(discountFactor(block.timestamp * 1e18 - start));
  }
```
NOTE: So literally, the code is: **discountFactor(block.timestamp * 1e18 - avgStart[user])**


+ if the reference staked time has elapsed, then the below math is valid:
`refTime >= block.timestamp * 1e18  -  avgStart[user]`
This therefore implies that, the minimum value for the time parameter in the discountFactor computation above is refTime. 

3. time  in 1e18 base decimals = = 14_515_200 * 1e18;



# Now, in essence, the code can be rewritten as:
```solidity
((1e18 - excessFactor) * (refTime * 1e18 * 1e18) / time)  + excessFactor
```

computing the formula using the defined values will yield:
returned discountFactor = 5e35 + 5e17 = 5e35. 
Now, this value is far greater than 1e18, breaking a core protocol invariant and giving rise to the vulnerability. 

For elapsed time of 36 weeks (21772800 seconds),
returned discountFactor = 0.666667e36. 

### Mitigation

The corrected formula is to be:

`( (1e18 - memExcessFactor) * (memRefTime) / Time) + memExcessFactor `

OR 

`( (1e18 - memExcessFactor) * (refTime * 1e18)  / Time) + memExcessFactor `


I believe that with this new computation, precision will be achieved the right way (multiplication before division, and due to the default solidity rounding down) without giving rise to the vulnerability.