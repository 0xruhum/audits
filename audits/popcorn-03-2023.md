# Popcorn Audit
- [H-01: MasterChefAdapter implements the wrong interface](#h-01-masterchefadapter-implements-the-wrong-interface)
  - [Vulnerability Details](#vulnerability-details)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps)
- [H-02: CompoundV2Adapter won't be able to claim any COMP rewards](#h-02-compoundv2adapter-wont-be-able-to-claim-any-comp-rewards)
  - [Vulnerability Details](#vulnerability-details-1)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-1)
- [H-03: Adapters only check for protocol rewards on initialization](#h-03-adapters-only-check-for-protocol-rewards-on-initialization)
  - [Vulnerability Details](#vulnerability-details-2)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-2)
- [H-04: MultiRewardStaking can't distribute reward tokens instantly](#h-04-multirewardstaking-cant-distribute-reward-tokens-instantly)
  - [Vulnerability Details](#vulnerability-details-3)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-3)
- [M-01: AaveV3Adapter doesn't return the correct list of reward tokens in `rewardTokens()`](#m-01-aavev3adapter-doesnt-return-the-correct-list-of-reward-tokens-in-rewardtokens)
  - [Vulnerability Details](#vulnerability-details-4)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-4)
- [M-02: Adapter will receive rewards from protocols without calling `claim()` which are not accounted for](#m-02-adapter-will-receive-rewards-from-protocols-without-calling-claim-which-are-not-accounted-for)
  - [Vulnerability Details](#vulnerability-details-5)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-5)
- [M-03: AaveV2Adapter checks the wrong value for reward token emissions](#m-03-aavev2adapter-checks-the-wrong-value-for-reward-token-emissions)
  - [Vulnerability Details](#vulnerability-details-6)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-6)
- [M-04: MultiRewardStaking accrual can round to 0 for reward tokens with fewer decimals](#m-04-multirewardstaking-accrual-can-round-to-0-for-reward-tokens-with-fewer-decimals)
  - [Vulnerability Details](#vulnerability-details-7)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-7)
- [M-05: MultiRewardStaking allows adding a reward token with escrow enabled but duration set to zero](#m-05-multirewardstaking-allows-adding-a-reward-token-with-escrow-enabled-but-duration-set-to-zero)
  - [Vulnerability Details](#vulnerability-details-8)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-8)
- [M-06: MultiRewardStaking's limit of 20 reward tokens could lead to issues](#m-06-multirewardstakings-limit-of-20-reward-tokens-could-lead-to-issues)
  - [Vulnerability Details](#vulnerability-details-9)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-9)
- [M-07: vault collects management fees while no assets are under management](#m-07-vault-collects-management-fees-while-no-assets-are-under-management)
  - [Vulnerability Details](#vulnerability-details-10)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-10)
- [M-08: YearnAdapter can't withdraw funds from Yearn if the vault has a loss exceeding 0.01%](#m-08-yearnadapter-cant-withdraw-funds-from-yearn-if-the-vault-has-a-loss-exceeding-001)
  - [Vulnerability Details](#vulnerability-details-11)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-11)
- [QA-01: misleading comments](#qa-01-misleading-comments)
- [QA-02: move external call success check into AdminProxy contract to reduce redundancy](#qa-02-move-external-call-success-check-into-adminproxy-contract-to-reduce-redundancy)


# Summary

**Project**: Popcorn

**Repo**: https://github.com/Popcorn-Limited/audit2

**Commit**: 298cf48f0369401bf21a43d796f9b2e27591297b

**Timeline**: 29/03/2023 - 07/04/2023

Out-of-scope contracts:
- src/vault/strategy/StrategyBase.sol
- src/vault/strategy/RewardsClaimer.sol
- src/vault/strategy/Pool2SingleAssetCompounder.sol
- src/vault/VaultRouter.sol
- src/utils/Owned.sol
- src/utils/OwnedUpgradeable.sol

**Issues found**:
| Severity | Count |
| -------- | ----- |
| High     | 4     |
| Medium   | 8     |
| QA       | 2     |

# High Risk

## H-01: MasterChefAdapter implements the wrong interface

The MasterChefAdapter meant as a wrapper for MasterChefV2 is using the interface for V1. It won't be usable as an adapter for MasterChefV2.

### Vulnerability Details

The interface used for the adapter is: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/sushi/MasterChefAdapter.sol#L25

```sol
interface IMasterChef {
  struct PoolInfo {
    address lpToken;
    uint256 allocPoint;
    uint256 lastRewardBlock;
    uint256 accSushiPerShare;
  }

  struct UserInfo {
    uint256 amount;
    uint256 rewardDebt;
  }

  function poolInfo(uint256 pid) external view returns (IMasterChef.PoolInfo memory);

  function userInfo(uint256 pid, address adapterAddress) external view returns (IMasterChef.UserInfo memory);

  function totalAllocPoint() external view returns (uint256);

  function deposit(uint256 _pid, uint256 _amount) external;

  function withdraw(uint256 _pid, uint256 _amount) external;

  function enterStaking(uint256 _amount) external;

  function leaveStaking(uint256 _amount) external;

  function emergencyWithdraw(uint256 _pid) external;

  function pendingSushi(uint256 _pid, address _user) external view returns (uint256);
}
```

That's the interface of MasterChefV1: https://github.com/sushiswap/sushiswap/blob/archieve/canary/contracts/MasterChef.sol


[MasterChefV2](https://github.com/sushiswap/sushiswap/blob/archieve/canary/contracts/MasterChefV2.sol) has a different interface for depositing, withdrawing, and claiming rewards: 
- depositing:
  - V1: `function deposit(uint256 _pid, uint256 _amount) external;`
  - V2: `function deposit(uint256 pid, uint256 amount, address to) external;`
- withdrawing:
  - V1: `function withdraw(uint256 _pid, uint256 _amount) external;`
  - V1: `function withdraw(uint256 pid, uint256 amount, address to) external;`
- claiming:
  - V1 claims automatically with every deposit & withdrawal
  - V2 uses a dedicated claim function: `function harvest(uint256 pid, address to) external;`

### Recommended Mitigation Steps

Use the MasterChefV2 interface.

## H-02: CompoundV2Adapter won't be able to claim any COMP rewards

The CompoundV2Adapter won't claim any COMP rewards because it uses the wrong function to check whether the given cToken is eligible for COMP rewards.

### Vulnerability Details

When a CompoundV2Adapter is initialized, it checks through the Comptroller whether the given asset is eligble for COMP rewards. For that, it uses the `compSpeeds()` function: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/compound/compoundV2/CompoundV2Adapter.sol#L76 

```sol
    uint256 compSpeed = comptroller.compSpeeds(address(cToken));
    isActiveCompRewards = compSpeed > 0 ? true : false;
```

But, that function will always return 0. For some reason, Comptroller never assigns a value to it but exposes the variable to other contracts. You can verify it here:
- https://etherscan.io/address/0x3d9819210a31b4961b30ef54be2aed79b9c9cd3b#readProxyContract#F19 with inputs:
    - cETH: 0x4Ddc2D193948926D02f9B1fE9e1daa0718270ED5
    - cDAI: 0x5d3a536E4D6DbD6114cc1Ead35777bAB948E3643
    - cUSDC: 0x39AA39c021dfbaE8faC545936693aC917d5E7563

You can also go through the current implementation contract and search for `compSpeeds`. There's no code where it assigns any value to that state variable: https://etherscan.io/address/0xbafe01ff935c7305907c33bf824352ee5979b526/advanced#code

Instead, comp speeds are assigned through the `setCompSpeedInternal()` function using `compSupplySpeeds` and `compBorrowSpeeds`. Since the adapter is only providing assets to the vault, you only need to check `compSupplySpeeds`. For cDAI and cUSDC you get a value >0: https://etherscan.io/address/0x3d9819210a31b4961b30ef54be2aed79b9c9cd3b#readProxyContract#F21

### Recommended Mitigation Steps

Use `compSupplySpeeds()` to check whether the given cToken is eligible for COMP rewards:

```sol
    uint256 compSpeed = comptroller.compSupplySpeeds(address(cToken));
    isActiveCompRewards = compSpeed > 0 ? true : false;
```

## H-03: Adapters only check for protocol rewards on initialization

Some protocols distribute reward tokens. Adapters for these protocols expose a `claim()` function to allow a strategy to harvest and distribute those reards. When the adapter is initialized it checks whether it's eligible for any rewards. If not, it sets a flag so that calls to `claim()` revert. But, reward distribution by most protocols is dynamic. While there might not be any at the time of initialization, it could change in the future. But, at that point, the adapter won't be able to harvest those tokens causing it to lose funds.

### Vulnerability Details

For example, in CompoundV2Adapter the adapter checks whether any COMP is awarded for the given asset. If not, it sets `isActiveCompRewards` to `false`:  https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/compound/compoundV2/CompoundV2Adapter.sol#LL54-L78C4

```sol
  function initialize(
    bytes memory adapterInitData,
    address comptroller_,
    bytes memory compoundV2InitData
  ) external initializer {
    // ...

    uint256 compSpeed = comptroller.compSpeeds(address(cToken));
    isActiveCompRewards = compSpeed > 0 ? true : false;
  }
```

That causes `claim()` to revert preventing the adapter from harvesting any reward tokens: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/compound/compoundV2/CompoundV2Adapter.sol#LL136-L139C4

```sol
  function claim() public override onlyStrategy {
    if (isActiveCompRewards == false) revert IncentivesNotActive();
    comptroller.claimComp(address(this));
  }
```

The Comptroller is able to change the reward rate at any point through the `setCompSpeeds()` function: https://etherscan.io/address/0x3d9819210A31b4961b30EF54bE2aeD79B9c9Cd3B#writeProxyContract#F7

### Recommended Mitigation Steps

`claim()` should always try to harvest the tokens. Also, it shouldn't revert at any point. That can result in the whole adapter breaking because `claim()` is part of the main happy path. With every deposit & withdrawal, the base adapter calls `harvest()` which will trigger `claim()`. If that is not wrapped in a `try-catch` block, the whole function will revert.

## H-04: MultiRewardStaking can't distribute reward tokens instantly

The MultiRewardStaking contract is not able to distribute reward tokens instantly, i.e. `rewardsPerSecond == 0`.

### Vulnerability Details

As described in the [`addRewardToken()`](https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardStaking.sol#L232) function, the MultiRewardStaking contract is supposed to be able to distribute reward tokens instantly instead of releasing them linearly:

```sol
  /**
   * @notice Adds a new rewardToken which can be earned via staking. Caller must be owner.
   * @param rewardToken Token that can be earned by staking.
   * @param rewardsPerSecond The rate in which `rewardToken` will be accrued.
   * @param amount Initial funding amount for this reward.
   * @param useEscrow Bool if the rewards should be escrowed on claim.
   * @param escrowPercentage The percentage of the reward that gets escrowed in 1e18. (1e18 = 100%, 1e14 = 1 BPS)
   * @param escrowDuration The duration of the escrow.
   * @param offset A cliff after claim before the escrow starts.
   * @dev The `rewardsEndTimestamp` gets calculated based on `rewardsPerSecond` and `amount`.
   * @dev If `rewardsPerSecond` is 0 the rewards will be paid out instantly. In this case `amount` must be 0.
   * @dev If `useEscrow` is `false` the `escrowDuration`, `escrowPercentage` and `offset` will be ignored.
   * @dev The max amount of rewardTokens is 20.
   */
  function addRewardToken(
    IERC20 rewardToken,
    uint160 rewardsPerSecond,
    uint256 amount,
    bool useEscrow,
    uint192 escrowPercentage,
    uint32 escrowDuration,
    uint32 offset
  ) external onlyOwner {
```

But, the accrual logic doesn't support instant distribution, i.e. `rewardsPerSecond == 0`. Here's a simple test to showcase the issue:

```sol
// MultiRewardStaking.t.sol
  function testInstantReward() public {
    stakingToken.mint(alice, 1e18);

    vm.startPrank(alice);
    stakingToken.approve(address(staking), 1e18);
    staking.deposit(1e18, alice);
    vm.stopPrank();

    // Prepare to transfer reward tokens
    rewardToken1.mint(address(this), 100e18);
    rewardToken1.approve(address(staking), 100e18);

    staking.addRewardToken(iRewardToken1, 0, 100e18, false, 0, 0, 0);

    IERC20[] memory rewardTokens = staking.getAllRewardsTokens();
    vm.prank(alice);
    staking.claimRewards(alice, rewardTokens);
  }
```

Output:
```
Running 1 test for test/MultiRewardStaking.t.sol:MultiRewardStakingTest
[FAIL. Reason: ZeroRewards(0x2e234DAe75C793f67A35089C9d99245E1C58470b)] testInstantReward() (gas: 412444)
Test result: FAILED. 0 passed; 1 failed; finished in 2.02ms
```

The issue is that the accrual logic used to distribute rewards can't handle the instant distribution model properly. 
When a reward token is added, it sets `rewards.index == ONE`: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardStaking.sol#L274

```sol
    rewardInfos[rewardToken] = RewardInfo({
      ONE: ONE,
      rewardsPerSecond: rewardsPerSecond, // == 0
      rewardsEndTimestamp: rewardsEndTimestamp, == block.timestamp
      index: ONE,
      lastUpdatedTimestamp: block.timestamp.safeCastTo32()
    });
```

When a user tries to claim their rewards through `claimRewards()`, it first accrues the rewards for the user through the `accrueRewards()` modifier: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardStaking.sol#L162

```sol
  function claimRewards(address user, IERC20[] memory _rewardTokens) external accrueRewards(msg.sender, user) {
    for (uint8 i; i < _rewardTokens.length; i++) {
      uint256 rewardAmount = accruedRewards[user][_rewardTokens[i]];

      if (rewardAmount == 0) revert ZeroRewards(_rewardTokens[i]);
```

`accrueRewards()` will then only execute `_accrueUser()` for the given reward token:

```sol
  modifier accrueRewards(address _caller, address _receiver) {
    IERC20[] memory _rewardTokens = rewardTokens;
    for (uint8 i; i < _rewardTokens.length; i++) {
      IERC20 rewardToken = _rewardTokens[i];
      RewardInfo memory rewards = rewardInfos[rewardToken];

      if (rewards.rewardsPerSecond > 0) _accrueRewards(rewardToken, _accrueStatic(rewards));
      _accrueUser(_receiver, rewardToken);

      // If a deposit/withdraw operation gets called for another user we should accrue for both of them to avoid potential issues like in the Convex-Vulnerability
      if (_receiver != _caller) _accrueUser(_caller, rewardToken);
    }
    _;
  }

  /// @notice Sync a user's rewards for a rewardToken with the global reward index for that token
  function _accrueUser(address _user, IERC20 _rewardToken) internal {
    RewardInfo memory rewards = rewardInfos[_rewardToken];

    uint256 oldIndex = userIndex[_user][_rewardToken];

    // If user hasn't yet accrued rewards, grant them interest from the strategy beginning if they have a balance
    // Zero balances will have no effect other than syncing to global index
    if (oldIndex == 0) {
      oldIndex = rewards.ONE;
    }

    // @audit for rewardsPerSecond == 0, this is ONE - ONE = 0
    uint256 deltaIndex = rewards.index - oldIndex;

    // Accumulate rewards by multiplying user tokens by rewardsPerToken index and adding on unclaimed
    uint256 supplierDelta = balanceOf(_user).mulDiv(deltaIndex, uint256(10 ** decimals()), Math.Rounding.Down);
    // stakeDecimals  * rewardDecimals / stakeDecimals = rewardDecimals
    // 1e18 * 1e6 / 10e18 = 0.1e18 | 1e6 * 1e18 / 10e18 = 0.1e6

    userIndex[_user][_rewardToken] = rewards.index;

    accruedRewards[_user][_rewardToken] += supplierDelta;
  }
```

Because `rewards.index` was never updated, it will use the initial value, `ONE`. The same value as `oldIndex`. That cause s `deltaIndex` to be 0, which in turn causes `supplierDelta` to be 0. Thus, `accruedRewards` is not increased and the call to `claimRewards()` fails.

Funds sent to `addRewardToken()` with `rewardsPerSecond == 0` will be locked up.

### Recommended Mitigation Steps

In `addRewardToken()`, if `rewardsPerSecond == 0` the `index` property should be set to:

`ONE + amount.mulDiv(uint256(10 ** decimals()), supplyTokens, Math.Rounding.Down).safeCastTo224();`

That way, all the tokens are distributed to existing shareholders.

# Medium Risk

## M-01: AaveV3Adapter doesn't return the correct list of reward tokens in `rewardTokens()`

Instead of returning the reward tokens for the adapter's asset, it returns **all** the reward tokens of the AAVE reward contract.

### Vulnerability Details

`rewardTokens()` returns all the tokens registered in the AAVE reward contract: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/aave/aaveV3/AaveV3Adapter.sol#L101

```sol
  function rewardTokens() external view override returns (address[] memory) {
    return aaveIncentives.getRewardsList();
  }
```

But, the reward tokens depend on the underlying asset of the adapter. To get the correct list, you have to call `getRewardsByAsset()`: https://github.com/aave/aave-v3-periphery/blob/master/contracts/rewards/RewardsDistributor.sol#LL90-L104C4

```sol
  /// @inheritdoc IRewardsDistributor
  function getRewardsByAsset(address asset) external view override returns (address[] memory) {
    uint128 rewardsCount = _assets[asset].availableRewardsCount;
    address[] memory availableRewards = new address[](rewardsCount);

    for (uint128 i = 0; i < rewardsCount; i++) {
      availableRewards[i] = _assets[asset].availableRewards[i];
    }
    return availableRewards;
  }

  /// @inheritdoc IRewardsDistributor
  function getRewardsList() external view override returns (address[] memory) {
    return _rewardsList;
  }
```

### Recommended Mitigation Steps

Change `rewardTokens()` to:

```sol
  function rewardTokens() external view override returns (address[] memory) {
    return aaveIncentives.getRewardsByAsset(asset());
  }
```

## M-02: Adapter will receive rewards from protocols without calling `claim()` which are not accounted for

The adapter earns rewards from the protocol it deposits into. The amount of rewards depends on the deposit amount. Generally, when you withdraw funds, the protocol will update your reward balance **before** executing the withdrawal. In some cases, e.g. Compound, it will send the accrued rewards to the caller's address.

When the adapter withdraws funds from the underlying protocol it doesn't harvest any rewards before that. Any funds that are sent to the adapter by the protocol through the withdrawal won't be accounted for.

### Vulnerability Details

When an adapter is paused it withdraws all of its funds without harvesting reward tokens: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/abstracts/AdapterBase.sol#L363

```sol
  function pause() external onlyOwner {
    _protocolWithdraw(totalAssets(), totalSupply());
    _pause();
  }
```

ERC4626 withdrawals will also execute `harvest()` **after** withdrawing from the underlying protocol:https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/abstracts/AdapterBase.sol#L134-L142

```sol
  function _withdraw(
      address caller,
      address receiver,
      address owner,
      uint256 assets,
      uint256 shares
  ) internal virtual override {
      if (caller != owner) {
          _spendAllowance(owner, caller, shares);
      }

      if (!paused()) {
          _protocolWithdraw(assets, shares);
      }

      _burn(owner, shares);

      IERC20(asset()).safeTransfer(receiver, assets);

      harvest();

      emit Withdraw(caller, receiver, owner, assets, shares);
  }
```


Some protocols will send reward tokens before executing the withdrawal, e.g. Compound. When the user redeems their cTokens, it will execute `redeemAllowed()` which will distribute supplier COMP rewards: https://etherscan.io/address/0xbafe01ff935c7305907c33bf824352ee5979b526#code#F4#L279

```sol
    function redeemAllowed(address cToken, address redeemer, uint redeemTokens) external returns (uint) {
        uint allowed = redeemAllowedInternal(cToken, redeemer, redeemTokens);
        if (allowed != uint(Error.NO_ERROR)) {
            return allowed;
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, redeemer);

        return uint(Error.NO_ERROR);
    }
```

These protocols send rewards on withdrawal:
- Compound V2
- MasterChef V1
- Convex (depends on flag passed to withdrawal function)

The other protocols just accrue the current rewards to the user and then update their balance. 

Thus, the strategy contract has to be able to handle the scenario where the adapter receives reward tokens without `claim()` being executed.

### Recommended Mitigation Steps

That means, harvesting with every withdrawal is unnecessary. You can just use an EOA to claim at any point you want. As long as the strategy contract properly handles that. 

## M-03: AaveV2Adapter checks the wrong value for reward token emissions

In the initialize function the adapter checks whether the given asset is eligible for reward tokens. But, it uses the wrong value to check which will cause it to always evaluate to `true` even if there are no emissions for the asset.

### Vulnerability Details

It calls the `assets` state variable's getter function to check the emission rate for the given asset: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/aave/aaveV2/AaveV2Adapter.sol#L77

```sol
    uint128 emission;
    if (address(aaveMining) != address(0)) {
      (, emission, ) = aaveMining.assets(asset());
    }

    isActiveMining = emission > 0 ? true : false;
```

But, for `assets()` the first return value is the emission rate and the second value is the asset's index:
https://etherscan.io/address/0xd784927Ff2f95ba542BfC824c8a8a98F3495f6b5#readProxyContract#F7

```sol
  struct AssetData {
    uint104 emissionPerSecond;
    uint104 index;
    uint40 lastUpdateTimestamp;
    mapping(address => uint256) users;
  }
```

Since the `index` won't be 0, the `isActiveMining` state variable will always evaluate to `true`.

### Recommended Mitigation Steps

Change to:

```sol
    uint104 emission;
    if (address(aaveMining) != address(0)) {
      (emission,,) = aaveMining.assets(asset());
    }

    isActiveMining = emission > 0 ? true : false;
```

## M-04: MultiRewardStaking accrual can round to 0 for reward tokens with fewer decimals

The MultiRewardStaking contract doesn't properly handle the accrued amount being rounded down to 0. That will cause a loss of funds.

### Vulnerability Details

The lowest amount that can be accrued for any given reward token is `rewardsPerSecond * 12`, because every 12 seconds a new block is processed. If the accrued amount distributed to all the shareholders (`rewardsPerSecond * 12 / totalSupply`) rounds down to 0, those tokens won't be accounted for.

This is generally an issue for reward tokens with fewer decimals, e.g. USDC.

The accrual logic is simple: `rewardsPerSecond * elapsedTime / totalSupply` is allocated to every share of the MultiRewardStaking contract: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardStaking.sol#LL380-L402C4

```sol
  function _accrueStatic(RewardInfo memory rewards) internal view returns (uint256 accrued) {
    uint256 elapsed;
    if (rewards.rewardsEndTimestamp > block.timestamp) {
      elapsed = block.timestamp - rewards.lastUpdatedTimestamp;
    } else if (rewards.rewardsEndTimestamp > rewards.lastUpdatedTimestamp) {
      elapsed = rewards.rewardsEndTimestamp - rewards.lastUpdatedTimestamp;
    }

    accrued = uint256(rewards.rewardsPerSecond * elapsed);
  }

  /// @notice Accrue global rewards for a rewardToken
  function _accrueRewards(IERC20 _rewardToken, uint256 accrued) internal {
    uint256 supplyTokens = totalSupply();
    uint224 deltaIndex; // DeltaIndex is the amount of rewardsToken paid out per stakeToken
    if (supplyTokens != 0)
      deltaIndex = accrued.mulDiv(uint256(10 ** decimals()), supplyTokens, Math.Rounding.Down).safeCastTo224();
    // rewardDecimals * stakeDecimals / stakeDecimals = rewardDecimals
    // 1e18 * 1e6 / 10e6 = 0.1e18 | 1e6 * 1e18 / 10e18 = 0.1e6

    rewardInfos[_rewardToken].index += deltaIndex;
    rewardInfos[_rewardToken].lastUpdatedTimestamp = block.timestamp.safeCastTo32();
  }
```

Given that the contract is funded with USDC, a token with only 6 decimals, where we want to distribute 10,000 USDC over a month:
- amount = 10,000e6
- rewardsPerSecond = 10,000e6 / (30* 24 * 60 * 60) = 3858
- totalSupply = 100,000e18

Then, the minimum amount of accrued tokens is: `3858 * 12 = 46296`. `deltaIndex` is the number of reward tokens allocated for each share of the staking contract:
`deltaIndex = accrued.mulDiv(uint256(10 ** decimals()), supplyTokens, Math.Rounding.Down).safeCastTo224();`

`46296 * 1e18 / 100,000e18 = 0.46296 = 0`

So by accruing the rewards for USDC with every block we can prevent any rewards from being paid out to shareholders.

Here's a simple test showcasing the issue:
```sol
  // MultiRewardStaking.t.sol
  function testRounding() public {
    MockERC20 rewardToken3 = new MockERC20("RewardsToken3", "RTKN2", 6);
    IERC20 iRewardToken3 = IERC20(address(rewardToken3));
    stakingToken.mint(alice, 100_000e18);

    vm.startPrank(alice);
    // Just for the test, alice holds all the supply.
    // In a normal envirenmont this would be split across hundreds of addresses
    stakingToken.approve(address(staking), 100_000e18);
    staking.deposit(100_000e18, alice);
    vm.stopPrank();

    // Prepare to transfer reward tokens
    rewardToken3.mint(address(this), 10_000e6);
    rewardToken3.approve(address(staking), 10_000e6);

    staking.addRewardToken(iRewardToken3, 3858, 10_000e6, false, 0, 0, 0);

    // next block
    vm.warp(block.timestamp + 12);

    IERC20[] memory rewardTokens = staking.getAllRewardsTokens();
    vm.prank(alice);

    // will fail because rewards were rounded down to 0
    staking.claimRewards(alice, rewardTokens);
  }

```

The same thing applies to accruing each user's rewards. To calculate the user's rewards, it uses `shareBalance * deltaIndex / 1e18`. That means if `shareBalance * deltaIndex < 1e18` the user will receive 0 rewards while updating their index. 

### Recommended Mitigation Steps

To prevent this from happening, `_accrueRewards()` and `_accrueUser()` have to not update the reward state if their respective values round down to 0. You **can't** check for this when a new reward token is added or its distribution configuration is changed. `totalSupply` is dynamic so even if a configuration works at first, it might become an issue after more people deposited their funds.

A potential fix would be:

```sol
  function _accrueStatic(RewardInfo memory rewards) internal view returns (uint256 accrued) {
    uint256 elapsed;
    if (rewards.rewardsEndTimestamp > block.timestamp) {
      elapsed = block.timestamp - rewards.lastUpdatedTimestamp;
    } else if (rewards.rewardsEndTimestamp > rewards.lastUpdatedTimestamp) {
      elapsed = rewards.rewardsEndTimestamp - rewards.lastUpdatedTimestamp;
    }

    accrued = uint256(rewards.rewardsPerSecond * elapsed);
  }

  /// @notice Accrue global rewards for a rewardToken
  function _accrueRewards(IERC20 _rewardToken, uint256 accrued) internal {
    uint256 supplyTokens = totalSupply();
    uint224 deltaIndex; // DeltaIndex is the amount of rewardsToken paid out per stakeToken
    if (supplyTokens != 0)
      deltaIndex = accrued.mulDiv(uint256(10 ** decimals()), supplyTokens, Math.Rounding.Down).safeCastTo224();
    // rewardDecimals * stakeDecimals / stakeDecimals = rewardDecimals
    // 1e18 * 1e6 / 10e6 = 0.1e18 | 1e6 * 1e18 / 10e18 = 0.1e6

    // skip accrual if it rounds down to 0
    if (deltaIndex == 0) {
      return;
    }

    rewardInfos[_rewardToken].index += deltaIndex;
    rewardInfos[_rewardToken].lastUpdatedTimestamp = block.timestamp.safeCastTo32();
  }

  function _accrueUser(address _user, IERC20 _rewardToken) internal {
    RewardInfo memory rewards = rewardInfos[_rewardToken];

    uint256 oldIndex = userIndex[_user][_rewardToken];

    // If user hasn't yet accrued rewards, grant them interest from the strategy beginning if they have a balance
    // Zero balances will have no effect other than syncing to global index
    if (oldIndex == 0) {
      oldIndex = rewards.ONE;
    }

    uint256 deltaIndex = rewards.index - oldIndex;

    // Accumulate rewards by multiplying user tokens by rewardsPerToken index and adding on unclaimed
    uint256 supplierDelta = balanceOf(_user).mulDiv(deltaIndex, uint256(10 ** decimals()), Math.Rounding.Down);
    // stakeDecimals  * rewardDecimals / stakeDecimals = rewardDecimals
    // 1e18 * 1e6 / 10e18 = 0.1e18 | 1e6 * 1e18 / 10e18 = 0.1e6

    if (supplierDelta == 0) {
      return;
    }

    userIndex[_user][_rewardToken] = rewards.index;

    accruedRewards[_user][_rewardToken] += supplierDelta;
  }
```

## M-05: MultiRewardStaking allows adding a reward token with escrow enabled but duration set to zero

When adding a reward token to the MultiRewardStaking contract you're able to enable the escrow feature while setting the escrow duration to zero. That will result in nobody being able to claim their rewards, effectively locking up the reward tokens.

This will also result in the reward token not being usable anymore for the given staking contract.

### Vulnerability Details

In `MultiRewardEscrow.lock()` there's a check that `duration != 0`: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardEscrow.sol#L99

```sol
  function lock(
    IERC20 token,
    address account,
    uint256 amount,
    uint32 duration,
    uint32 offset
  ) external {
    if (token == IERC20(address(0))) revert ZeroAddress();
    if (account == address(0)) revert ZeroAddress();
    if (amount == 0) revert ZeroAmount();
    if (duration == 0) revert ZeroAmount();
```

But, in `MultiRewardStaking.addRewardToken()` the owner can set the escrow duration to 0. That small error will lock up the provided reward tokens because the call to `lock()` in `claimRewards()` will revert. That will also result in the reward token not being usable anymore since you can't update the property.

Here's a small test to showcase the issue:

```sol
  function testEscrowDurationZero() public {
    stakingToken.mint(alice, 1e18);

    vm.startPrank(alice);
    stakingToken.approve(address(staking), 1e18);
    staking.deposit(1e18, alice);
    vm.stopPrank();

    // Prepare to transfer reward tokens
    rewardToken1.mint(address(this), 100e18);
    rewardToken1.approve(address(staking), 100e18);

    // escrow duration set to 0
    staking.addRewardToken(iRewardToken1, 1e18, 100e18, true, 1e17, 0, 0);

    vm.warp(block.timestamp + 10);

    IERC20[] memory rewardTokens = staking.getAllRewardsTokens();
    vm.prank(alice);
    staking.claimRewards(alice, rewardTokens);
  }
```

Output:

```
    ├─ [66890] MultiRewardStaking::claimRewards(alice: [0x000000000000000000000000000000000000ABcD], [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   ├─ [25012] MockERC20::transfer(alice: [0x000000000000000000000000000000000000ABcD], 9000000000000000000)
    │   │   ├─ emit Transfer(from: MultiRewardStaking: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: alice: [0x000000000000000000000000000000000000ABcD], value: 9000000000000000000)
    │   │   └─ ← true
    │   ├─ [797] MultiRewardEscrow::lock(MockERC20: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], alice: [0x000000000000000000000000000000000000ABcD], 1000000000000000000, 0, 0)
    │   │   └─ ← "ZeroAmount()"
    │   └─ ← "ZeroAmount()"
    └─ ← "ZeroAmount()"
```

### Recommended Mitigation Steps

In `addRewardToken()` add a check to the escrow if-clause to prevent the duration from being 0:

```sol
    if (useEscrow) {
      if (escrowPercentage == 0 || escrowPercentage > 1e18 || escrowDuration == 0) revert InvalidConfig();

      escrowInfos[rewardToken] = EscrowInfo({
        escrowPercentage: escrowPercentage,
        escrowDuration: escrowDuration,
        offset: offset
      });
```

## M-06: MultiRewardStaking's limit of 20 reward tokens could lead to issues

There's a hard limit of 20 reward tokens a staking contract can use. There's no way for a vault to change its staking contract. Thus, that limit also applies to a vault. To allow vaults to be future-proof, the owner should be able to add new reward tokens by removing old ones.

### Vulnerability Details

The cap on reward tokens was implemented to prevent the contract from running out of gas when looping over all the tokens. But, by not allowing the owner to remove existing tokens (after the distribution is finished) you limit the staking contract to only 20 tokens over its entire lifetime. 

The `addRewardToken()` function only allows up to 20 tokens: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/utils/MultiRewardStaking.sol#L241

### Recommended Mitigation Steps

Add a function to remove reward tokens that have already been distributed.

## M-07: vault collects management fees while no assets are under management

> This issue was first reported in the [Code4rena audit](https://code4rena.com/contests/2023-01-popcorn-contest) by [ast3ros](https://github.com/code-423n4/2023-01-popcorn-findings/issues/499).

The vault starts collecting management fees from the time it is initialized. At that point, there are no assets under management.

### Vulnerability Details

Management fees are determined using `accruedManagementFee()`: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/Vault.sol#L375
```sol
  function accruedManagementFee() public view returns (uint256) {
    uint256 managementFee = fees.management;
    return
      managementFee > 0
        ? managementFee.mulDiv(
          totalAssets() * (block.timestamp - feesUpdatedAt),
          SECONDS_PER_YEAR,
          Math.Rounding.Down
        ) / 1e18
        : 0;
  }
```

`feesUpdatedAt` is set to `block.timestamp` when the vault is initialized: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/Vault.sol#L90

Thus, the vault starts collecting management fees from the time it is initialized and not the time at which it receives the first deposit.

### Recommended Mitigation Steps

Initialize `feesUpdatedAt` with the first deposit.

## M-08: YearnAdapter can't withdraw funds from Yearn if the vault has a loss exceeding 0.01%

> This issue was first reported in the [Code4rena audit](https://code4rena.com/contests/2023-01-popcorn-contest) by [rbserver](https://github.com/code-423n4/2023-01-popcorn-findings/issues/581).

If the underlying Yearn vault experiences a loss of funds so that 1 share is not equal to or worth at least 1 asset, the adapter will fail to withdraw funds from the vault. That will lock up all the user funds inside the adapter.

### Vulnerability Details

Yearn vault's `withdraw()` function has a `maxLoss` parameter to specify "the maximum acceptable loss to sustain on withdrawal. Defaults to 0.01%.": https://github.com/yearn/yearn-vaults/blob/master/contracts/Vault.vy#L1074

The YearnAdapter doesn't specify a value for that parameter so the default value is used. Thus, the maximum loss that is accepted is 0.01%: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/yearn/YearnAdapter.sol#L141

```sol
  function _protocolWithdraw(uint256 assets, uint256 shares) internal virtual override {
    yVault.withdraw(convertToUnderlyingShares(assets, shares));
  }
```

Thus, if the vault experiences losses, user funds inside the adapter won't be withdrawable anymore.

### Recommended Mitigation Steps

The user should be able to specify their own value for `maxLoss` when withdrawing from the adapter.

# QA

## QA-01: misleading comments

There are multiple functions with misleading comments:
- `DeploymentController.addTemplate()` is callable by anyone, not just the owner: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/DeploymentController.sol#L60 
- Only the owner of VaultController can pause & unpause adapters: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/VaultController.sol#L683, https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/VaultController.sol#L695
- Should link to the Convex contract instead of Beefy: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/convex/ConvexAdapter.sol#L15
- Should say Convex and not AAVE: https://github.com/Popcorn-Limited/audit2/blob/298cf48f0369401bf21a43d796f9b2e27591297b/src/vault/adapter/convex/ConvexAdapter.sol#L87

## QA-02: move external call success check into AdminProxy contract to reduce redundancy

Throughout the VaultController contract you have multiple instances where you call the `AdminProxy.execute()`. That function
executes a low-level `call`. Instead of checking whether the call was successful or not in the VaultController that check should be moved into AdminProxy. There are 28 instances of that:

```sol
(bool success, bytes memory returnData) = adminProxy.execute();
if (!success) revert UnderlyingError(returnData);
```

Search for "adminProxy.execute" to find all the instances.