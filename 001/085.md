Strong Fiery Halibut

Medium

# `permitAndDeposit` is allowed to be called by the permit creator only

## Summary

The [`permitAndDeposit`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4) function is allowed to be called by the permit creator only. Not any other contracts will be able to execute these function on behalf of signer.

## Vulnerability Detail

The [`permitAndDeposit`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4) function is meant to permit a spender and deposits assets in a single transaction:

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4
```solidity
function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
    IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
    return deposit(assets, receiver);
  }
```

This function calls OZ's `permit` and passes `msg.sender` as `owner` to that function:

https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC20/extensions/ERC20PermitUpgradeable.sol#L49C5-L72C6
```solidity
function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual {
        if (block.timestamp > deadline) {
            revert ERC2612ExpiredSignature(deadline);
        }

        bytes32 structHash = keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline));

        bytes32 hash = _hashTypedDataV4(structHash);

        address signer = ECDSA.recover(hash, v, r, s);
        if (signer != owner) {
            revert ERC2612InvalidSigner(signer, owner);
        }

        _approve(owner, spender, value);
    }
```

This means, that the signer can be only the same person that called [`permitAndDeposit`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4) function.

However, the purpose of the permit is to allow someone to sign the approve signature, so that this signature can be used by another contract to call some function on behalf of signer.

## Impact

[`permitAndDeposit`](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4) is allowed to be called by the permit creator only.

## Code Snippet

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L173C2-L176C4
```solidity
function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
    IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
    return deposit(assets, receiver);
  }
```

## Tool used

Manual Review

## Recommendation

Use `receiver` instead of `msg.sender`:
```diff
function permitAndDeposit(uint256 assets, address receiver, Permit calldata p) external returns (uint256) {
    -IERC20Permit(asset()).permit(msg.sender, address(this), p.value, p.deadline, p.v, p.r, p.s);
    +IERC20Permit(asset()).permit(receiver, address(this), p.value, p.deadline, p.v, p.r, p.s);
    return deposit(assets, receiver);
  }
```
