Dancing Berry Sparrow

High

# Multiple Reward Claims Exploitation in Single Block

## Summary
The RewardsController contract is vulnerable to reward manipulation due to the ability to call the claim function multiple times within the same block, potentially leading to unfair reward distribution and economic loss for the protocol.

## Vulnerability Detail
The vulnerability stems from the interaction between the claim and update functions. The update function, which is called internally by claim, uses block.timestamp to determine whether to update global reward indexes. However, for multiple calls within the same block, this check fails after the first call, leading to outdated global states being used for reward calculations while still updating individual account states.

The claim function calls update for each market and reward.
update checks if block.timestamp > lastUpdate to decide on updating global indexes.
This condition is false for subsequent calls in the same block.
Global indexes remain static while individual accounts continue to accrue rewards.
## Impact
Users can receive more rewards than intended by making multiple claims in a single block.
## Code Snippet
```solidity
function update(address account, Market market, ERC20 reward, AccountOperation[] memory ops) internal {
    uint256 baseUnit = distribution[market].baseUnit;
    RewardData storage rewardData = distribution[market].rewards[reward];
    {
      uint256 lastUpdate = rewardData.lastUpdate;
      // `lastUpdate` can be greater than `block.timestamp` if distribution is set to start on a future date
      if (block.timestamp > lastUpdate && (lastUpdate < rewardData.end || rewardData.lastUndistributed != 0)) {
        (uint256 borrowIndex, uint256 depositIndex, uint256 newUndistributed) = previewAllocation(
          rewardData,
          market,
          block.timestamp - lastUpdate
        );
        if (borrowIndex > type(uint128).max || depositIndex > type(uint128).max) revert IndexOverflow();
        rewardData.borrowIndex = uint128(borrowIndex);
        rewardData.depositIndex = uint128(depositIndex);
        rewardData.lastUpdate = uint32(block.timestamp);
        rewardData.lastUndistributed = newUndistributed;
        emit IndexUpdate(market, reward, borrowIndex, depositIndex, newUndistributed, block.timestamp);
      }
    }

    mapping(bool => Account) storage operationAccount = rewardData.accounts[account];
    for (uint256 i = 0; i < ops.length; ) {
      AccountOperation memory op = ops[i];
      Account storage accountData = operationAccount[op.operation];
      uint256 accountIndex = accountData.index;
      uint256 newAccountIndex;
      if (op.operation) {
        newAccountIndex = rewardData.borrowIndex;
      } else {
        newAccountIndex = rewardData.depositIndex;
      }
      if (accountIndex != newAccountIndex) {
        accountData.index = uint128(newAccountIndex);
        if (op.balance != 0) {
          uint256 rewardsAccrued = accountRewards(op.balance, newAccountIndex, accountIndex, baseUnit);
          accountData.accrued += uint128(rewardsAccrued);
          emit Accrue(market, reward, account, op.operation, accountIndex, newAccountIndex, rewardsAccrued);
        }
      }
      unchecked {
        ++i;
      }
    }
  }


 function claim(
    MarketOperation[] memory marketOps,
    address to,
    ERC20[] memory rewardsList
  ) public claimSender returns (ERC20[] memory, uint256[] memory) {
    return claim(marketOps, _claimSender, to, rewardsList);
  }
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/RewardsController.sol#L339-#L358

```

## Tool used

Manual Review

## Recommendation
Implement a coolDown 
```solidity
mapping(address => uint256) public lastClaimBlock;
uint256 public constant CLAIM_COOLDOWN = 5; // blocks

function claim(...) public {
    require(block.number >= lastClaimBlock[msg.sender] + CLAIM_COOLDOWN, "Claim cooldown active");
    // ... (existing claim logic)
    lastClaimBlock[msg.sender] = block.number;
}
```