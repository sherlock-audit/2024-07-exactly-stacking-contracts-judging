Ancient Wintergreen Tardigrade

Medium

# In modifier `permitSender` according to `EIP712` specification wrong typeHash used in hashStruct calculation, missing `uint256 nonce` parameter in typeHash calculation.

## Summary

In modifier `permitSender` according to `EIP712` specification wrong `typeHash` used in `hashStruct` calculation, missing `uint256 nonce` parameter in typeHash calculation.

## Vulnerability Details

The `permitSender` modifier in the `RewardsController` contract computes the message hash using a method that deviates from the [EIP-712 standard](https://eips.ethereum.org/EIPS/eip-712#definition-of-hashstruct). Specifically the hashStruct construction includes a `nonce` parameter that is not defined in `typeHash` used in encoding data as per EIP-712 specifications.
This will cause calculating wrong `hashStruct` and finally creating wrong digest hash. This inconsistency will lead to interoperability issues and could affect the proper validation of signatures across different platforms and tools that rely on EIP-712.
`typeHash` and `hashStruct` are defined in [EIP-712 standard](https://eips.ethereum.org/EIPS/eip-712#definition-of-hashstruct)

[RewardsController.sol#L774-L798](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/RewardsController.sol#L774-L798)

```solidity
File : contracts/RewardsController.sol

774:  modifier permitSender(ClaimPermit calldata permit) {
775:    assert(_claimSender == address(0));
776:    assert(permit.deadline >= block.timestamp);
777:    unchecked {
778:      address recoveredAddress = ecrecover(
779:        keccak256(
780:          abi.encodePacked(
781:            "\x19\x01",
782:            DOMAIN_SEPARATOR(),
783:            keccak256(
784:              abi.encode(
785:                keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 deadline)"),//@audit missing nonce parameter which is included in encoded parameters
786:                permit.owner,
787:                msg.sender,
788:                permit.assets,
789:                nonces[permit.owner]++,
790:                permit.deadline
791:              )
792:            )
793:          )
794:        ),
795:        permit.v,
796:        permit.r,
797:        permit.s
798:      );

```

Here we can clearly see the typeHash used here is `keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 deadline)")` doesn't have `nonce` parameter which is included in encoded parameters just before `permit.deadline` parameter.
So this typeHash calculation is wrong due to missed parameter which will cause finally creating the wrong hash according to EIP712 specification.

## Impact

Due to missed missed nonce parameter `typeHash` will be wrongly calculated which will cause finally creating the wrong hash according to EIP712 specification.

## Tools Used

Manual Review

## Recommendations

Use `uint256 nonce` parameter in calculating `typeHash` when calculating `hashStruct` like below in `permitSender` modifier.

```diff
File : contracts/RewardsController.sol

774:  modifier permitSender(ClaimPermit calldata permit) {
775:    assert(_claimSender == address(0));
776:    assert(permit.deadline >= block.timestamp);
777:    unchecked {
778:      address recoveredAddress = ecrecover(
779:        keccak256(
780:          abi.encodePacked(
781:            "\x19\x01",
782:            DOMAIN_SEPARATOR(),
783:            keccak256(
784:              abi.encode(
- 785:                keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 deadline)"),
+ 785:                keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 nonce,uint256 deadline)"),
786:                permit.owner,
787:                msg.sender,
788:                permit.assets,
789:                nonces[permit.owner]++,
790:                permit.deadline
791:              )
792:            )
793:          )
794:        ),
795:        permit.v,
796:        permit.r,
797:        permit.s
798:      );

```
