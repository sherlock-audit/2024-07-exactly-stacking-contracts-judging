Large Fuchsia Dove

Medium

# Frozen/paused Market that is harvested from in StakedEXA will DoS deposits leading to loss of yield

### Summary

`StakedEXA::harvest()` withdraws from the `market` of the `StakedEXA` contract if the provider has assets there and has approved `StakedEXA`. However, the market may be paused/frozen, but `StakedEXA::harvest()` will try to withdraw anyway, which will make it revert. As it is part of the deposit flow, it means all deposits will revert and new users will not be able to collect yield.

### Root Cause

In `StakedEXA::_update()`, `StakedEXA::harvest()` is called if it is a mint (`from == address(0)`), [here](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L156). Then. in `StakedEXA:harvest()`, it [withdraws](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L356) from the `market` regardless of its state, reverting if frozen or paused.

### Internal pre-conditions

None.

### External pre-conditions

1. Market is frozen or paused.

### Attack Path

1. Market is frozen or paused.
2. StakedEXA deposits are frozen.

### Impact

Frozen `StakedEXA` deposits which means new users will miss out on the yield and current users will enjoy a guaranteed higher yield.

### PoC

Add the following test to `StakedEXA.t.sol` confirming that deposits are halted when the market is paused.
```solidity
function test_POC_halted_deposits() external {
  uint256 assets = 1e18;

  market = Market(address(new ERC1967Proxy(address(new Market(ERC20Solmate(address(providerAsset)), new Auditor(18))), "")));
  market.initialize(
    "STEXA",
    3,
    1e18,
    new InterestRateModel(
      IRMParameters({
        minRate: 3.5e16,
        naturalRate: 8e16,
        maxUtilization: 1.1e18,
        naturalUtilization: 0.75e18,
        growthSpeed: 1.1e18,
        sigmoidSpeed: 2.5e18,
        spreadFactor: 0.2e18,
        maturitySpeed: 0.5e18,
        timePreference: 0.01e18,
        fixedAllocation: 0.6e18,
        maxRate: 15_000e16
    }),
    Market(address(0))),
    0.02e18 / uint256(1 days),
    1e17,
    0,
    0.0046e18,
    0.42e18
  );
  market.grantRole(market.PAUSER_ROLE(), address(this));

  vm.prank(address(stEXA));
  providerAsset.approve(address(market), type(uint256).max);

  stEXA.setMarket(market);

  providerAsset.mint(PROVIDER, 1_000e18);
  vm.startPrank(PROVIDER);
  providerAsset.approve(address(market), type(uint256).max);
  market.deposit(1_000e18, PROVIDER);
  market.approve(address(stEXA), 1_000e18);
  vm.stopPrank();

  market.pause();

  exa.mint(address(this), assets);

  vm.expectRevert();
  stEXA.deposit(assets, address(this));
}
```

### Mitigation

Check if the market is paused or frozen and do not harvest if this is the case.