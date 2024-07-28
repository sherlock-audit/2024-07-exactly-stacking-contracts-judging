Flaky Watermelon Spider

Medium

# StakeEXA.permitandDeposit can be DoSed by front-running user and performing the permit()

## Summary
The `StakedEXA` contract having a vulnerability in its `permitAndDeposit` function that makes it susceptible to front-running attacks. This vulnerability could lead to Denial of Service (DoS) for users attempting to deposit funds using the permit functionality.

## Vulnerability Detail
The `permitAndDeposit` function combines an EIP-2612 permit call with a deposit action in a single transaction. However, it does not implement any protection against front-running attacks. An attacker can observe the transaction in the mempool, extract the permit signature, and execute a separate permit transaction before the original `permitAndDeposit` transaction is processed.

StakedEXA is using `OZ` ERC20PermitUpgradeable which clearly states about using permit function: 

```solidity
 ==== Security Considerations
 *
 * There are two important considerations concerning the use of `permit`. The first is that a valid permit signature
 * expresses an allowance, and it should not be assumed to convey additional meaning. In particular, it should not be
 * considered as an intention to spend the allowance in any specific way. The second is that because permits have
 * built-in replay protection and can be submitted by anyone, they can be frontrun. A protocol that uses permits should
 * take this into consideration and allow a `permit` call to fail. Combining these two aspects, a pattern that may be
 * generally recommended is:
 *
 * 
 * function doThingWithPermit(..., uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s) public {
 *     try token.permit(msg.sender, address(this), value, deadline, v, r, s) {} catch {}
 *     doThing(..., value);
 * }
 *
 * function doThing(..., uint256 value) public {
 *     token.safeTransferFrom(msg.sender, address(this), value);
 *     ...
 * }
```

## Impact
The primary impact of this vulnerability is potential Denial of Service (DoS) for users. When a user attempts to use the `permitAndDeposit` function, their transaction may fail due to a front-running attack. This can result in:

- Failed deposits, preventing users from staking their tokens.

## Code Snippet
[function permitAndDeposit](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/3eb87e3edf3bcd57c4cc1c6a73e8255f575b76de/protocol/contracts/StakedEXA.sol#L173)
```solidity
function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
  IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
  return deposit(assets, receiver);
}
```

## Tool used
Manual Review

## Recommendation
To mitigate this vulnerability, implement a try-catch block around the permit call. the use of `try/catch` allows the permit to fail and makes the code tolerant to frontrunning.

```diff
function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
-  IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
+ try IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s) {} catch {}
  return deposit(assets, receiver);
}
```
