Teeny Marmalade Elk

Medium

# In emergency situations, users cannot withdraw their assets (stEXA).

## Summary
Assets cannot be withdrawn in an emergency situation due to the `whenNotPaused` modifier in the `StakedEXA.sol#_update()` function.
## Vulnerability Detail
The administrator may pause the contract in emergency situations by using `StakedEXA.sol#pause()`.
```solidity
  /// @notice Sets the pause state to true in case of emergency, triggered by an authorized account.
  function pause() external onlyPausingRoles {
    _pause();
  }
```

On the other hand, the `StakedEXA.sol#_update()` function handles updates during token transfer (i.e. deposit/mint, withdraw/redeem).
However, the `StakedEXA.sol#_update()` function is limited to the `whenNotPaused` modifier.
```solidity
function _update(address from, address to, uint256 amount) internal override whenNotPaused {
```

As a result, in emergency situations, the `withdraw/redeem` functions revert and the user cannot withdraw assets.
Therefore, users may not be able to withdraw assets in time in emergency situations, which may result in potential loss of funds.

This problem is also considered a valid problem considering the points mentioned in SOLODIT's [Checklist](https://solodit.xyz/checklist).

SOL-CR-1:
 >Description: Users must be allowed to manage their existing positions in all protocol status. For example, users must be able to repay the debt even when the protocol is paused or the protocol should not accrue debts when it is paused.

SOL-CR-2:
 >Description: Some functionalities must work even when the whole protocol is paused. For example, users must be able to withdraw (or repay) assets even while the protocol is paused.

Additionally, since the `whenNotPaused` modifier is implemented in the `harvest()` function, `deposit/mint` functions will be effectively reverted in emergency situations.
## Impact
Users may not be able to withdraw assets in time in emergency situations, which may result in potential loss of funds.
## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L512-L515
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L129
## Tool used

Manual Review

## Recommendation
It is recommended to modify the `StakedEXA.sol#_update()` function as follows:
```solidity
---     function _update(address from, address to, uint256 amount) internal override whenNotPaused {
+++     function _update(address from, address to, uint256 amount) internal override {
```