Ancient Wintergreen Tardigrade

Medium

# In modifier `permitSender` directly encoding `assets` dynamic array to calculate `hashStruct` is wrong according to `EIP712` specification.

## Summary

In modifier `permitSender` directly encoding `assets` dynamic array to calculate `hashStruct` is wrong according to `EIP712` specification.

## Vulnerability Details

According to EIP-712 specification :
_The array values are encoded as the keccak256 hash of the concatenated encodeData of their contents_ [See](https://eips.ethereum.org/EIPS/eip-712#definition-of-encodedata)

The `permitSender` modifier in the `RewardsController` contract computes the message hash by passing directly dynamic array into abi.encode with other params to calculate hashStruct. While as string or bytes keccak256 hash should be used to concatenate with other params.
Similarly for dynamic arrays the keccak256 hash of the concatenated encodeData of dynamic arrays should be used instead of directly using dynamic array. It will result in wrongly calculated hash. This inconsistency will lead to interoperability issues and could affect the proper validation of signatures across different platforms and tools that rely on EIP-712.

Here is the [article see 2.2](https://mirror.xyz/jaredborders.eth/G2RP5XAfLbNZv01DXgxuzv_34bQF_PuO1X2u0Nhop9g) that explains how to encode complex data like dynamic arrays to compute message hash to be signed according to EIP712 specification.

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
785:                keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 deadline)"),
786:                permit.owner,
787:                msg.sender,
788:                permit.assets,//@audit dynamic array directly passed to be encoded with other params, it's concatenated hash should be passed
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

So using `keccak256(abi.encodePacked(permit.assets))` will work instead of using `permit.assets`. Since it is made atomic now since hash is type bytes32 which is a atomic data type.

## Impact

Not following EIP712 standards properly and passing directly `assets` dynamic array to be concatenated with other params to calculate `hashStruct` will finally result in calculating wrong message hash. Which could lead to create wrong signatures.

## Tools Used

Manual Review

## Recommendation

Use `keccak256(abi.encodePacked(permit.assets))` instead of using `permit.assets` to calculate correct hash according to EIP712 specification.

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
785:                keccak256("ClaimPermit(address owner,address spender,address[] assets,uint256 deadline)"),
786:                permit.owner,
787:                msg.sender,
- 788:                permit.assets,
+ 788:                keccak256(abi.encodePacked(permit.assets)),
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
