- [M-01: Keeper incurs losses on price movement between asset and yield asset](#m-01-keeper-incurs-losses-on-price-movement-between-asset-and-yield-asset)
  - [Vulnerability Details](#vulnerability-details)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps)
- [M-02: Pausing the strategy should prevent the keeper from pushing assets](#m-02-pausing-the-strategy-should-prevent-the-keeper-from-pushing-assets)
  - [Vulnerability Details](#vulnerability-details-1)
  - [Recommended Mitigation Steps](#recommended-mitigation-steps-1)
- [L-01: AnyConverter's slippage parameter is pointless](#l-01-anyconverters-slippage-parameter-is-pointless)
- [L-02: Remove default implementation for virtual functions](#l-02-remove-default-implementation-for-virtual-functions)
- [Q-01: `AnyConverter.claimReserved()` emits event even if nothing was claimed](#q-01-anyconverterclaimreserved-emits-event-even-if-nothing-was-claimed)
- [Q-02: `AnyConverter.ReserveClaimed` event should include the original withdrawable and the actual withdrawn amount](#q-02-anyconverterreserveclaimed-event-should-include-the-original-withdrawable-and-the-actual-withdrawn-amount)
- [Q-03: Remove out-of-date TODOs](#q-03-remove-out-of-date-todos)
- [Q-04: Remove unused code](#q-04-remove-unused-code)


# Summary

**Project**: Popcorn AnyToAny

**Repo**: https://github.com/Popcorn-Limited/contracts/tree/audit/anyToAny

**Commit**: b226a7978b1b746cc4e312c448a866b70362c1e4

**Timeline**: 16/08/2024 - 18/08/2024

Scope:
- src/strategies/AnyConverter.sol
- src/strategies/AnyCompounderNaive.sol
- src/strategies/AnyDepositor.sol
- src/strategies/BaseStrategy.sol

**Issues found**:
| Severity | Count |
| -------- | ----- |
| High     | 0     |
| Medium   | 2     |
| Low      | 2     |
| QA       | 4     |

# Medium Risk

## M-01: Keeper incurs losses on price movement between asset and yield asset

Price movement between asset and yield asset could cause the keeper to incur unexpected losses..

### Vulnerability Details

After a keeper reserves funds, they can claim at any time $X$ as long as $X >= unlockTime$. At most, the keeper can claim `Reserved.withdrawable` from the strategy. In case the price decreases between reserving and claiming, the keeper will get the worse quote:

https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L284
```sol
    function claimReserved(uint256 blockNumber, bool isYieldAsset) external {
        address base = isYieldAsset ? asset() : yieldAsset;
        address quote = isYieldAsset ? yieldAsset : asset();

        Reserved memory _reserved = reserved[msg.sender][base][blockNumber];
        if (_reserved.unlockTime != 0 && _reserved.unlockTime < block.timestamp) {
            // if the assets value went down after the keeper reserved the funds,
            // we want to use the new favorable quote.
            // If the assets value went up, we want to use the old favorable quote.
            uint256 withdrawable = Math.min(oracle.getQuote(_reserved.deposited, base, quote), _reserved.withdrawable);

        // ...
    }
```
They don't benefit from the price increase but lose funds if it decreases.
Thus, it's in the keeper's best interest to claim at a time when the price is closest to the one
at the time at which they reserved the funds.

As a keeper, you lose either way if the price moves:
- price of deposited asset $X$ increases => you receive same amount of $Y$ as before
- price of deposited asset $X$ decreases => you receive less $Y$ than before
- price of deposited asset $X$ doesn't change => you receive same amount of $Y$ as before

Having the keeper take on too many losses could cause the keeper to stop operating the strategy. 
That would mean that funds won't move in either direction causing the user's funds to be locked up 
until a new keeper takes up the operation.

### Recommended Mitigation Steps

The keeper should receive the same amount as originally calculated at the time of reserving the funds. 

**Popcorn**: Acknowledged. Preventing the keeper from withdrawing more funds than allowed through oracle manipulation is of higher priority.

## M-02: Pausing the strategy should prevent the keeper from pushing assets

Pausing the strategy prevents users from depositing additional assets. Only withdrawals are allowed.
Allowing the keeper to push funds (trade yield assets for underlying assets) would prevent users from withdrawing their funds.

### Vulnerability Details

Pausing the strategy doesn't prevent the keeper from pushing funds:

https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L109
```sol
function pushFunds(uint256 yieldAssets, bytes memory) external override onlyKeeperOrOwner {
    // caching
    address _asset = asset();
    address _yieldAsset = yieldAsset;
    uint256 _floatRatio = floatRatio;

    uint256 ta = totalAssets();
    // TODO: should take into account the reserved assets
    uint256 bal = IERC20(_asset).balanceOf(address(this)) - totalReservedAssets;

    IERC20(_yieldAsset).safeTransferFrom(msg.sender, address(this), yieldAssets);

    uint256 withdrawable = oracle.getQuote(yieldAssets, _yieldAsset, _asset);

    // we revert if:
    // 1. we don't have enough funds to cover the withdrawable amount
    // 2. we don't have enough float after the withdrawal (if floatRatio > 0)
    if (_floatRatio > 0) {
        uint256 float = ta.mulDiv(_floatRatio, 10_000, Math.Rounding.Floor);
        if (float > bal) {
            revert NotEnoughFloat();
        } else {
            uint256 balAfterFloat = bal - float;
            if (balAfterFloat < withdrawable) revert BalanceTooLow();
        }
    } else {
        if (bal < withdrawable) revert BalanceTooLow();
    }

    _reserveToken(yieldAssets, withdrawable, _yieldAsset, false);
    uint256 postTa = totalAssets();

    if (postTa < ta.mulDiv(10_000 - slippage, 10_000, Math.Rounding.Floor)) {
        revert SlippageTooHigh();
    }

    emit PushedFunds(yieldAssets, withdrawable);
}
```

### Recommended Mitigation Steps

Add the `whenNotPaused` modifier to the function:

```sol
function pushFunds(uint256 yieldAssets, bytes memory) external override whenNotPaused onlyKeeperOrOwner {
  // ...
}
```

**Popcorn**: Fixed in commit [2ed8aab](https://github.com/Popcorn-Limited/contracts/commit/2ed8aab7ae30280b86e4567e06eb15b5137d4b52)

# Low 

## L-01: AnyConverter's slippage parameter is pointless

When the keeper pushes funds by transferring yield assets into the strategy they are allowed
to withdraw $X$ amount of underlying assets of the same value.

```sol
uint256 withdrawable = oracle.getQuote(yieldAssets, _yieldAsset, _asset);
```

The slippage parameter is used to check at the end of the `pushFunds()` function whether
the total assets have not decreased by *too* much. But, `totalAssets()` will never change.
The amount of yield assets transferred and the amount of assets reserved are of the same value, i.e. `prePushTotalAssets == postPushTotalAssets`.

The slippage parameter can be safely removed.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L141
- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L162 

## L-02: Remove default implementation for virtual functions

`BaseStrategy` has multiple virtual functions with default implementations that should be overridden by the inheriting contracts.
Having a default implementation will prevent the compiler from throwing an error if the inheriting contract doesn't override it.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L168
- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L197
- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L202
- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L219
- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L225

If the inheriting contract doesn't use the functions, it should explicitly override them with an empty implementation to prevent unexpected integration issues.

**Popcorn**: Fixed in commit [2ed8aab](https://github.com/Popcorn-Limited/contracts/commit/2ed8aab7ae30280b86e4567e06eb15b5137d4b52)

# QA

## Q-01: `AnyConverter.claimReserved()` emits event even if nothing was claimed

The reserved funds aren't claimed if `withdrawable == 0`. It shouldn't emit an event in that case.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L306

**Popcorn**: Fixed in commit [2ed8aab](https://github.com/Popcorn-Limited/contracts/commit/2ed8aab7ae30280b86e4567e06eb15b5137d4b52)

## Q-02: `AnyConverter.ReserveClaimed` event should include the original withdrawable and the actual withdrawn amount

Currently, it will only include the original withdrawable amount but the actual amount that was claimed can differ.
It will help track the strategy's balance off-chain.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L306

## Q-03: Remove out-of-date TODOs

TODO in `AnyConverter.pushFunds()` is already implemented.
The function takes into account the reserved funds.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/AnyConverter.sol#L116

**Popcorn**: Fixed in commit [2ed8aab](https://github.com/Popcorn-Limited/contracts/commit/2ed8aab7ae30280b86e4567e06eb15b5137d4b52)

## Q-04: Remove unused code

There's a commented-out implementation of `claim()` inside `BaseStrategy`. It's not used anywhere and can be removed.

- https://github.com/Popcorn-Limited/contracts/blob/audit/anyToAny/src/strategies/BaseStrategy.sol#L220

**Popcorn**: Fixed in commit [2ed8aab](https://github.com/Popcorn-Limited/contracts/commit/2ed8aab7ae30280b86e4567e06eb15b5137d4b52)