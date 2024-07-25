Able Azure Gecko

Medium

# `permitAndDeposit` will be broken with tokens that do not follow `ERC2612` standard

## Summary
The `StakedEXA.sol` contract's `permitAndDeposit` function assumes all tokens adhere to the `IERC20Permit` standard. However, tokens like `DAI` use a different `permit` function signature. This discrepancy causes permit transactions to revert for such tokens, preventing their use in deposits.

## Vulnerability Detail
According to the contest [readme](https://audits.sherlock.xyz/contests/396#:~:text=If%20you%20are%20integrating%20tokens%2C%20are%20you%20allowing%20only%20whitelisted%20tokens%20to%20work%20with%20the%20codebase%20or%20any%20complying%20with%20the%20standard%3F%20Are%20they%20assumed%20to%20have%20certain%20properties%2C%20e.g.%20be%20non%2Dreentrant%3F%20Are%20there%20any%20types%20of%20weird%20tokens%20you%20want%20to%20integrate%3F), protocol is meant to integrate any ERC20 tokens (without weird traits):

> If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of weird tokens you want to integrate?

>only fully ERC-20 compliant tokens without weird traits.


`StakedEXA.sol` has a [permitAndDeposit](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L173) function which lets a user permit a spender and deposits assets in a single transaction.

```js
  function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
    IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
    return deposit(assets, receiver);
  }
```
However, certain tokens may not adhere to the `IERC20Permit` standard. For example, the `DAI` Stablecoin utilizes a `permit()` function that deviates from the reference implementation.

Below is the `DAI` token's permit signature:

```js
function permit(address holder, address spender, uint256 nonce, uint256 expiry, bool allowed, uint8 v, bytes32 r, bytes32 s) external {}
```
We can clearly see the code that the permit function call does not match the signature of `DAI's` permit function

## Impact
Due to the missing `nonce` field, `DAI`, a token which allows permit based interactions, cannot be used with signed messages for depositing. Due to the wrong parameters, the permit transactions will revert.

## Code Snippet
https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L173

## Tool used
Manual Review

## Recommendation
Update the `permitAndDeposit` function to accomodate for tokens like `DAI` with unique permit signatures
