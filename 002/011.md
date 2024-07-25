Curly Vanilla Barracuda

Medium

# Incorrect Comparison in Harvest Function Leading to Potential Distribution Issues

## Summary
In the `harvest` function of the provided smart contract, there is a comparison intended to ensure the correctness of the calculated `amount`. However, the current implementation uses a flawed condition that may lead to incorrect logic.

```solidity
  uint256 amount = assets.mulWadDown(providerRatio);
    IERC20 providerAsset = IERC20(address(memMarket.asset()));
    uint256 duration = rewards[providerAsset].duration;
    if (duration == 0 || amount < rewards[providerAsset].duration) return; << @audit-info
```
The purpose of the `harvest` function is to distribute rewards based on the `providerRatio`. The `amount` is derived from `assets` and `providerRatio`, and the `duration` ensures that the reward distribution period is set. However, the comparison `amount < rewards[providerAsset].duration` does not logically correlate amount with duration, potentially leading to incorrect behavior.
## Vulnerability Detail
**Contract Context**:
- Assume the contract is designed to distribute rewards to users based on their assets and a provider ratio.
- The `harvest` function is responsible for calculating and distributing these rewards.

**Assumptions**:
- `providerAsset` is a token that users have staked or provided.
- `amount` represents the amount of rewards that should be distributed.
- `rewards[providerAsset].duration` represents a threshold value that should be checked against the calculated amount to decide if rewards can be distributed.

**Original Check**:
```solidity
if (duration == 0 || amount < rewards[providerAsset].duration) return;
```
**Faulty Comparison Scenario**:
1. State Before Harvest:

- `providerAsset` = "TokenA"
- `duration` = 0 (indicating that the contract might be in a state where duration isn't relevant or hasn't started yet)
- `amount` = 100 (calculated based on assets and provider ratio)
- `rewards[providerAsset].duration` = 200

2. Expected Behavior:

- If `duration` is 0, the function should continue regardless of `amount` since no threshold check is required.
- If `amount` is less than `rewards[providerAsset].duration`, the function should halt because it's not worth distributing rewards.

3. Incorrect Comparison Issue:

- The faulty comparison `amount < rewards[providerAsset].duration` should have been `amount < rewards[providerAsset].duration` (where amount should be compared with a calculated threshold based on current conditions, not just `duration`).

4. **Result**:

- Given the state, the condition `duration == 0` would normally allow the function to proceed with reward distribution.
- However, due to the incorrect comparison, if `amount` (100) is compared directly with `rewards[providerAsset].duration` (200), it incorrectly triggers the return condition even though it shouldn’t.

## Impact
- The `harvest` function exits prematurely due to the faulty comparison, resulting in no rewards being distributed.
- Users who expect to receive rewards based on their assets are left dissatisfied as the contract fails to execute reward distribution correctly.
## Code Snippet
[StakedEXA.sol#L354](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L354)
## Tool used

Manual Review

## Recommendation
 The comparison should be simplified to check for non-zero `duration` and a valid `amount`, ensuring that the reward distribution occurs only when both are appropriate.
 PERHAPS:
 ```solidity
 if (duration == 0 || amount < assets.mulWadDown(providerRatio)) return;
```