Large Fuchsia Dove

Medium

# Deposits made when `providerRatio_` is set to `0` will never send rewards from the provider to savings when harvesting, harming the protocol

### Summary

`StakedEXA::harvest()`, called when depositing, more specifically in `StakedEXA::update()` returns if the amount from the provider is smaller than the duration of the rewards. However, the provider ratio may be set to 0 to always send the rewards to savings, in which case it will still return early instead of depositing to `savings`. This will harm the protocol as it will not have access to these funds.

### Root Cause

In `StakedEXA::354`, `StakedEXA::harvest()`, it [returns](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L354) if the amount is smaller than the duration `if (duration == 0 || amount < rewards[providerAsset].duration) return;`. It does this to prevent rounding down to 0 when notifying the reward amount, but does not consider the fact that the amount may be 0 or very small if `providerRatio` is null, which is a possible value if the protocol intends to send all assets to savings.

### Internal pre-conditions

1. `providerRatio` is null.

### External pre-conditions

None.

### Attack Path

1. Provider approves assets to `StakedEXA`.
2. Deposits do not send provider assets to savings because it returns early as the amount from the provider is 0.

### Impact

Protocol does not collect savings.

### PoC

No POC is required as the early return is straightforward.

### Mitigation

It should not do an early return if the savings amount is significant.