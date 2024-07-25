Large Fuchsia Dove

High

# Depositing to another receiver othan than `msg.sender` will lead to stuck funds by increasing `avgStart` without claiming

### Summary

`StakedEXA::_update()` is called when minting and if the time staked is smaller than the reference time, it calls `_claim(reward), which claims the rewards for the `msg.sender`. However, the mint may be to someone else other than `msg.sender`, so `avgStart[to]` will increase without first claiming, which will lead to loss of rewards for the receiver of the new deposit.

### Root Cause

In `StakedEXA.sol::146` on [claim(reward)](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L146), it [claims](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L371) to the `msg.sender`, but the deposit is done to the `to` address, which may be different.

### Internal pre-conditions

1. Anyone must do a deposit to a different address.

### External pre-conditions

None.

### Attack Path

1. User has some StakedEXA from previous deposits.
2. Another user deposits/mints to user 1, which increases `avgStart[user1]`, but does not claim first for user1, making the funds stuck.

### Impact

The funds are stuck for much longer than they are supposed to. The duration is `24 weeks` in the tests, so users can have to wait up to 24 weeks more if they do a really big deposit while having fully claimable rewards. Additionally, in case they decide to withdraw after depositing, before the rewards vest again, they will lose these funds to savings.

### PoC

Paste the following test in `StakedEXA.t.sol` to confirm that `address(this)` deposits more without claiming.
```solidity
function test_POC_increaseBalance_withoutClaim() external {
  uint256 assets = 1e18;
  uint256 time = minTime;

  exa.mint(address(this), assets);
  stEXA.deposit(assets, address(this));

  uint256 initTime = block.timestamp * 1e18;
  skip(time + 1);
  uint256 finalTime = block.timestamp * 1e18;

  assertEq(rA.balanceOf(address(this)), 0);
  assertEq(stEXA.avgStart(address(this)), initTime);
  assertEq(stEXA.rawClaimable(rA, address(this), assets), 20833367779982251207);
  assertEq(stEXA.earned(rA, address(this), assets), 41666735559964287164);

  address anotherAccount = makeAddr("AnotherAccount");
  exa.mint(anotherAccount, assets);
  vm.startPrank(anotherAccount);
  exa.approve(address(stEXA), assets);
  stEXA.deposit(assets, address(this));
  vm.stopPrank();

  uint256 newAssets = 2*assets;

  assertEq(rA.balanceOf(address(this)), 0);
  assertEq(stEXA.avgStart(address(this)), (initTime + finalTime) / 2);
  assertEq(stEXA.rawClaimable(rA, address(this), newAssets), 0);
  assertEq(stEXA.earned(rA, address(this), newAssets), 41666735559964287164);
}
```

### Mitigation

Claiming should be made to the receiver of the funds, `to`. The `_claim()` function should be adapted to receiver an `account` argument.