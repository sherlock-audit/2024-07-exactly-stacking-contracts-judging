Tiny Paisley Mink

High

# Loss of user's fund via donation attack in `permitAndDeposit()`

## Summary
- Loss of user's fund via donation attack in `permitAndDeposit()`
## Vulnerability Detail
-When totalSupply is zero, an attacker goes ahead and executes the following steps:
- They deposit 1 wei of the underlying token through `permitAndDeposit()`.
- They will get back 1 wei of share Token because totalSupply is zero.
- They transfer z underlying tokens directly to ERC4626 Vault's address.
- This leads to 1 wei of share Token worth z (+ some small amount).
- Even though protocol uses `ERC4626 Upgradeable` still the attack is possible as the protocol team didn't override the `_decimalsOffset` value which makes the `attack` still possible.
- Let me give an example -
  - Let's assume there is a first user trying to mint some share Token in Vault using their k*z underlying tokens.
  - An attacker carry out the above-described attack making sure that k<1.
  - This leads to the first depositor getting zero share Tokens for their k*z underlying tokens. Thus, 100% loss of funds(Now the underlying reserve of the Vault contract is also increased with each deposit, so the upcoming depositor will have to deposit an even greater amount to mint some share Token)
  - All upcoming depositors will face the same issue.


## Impact
- This attack has two implications: Implicit minimum Amount and funds lost due to rounding errors
    - If an attacker is successful in making 1 share worth z assets and a user tries to mint shares using k*z assets then,
        - If k<1, then the user gets zero share and they loose all of their tokens to the attacker
        - If k>1, then users still get some shares but they lose (k- floor(k)) * z) of assets which get proportionally divided between existing share holders (including the attacker) due to rounding errors.
        - This means that for users to not lose value, they have to make sure that k is an integer.
## Code Snippet
- https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173

## Tool used

Manual Review

## Recommendation
- Override the value of `_decimalsOffset` in ERC4626 Upgradeable contract to token's decimal (Eg - 1e18 or 1e6).