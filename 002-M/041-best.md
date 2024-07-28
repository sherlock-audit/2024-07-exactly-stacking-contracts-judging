Large Fuchsia Dove

Medium

# Setting a new market will make depositing to the market impossible when harvesting, DoSing deposits

### Summary

In `StakedEXA::harvest()`, it deposits to the `market` the resulting savings that are not notified as rewards from the provider. The approval of the asset of the `Market` to the `Market` is only performed in the constructor, which means that if a new `Market` is set via `StakedEXA::setMarket()`, it currently does not approve it and will DoS deposits and make it impossible to correctly switch markets.

### Root Cause

In `StakedEXA.sol::460`, `setMarket()`, a new market is [set](https://github.com/sherlock-audit/2024-07-exactly-stacking-contracts/blob/main/protocol/contracts/StakedEXA.sol#L460) but an approval is not given of the `market.asset()` to the `market`, which will DoS deposits.

### Internal pre-conditions

1. A new market is set.

### External pre-conditions

None.

### Attack Path

1. `StakedEXA::setMarket()` is called, setting a new market.
2. Deposits are DoSed because it tries to deposit to the new market without approving it first and reverts.

### Impact

DoSed deposits and impossability of correctly changing markets.

### PoC

```solidity
function test_POC_DoSedDeposits_DueToSettingANewMarket() external {
  bool triggerRevert = true;

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

  // FIX market change
  if (!triggerRevert) {
    vm.prank(address(stEXA));
    providerAsset.approve(address(market), type(uint256).max);
  }

  stEXA.setMarket(market);

  providerAsset.mint(PROVIDER, 1_000e18);

  vm.startPrank(PROVIDER);
  providerAsset.approve(address(market), type(uint256).max);
  market.deposit(1_000e18, PROVIDER);
  market.approve(address(stEXA), 1_000e18);
  vm.stopPrank();

  exa.mint(address(this), assets);

  if (triggerRevert) vm.expectRevert();
  stEXA.deposit(assets, address(this));
}
```

### Mitigation

Do the same as in the constructor and approve the new `Market` when `setMarket()` is called with `type(uint256).max` `market.asset()` assets.