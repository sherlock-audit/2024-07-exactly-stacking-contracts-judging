Shambolic Iron Shetland

Medium

# ERC20 non-compliance: Zero value transfer unsupported.

### Summary

The README says ERC-20 strict compliance (stEXA). 

The protocol does not comply with the following part of [EIP-20](https://eips.ethereum.org/EIPS/eip-20#:~:text=tokens%20to%20spend.-,Note%20Transfers%20of%200%20values%20MUST%20be%20treated%20as%20normal%20transfers%20and%20fire%20the%20Transfer%20event.,-function%20transfer)
> Note Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event.





### Root Cause

Zero-value transfer are unsupported as the `_update` function called by `transfer` and `transferFrom` reverts when amount = 0.

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L130
```solidity
  /// @notice Hook to handle updates during token transfer.
  /// @param from The address transferring the tokens.
  /// @param to The address receiving the tokens.
  /// @param amount The amount of tokens being transferred.
  function _update(address from, address to, uint256 amount) internal override whenNotPaused {
    if (amount == 0) revert ZeroAmount();
```


### Internal pre-conditions

n/a

### External pre-conditions

n/a

### Attack Path

n/a

### Impact

Zero value transfer unsupported, non-compliance with ERC-20 spec thereby contradicting README

### PoC

_No response_

### Mitigation

Allow zero-value transfers.