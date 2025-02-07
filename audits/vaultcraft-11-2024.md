# VaultCraft - Gnosis Vault Security Review by Ruhum

- [Summary](#summary)
- [High Risk](#high-risk)
  - [H-01: Depositing or minting with claimable assets leads to an inflated share supply](#h-01-depositing-or-minting-with-claimable-assets-leads-to-an-inflated-share-supply)
  - [H-02: User's withdrawal uses share price without fees taken](#h-02-users-withdrawal-uses-share-price-without-fees-taken)
  - [H-03: Vault expects the asset to have 18 decimals when calculating the performance fee](#h-03-vault-expects-the-asset-to-have-18-decimals-when-calculating-the-performance-fee)
  - [H-04: User can sandwich the PushOracle](#h-04-user-can-sandwich-the-pushoracle)
  - [H-05: Vault should take the fee before the user's deposit](#h-05-vault-should-take-the-fee-before-the-users-deposit)
- [Medium Risk](#medium-risk)
  - [M-01: Vault shares should be burned in `fulfillRedeem()`](#m-01-vault-shares-should-be-burned-in-fulfillredeem)
  - [M-02: Adjusting the vault's fees can break the `highWaterMark`](#m-02-adjusting-the-vaults-fees-can-break-the-highwatermark)
- [Low](#low)
  - [L-01: `maxMint()` should return the remaining depositLimit](#l-01-maxmint-should-return-the-remaining-depositlimit)
  - [L-02: `AsyncVault._fulfillRedeem()` will not pull the correct amount of assets](#l-02-asyncvault_fulfillredeem-will-not-pull-the-correct-amount-of-assets)
  - [L-03: `AsyncVault.fulfillMultipleRedeems()` will use the wrong share price](#l-03-asyncvaultfulfillmultipleredeems-will-use-the-wrong-share-price)


## Summary

**Author**: Ruhum

**Project**: VaultCraft - Gnosis Vault

**Repo**: https://github.com/Popcorn-Limited/contracts/tree/feat/GnosisVault

**Commit**: 983f437a0c36752765a393534b691a370ec1b771

**Timeline**: 2024/11/04 - 2024/11/17

Scope:
```
├── src
│   ├── vaults
│   │   └── multisig
│   │       └── phase1
│   │            ├── AsyncVault.sol
│   │            ├── BaseControlledAsyncRedeem.sol
│   │            ├── BaseERC7540.sol
│   │            └── OracleVault.sol
│   └── peripheral
|        └── oracles
|             ├── adapter
│             │   └── pushOracle
│             │       └── PushOracle.sol
│             └── OracleVaultController.sol
├── test
│   ├── vaults
│   │   └── multisig
│   │       └── phase1
│   │            ├── AsyncVault.t.sol
│   │            ├── BaseControlledAsyncRedeem.sol
│   │            ├── BaseERC7540.t.sol
│   │            └── OracleVault.t.sol
│   └── peripheral
│       ├── PushOracle.t.sol
│       └── OracleVaultController.t.sol
```

**Issues found**:
| Severity | Count |
| -------- | ----- |
| High     | 5     |
| Medium   | 2     |
| Low      | 3     |
| QA       | 0     |

## High Risk

### H-01: Depositing or minting with claimable assets leads to an inflated share supply

When a user deposits or mints while they have claimable assets from a fulfilled redemption request, the
share supply is inflated. The BaseControlledAsyncRedeem contract doesn't burn the shares that were 
transferred when the user requested their redemption while minting new shares for the deposit executed
with the already deposited assets.

#### Vulnerability Details

When a user mints or redeems, BaseControlledAsyncRedeem first uses the claimable assets to fulfill the
deposit using existing funds through `claimableAssets`. It then mints the full amount of shares for the deposit, see [BaseControlledAsyncRedeem.sol:L45](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L45-L45) & [BaseControlledAsyncRedeem.sol:L96](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L96)

```sol
function deposit(
    uint256 assets,
    address receiver
) public override whenNotPaused returns (uint256 shares) {
    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    // Utilise claimable balance first
    uint256 assetsToTransfer = assets;
    RequestBalance storage currentBalance = requestBalances[msg.sender];
    if (currentBalance.claimableAssets > 0) {
        // Ensures we cant underflow when subtracting from assetsToTransfer
        uint256 claimableAssets = assetsToTransfer >
            currentBalance.claimableAssets
            ? currentBalance.claimableAssets
            : assetsToTransfer;

        // Modify the currentBalance state accordingly
        _withdrawClaimableBalance(claimableAssets, currentBalance);

        assetsToTransfer -= claimableAssets;
    }

    // Transfer the remaining assets from the sender
    if (assetsToTransfer > 0) {
        // Need to transfer before minting or ERC777s could reenter.
        SafeTransferLib.safeTransferFrom(
            asset,
            msg.sender,
            address(this),
            assetsToTransfer
        );
    }

    // Mint shares to the receiver
    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);

    // Additional logic for inheriting contracts
    afterDeposit(assets, shares);
}

function mint(
    uint256 shares,
    address receiver
) public override whenNotPaused returns (uint256 assets) {
    require(shares != 0, "ZERO_SHARES");
    assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.

    // Utilise claimable balance first
    uint256 assetsToTransfer = assets;
    RequestBalance storage currentBalance = requestBalances[msg.sender];
    if (currentBalance.claimableAssets > 0) {
        // Ensures we cant underflow when subtracting from assetsToTransfer
        uint256 claimableAssets = assetsToTransfer >
            currentBalance.claimableAssets
            ? currentBalance.claimableAssets
            : assetsToTransfer;

        // Modify the currentBalance state accordingly
        _withdrawClaimableBalance(claimableAssets, currentBalance);

        assetsToTransfer -= claimableAssets;
    }

    // Transfer the remaining assets from the sender
    if (assetsToTransfer > 0) {
        // Need to transfer before minting or ERC777s could reenter.
        SafeTransferLib.safeTransferFrom(
            asset,
            msg.sender,
            address(this),
            assetsToTransfer
        );
    }

    // Mint shares to the receiver
    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);

    // Additional logic for inheriting contracts
    afterDeposit(assets, shares);
}
```

When a user requests a withdrawal, it will transfer the shares to the vault, see [BaseControlledAsyncRedeem.sol:L442](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L442)

```sol
function _requestRedeem(
    uint256 shares,
    address controller,
    address owner
) internal returns (uint256 requestId) {
    // ...

    // Transfer shares from owner to vault (these will be burned on withdrawal)
    SafeTransferLib.safeTransferFrom(this, owner, address(this), shares);

    // ...
}
```

Those shares are burned when the user exeuctes the withdrawal at the end. But, if they instead, use the shares to execute a deposit, those shares will be left in the vault infalting the total supply.

Given that:
1. Alice deposits 100 assets to receive 100 shares.
2. Alice requests a redemption for 100 shares

At this point, the vault holds 100 assets and 100 shares.

3. Alice deposits another 100 assets (because she has 100 claimable assets from the redemption request,
she won't have to transfer additional funds)

Now, the vault holds 100 assets and has issued 200 shares. 100 of those shares are held by Alice while
the other 100 are in the vault.

#### Recommended Mitigation Steps

The vault should transfer those shares back to the user and only mint new shares if the user deposited
more assets than they had stored in `claimableAssets`.

**VaultCraft**: fixed in f8e745c0b1db785d3ade509dcbcaef75b0eda520

### H-02: User's withdrawal uses share price without fees taken

For a user's withdrawal, the vault uses the share price without taking the fees into account, giving the user a favorable share price.

#### Vulnerability Details

In the withdrawal flow, fees are taken when the user executes their withdrawal, **after** it was fulfilled by the keeper, see [AsyncVault.sol:343](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L343) & [BaseControlledAsyncRedeem.sol:170](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L170)

```sol
// AsyncVault.sol
function beforeWithdraw(uint256 assets, uint256) internal virtual override {
    if (!paused) _takeFees();
}

// BaseControlledAsyncRedeem.sol
function withdraw(
    uint256 assets,
    address receiver,
    address controller
) public virtual override returns (uint256 shares) {
    require(
        controller == msg.sender || isOperator[controller][msg.sender],
        "ERC7540Vault/invalid-caller"
    );
    require(assets != 0, "ZERO_ASSETS");

    RequestBalance storage currentBalance = requestBalances[controller];
    shares = assets.mulDivUp(
        currentBalance.claimableShares,
        currentBalance.claimableAssets
    );

    // Modify the currentBalance state accordingly
    _withdrawClaimableBalance(assets, currentBalance);

    // Additional logic for inheriting contracts
    beforeWithdraw(assets, shares);

    // Burn controller's shares
    _burn(address(this), shares);

    // Transfer assets to the receiver
    SafeTransferLib.safeTransfer(asset, receiver, assets);

    emit Withdraw(msg.sender, receiver, controller, assets, shares);
}
```

At this point, the share -> asset conversion has already happened. That takes place in `fulfillRedeem()` before any fees are taken, see [AsyncVault.sol:L259](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L259)

```sol
function fulfillRedeem(
    uint256 shares,
    address controller
) external override returns (uint256 assets) {
    // Using the lower bound totalAssets ensures that even with volatile strategies and market conditions we will have sufficient assets to cover the redeem
    assets = convertToLowBoundAssets(shares);

    // Calculate the withdrawal incentive fee from the assets
    Fees memory fees_ = fees;
    uint256 fees = assets.mulDivDown(
        uint256(fees_.withdrawalIncentive),
        1e18
    );

    // Fulfill the redeem request
    _fulfillRedeem(assets - fees, shares, controller);

    // Send the withdrawal incentive fee to the fee recipient
    handleWithdrawalIncentive(fees, fees_.feeRecipient);
}
```

**VaultCraft**: fixed in b5c20a41f490912b4632ad7367718b4abd509a27

### H-03: Vault expects the asset to have 18 decimals when calculating the performance fee

The vault will miscalculate the performance fee if the asset has more or less than 18 decimals.

#### Vulnerability Details

The vault expects the underlying asset to have 18 decimals when calculating the performance fee.

In `_accruedPerformanceFee()` it will calculate the performance fee as: `performanceFee.mulDivUp((shareValue - highWaterMark) * totalSupply, 1e36)`

`performanceFee` is scaled by 1e18, while `shareValue`, `highWaterMark`, and `totalSupply` have the same number of decimals as the asset.
So if the underlying asset is USDC for example, the performance fee will very likely round down to 0.

Here's a PoC showing that with a 6-decimal token:

```sol
// AsyncVault.t.sol
function test_performance_fee() public {
    asset = new MockERC20("Test Token", "TEST", 6);

    InitializeParams memory params = InitializeParams({
        asset: address(asset),
        name: "Vault Token",
        symbol: "vTEST",
        owner: owner,
        limits: Limits({depositLimit: type(uint256).max, minAmount: 0}),
        fees: Fees({
            performanceFee: 1e17, // 10%
            managementFee: 0,
            withdrawalIncentive: 0,
            feesUpdatedAt: uint64(block.timestamp),
            highWaterMark: 0,
            feeRecipient: feeRecipient
        })
    });

    asyncVault = new MockAsyncVault(params);

    // Alice deposits 100 USDC
    asset.mint(alice, 100e6);
    vm.startPrank(alice);
    asset.approve(address(asyncVault), 100e6);
    asyncVault.deposit(100e6, alice);
    vm.stopPrank();

    // Give the vault another 100 USDC so it can accrue a performance fee
    deal(address(asset), address(asyncVault), 100e6);

    vm.warp(block.timestamp + 1 days);

    asyncVault.takeFees();

    assertEq(asset.balanceOf(feeRecipient), 0);
}
```

`1e17 * (200e6 - 100e6) / 1e36 = 1e-11`.

#### Recommended Mitigation Steps

Use the asset's or vault's decimals for the calculation

**VaultCraft**: fixed in f8e745c0b1db785d3ade509dcbcaef75b0eda520

### H-04: User can sandwich the PushOracle

A user can sandwich the PushOracle's price update to sell shares and buy back at a lower price.

#### Vulnerability Details

With the PushOracle, any price update is public. A user can sandwich a price drawdown to frontrun 
the oracle update and sell shares for the current price, execute the oracle update, and buy back at a lower price.

Because [`AsyncVault.fulfillRedeem()`](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L259) is a permissionless function, anybody can execute withdrawals
from the Safe to execute a withdrawal atomically through a bundle.

```sol
function fulfillRedeem(
    uint256 shares,
    address controller
) external override returns (uint256 assets) {
    // Using the lower bound totalAssets ensures that even with volatile strategies and market conditions we will have sufficient assets to cover the redeem
    assets = convertToLowBoundAssets(shares);

    // Calculate the withdrawal incentive fee from the assets
    Fees memory fees_ = fees;
    uint256 fees = assets.mulDivDown(
        uint256(fees_.withdrawalIncentive),
        1e18
    );

    // Fulfill the redeem request
    _fulfillRedeem(assets - fees, shares, controller);

    // Send the withdrawal incentive fee to the fee recipient
    handleWithdrawalIncentive(fees, fees_.feeRecipient);
}
```

So the attack goes as follows:
1. Current share price is 1e18 and Alice holds 2e18 shares
2. Alice sees a PushOracle tx that sets the price to 1.1e18 (meaning the share price decreased)
3. Alice executes a bundle of transactions:
   1. Alice calls `requestRedeem()` to redeem her shares
   2. Alice calls `fulfillRedeem()` to execute the redemption of her 2e18 shares
    She receives $2e18 * 1e18 / 1e18 = 2e18$ assets
   3. Alice finalizes the withdrawal using `withdraw()` to receive her 2e18 assets
   4. The PushOracle updates the price to 9e17
   5. Alice executes `deposit()` to deposit her 2e18 assets. She receives $2e18 * 1.1e18 / 1e18 = 2.2e18$ shares

#### Recommended Mitigation Steps

Either you pause deposits & withdrawals **before** the oracle update tx hits the mempool or you make
`fulfillRedeem()` a permissioned function.

**VaultCraft**: Acknowledged

### H-05: Vault should take the fee before the user's deposit

By taking the fee after the user's deposit, the user will receive a less favorable share price.

#### Vulnerability Details

Currently, fees are taken after the user's deposit in `deposit()` and `mint()`, see [AsyncVault.sol:L353](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L353) & [BaseControlledAsyncRedeem.sol:L45](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L45)

```sol
// BaseControlledAsyncRedeem.sol
function deposit(
    uint256 assets,
    address receiver
) public override whenNotPaused returns (uint256 shares) {
    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    // Utilise claimable balance first
    uint256 assetsToTransfer = assets;
    RequestBalance storage currentBalance = requestBalances[msg.sender];
    if (currentBalance.claimableAssets > 0) {
        // Ensures we cant underflow when subtracting from assetsToTransfer
        uint256 claimableAssets = assetsToTransfer >
            currentBalance.claimableAssets
            ? currentBalance.claimableAssets
            : assetsToTransfer;

        // Modify the currentBalance state accordingly
        _withdrawClaimableBalance(claimableAssets, currentBalance);

        assetsToTransfer -= claimableAssets;
    }

    // Transfer the remaining assets from the sender
    if (assetsToTransfer > 0) {
        // Need to transfer before minting or ERC777s could reenter.
        SafeTransferLib.safeTransferFrom(
            asset,
            msg.sender,
            address(this),
            assetsToTransfer
        );
    }

    // Mint shares to the receiver
    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);

    // Additional logic for inheriting contracts
    afterDeposit(assets, shares);
}

// AsyncVault.sol
function afterDeposit(uint256 assets, uint256) internal virtual override {
    // deposit and mint already have the `whenNotPaused` modifier so we don't need to check it here
    _takeFees();
}
```

When the vault takes fees, it will mint new shares to the fee recipient of equal value to the fees taken.
That increases the total supply. That in turn means the amount of shares the user receives for their deposit
will also be higher.

#### Recommended Mitigation Steps

Take the fees before the user's deposit in `deposit()` and `mint()`.

**VaultCraft**: Acknowledged

## Medium Risk

### M-01: Vault shares should be burned in `fulfillRedeem()`

Vault shares should be burned when the user's redemption request is fulfilled to not inflate the share supply.

#### Vulnerability Details

Vault shares are currently burned when the user executes their withdrawal after it was fulfilled
by the keeper, see [BaseControlledAsyncRedeem.sol:L173](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/BaseControlledAsyncRedeem.sol#L173)

```sol
    function withdraw(
        uint256 assets,
        address receiver,
        address controller
    ) public virtual override returns (uint256 shares) {
        // ...

        // Burn controller's shares
        _burn(address(this), shares);

        // ...
    }
```

After the user's request is fulfilled, the assets will generally sit idly in the vault generating no yield.

So given the following state:
- 1000 assets in a yield-generating strategy (totalAssets = 1000), earning a yield of 5%
    - 50 assets per year
- 1000 shares issued to users

Alice requests 100 shares to be redeemed. Her request is filled immediately by the keeper
and 100 assets are transferred to the vault ready to be withdrawn by Alice.

At this point:
- 900 assets in a yield generating strategy & 100 assets in the vault (totalAssets = 1000)
- 900 shares in circulation & 100 shares held by the vault itself (totalSupply = 1000) 

Alice doesn't withdraw her funds immediately but lets them sit in the vault. 
A year passes and the vault has earned 45 assets in yield, boosting the totalAssets to 1045.
If Bob now decides to redeem 100 of his shares, he will receive: $100 * 1045 / 1000 = 104.5$ assets.
But, if the actual working supply of assets and total supply was used, he would receive: $100 * 945 / 900 = 105$ assets.
By Alice not executing her withdrawal and another user withdrawing before her, she decreases the earned
yield of that user without contributing any assets to the protocol.

#### Recommend Mitigation Steps

Burn shares when the user's redemption request is fulfilled.

**VaultCraft**: fixed in b5c20a41f490912b4632ad7367718b4abd509a27

### M-02: Adjusting the vault's fees can break the `highWaterMark`

When the admin changes the vault's fees, they have to also adjust the `highWaterMark`.
Using the wrong value there, will cause the vault to take a performance fee on any profit that's realised after that fee change.

#### Vulnerability Details

In [`AsyncVault._setFees()`](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L477) it will set the `highWaterMark` to the current share value.
Then it will override that value with the one passed by the caller:

```sol
function _setFees(Fees memory fees_) internal {
    // ...

    fees_.highWaterMark = convertToAssets(1e18);

    fees = fees_;
}
```

So setting the `highWaterMark` as the current share price is pointless. If it were not overridden that
would be a bug in itself. Because the current share price will always be equal to or lower than the
`highWaterMark`. So you will always take a performance fee without the vault realising any profit.

Having the `highWaterMark` set by the caller is also prone to errors The correct approach is
to not modify the value at all. `_takeFees()` will take care of that.

#### Recommended Mitigation Steps

Don't update `highWaterMark` in `_setFees()`. This will require you to remove the `highWaterMark`
from the `Fees` struct.
Currently, the `highWaterMark` is initialized through the `_setFees()` function in the constructor of
`AsyncVault`. So you have to manually set it to the current share price:
`highWaterMark = convertToAssets(10 ** IERC20(address(this)).decimals());`

**VaultCraft**: fixed in f8e745c0b1db785d3ade509dcbcaef75b0eda520

## Low 

### L-01: `maxMint()` should return the remaining depositLimit

If `depositLimit == type(uint256).max`, `maxMint()` will return `type(uint256).max` instead of the remaining deposit limit, see [AsyncVault.sol:210](https://github.com/Popcorn-Limited/contracts/blob/983f437a0c36752765a393534b691a370ec1b771/src/vaults/multisig/phase1/AsyncVault.sol#L210)

```sol
function maxMint(address) public view override returns (uint256) {
    uint256 assets = totalAssets();
    uint256 depositLimit_ = limits.depositLimit;

    if (paused || assets >= depositLimit_) return 0;
    if (depositLimit_ == type(uint256).max) return depositLimit_;

    return convertToShares(depositLimit_ - assets);
}
```

**VaultCraft**: fixed in f8e745c0b1db785d3ade509dcbcaef75b0eda520

### L-02: `AsyncVault._fulfillRedeem()` will not pull the correct amount of assets

In `AsyncVault._fulfillRedeem()` is supposed to handle the user's redeem request. It will pull the
given amount of assets into the vault to allow the user to execute their withdrawal. But
the given amount doesn't include the withdrawal incentive that's sent to the fee recipient.
That will cause the vault to not have enough assets to cover the full withdrawal.

`fulfillRedeem()` will call `_fulfillRedeem()` with `asset - fees`:

```sol
function fulfillRedeem(
    uint256 shares,
    address controller
) external override returns (uint256 assets) {
    // Using the lower bound totalAssets ensures that even with volatile strategies and market conditions we will have sufficient assets to cover the redeem
    assets = convertToLowBoundAssets(shares);

    // Calculate the withdrawal incentive fee from the assets
    Fees memory fees_ = fees;
    uint256 fees = assets.mulDivDown(
        uint256(fees_.withdrawalIncentive),
        1e18
    );

    // Fulfill the redeem request
    _fulfillRedeem(assets - fees, shares, controller);

    // Send the withdrawal incentive fee to the fee recipient
    handleWithdrawalIncentive(fees, fees_.feeRecipient);
}
```

In `_fulfillRedeem()` it will call `beforeFulfillRedeem()` which is supposed to transfer the assets
to the vault.
In `OracleVault` it pulls the funds from the safe for example.

```sol
    /// @dev Internal function to fulfill a redeem request
    function _fulfillRedeem(
        uint256 assets,
        uint256 shares,
        address controller
    ) internal virtual returns (uint256) {
        // ...

        // Additional logic for inheriting contracts
        beforeFulfillRedeem(assets, shares);

        // ...
    }
```

After that, it will send the withdrawal incentive to the fee recipient using `handleWithdrawalIncentive()`:

```sol
function handleWithdrawalIncentive(
    uint256 fee,
    address feeRecipient
) internal virtual {
    if (fee > 0) SafeTransferLib.safeTransfer(asset, feeRecipient, fee);
}
```

Meaning, if Alice has withdrawn 100 shares worth 100 assets, and we have a 5% withdrawal incentive,
95 assets will be pulled into the vault. Then, 5 assets will be sent to the fee recipient, leaving
the vault with only 90 assets. That's not enough to cover Alice's withdrawal of 95 assets.

> NOTE: this is not an issue for the `OracleVault` because it overrides `handleWithdrawalIncentive()` to send the funds from the safe to the fee recipient instead of sending them from the vault itself.
Because the AsyncVault isn't used independently in the current scope, I didn't think it was a valid MED/HIGH issue.

**VaultCraft**: Acknowledged

### L-03: `AsyncVault.fulfillMultipleRedeems()` will use the wrong share price

In `AsyncVault.fulfillMultipleRedeem()` it will send the withdrawal incentive at the very end of the function.
That will cause the share price to be inflated inside the for loop where it handles the redeem requests.

#### Vulnerability Details

> NOTE: this is not an issue with the OracleVault because it calculates its total assets using the vault's total supply and the share price through the oracle. But, if the AsyncVault is used standalone or inherited by a contract that doesn't use an oracle, it will be an issue.
Because the AsyncVault isn't used independently in the current scope, I didn't think it was a valid MED/HIGH issue.

In `fulfillMultipleRedeems()` it will loop through multiple requests and calculate the asset amount as well as the withdrawal incentive fee.
The share -> asset conversion uses `totalAssets()`. For a normal ERC4626 vault that will be the 
asset balance of the vault. By sending the withdrawal incentive at the end, `totalAssets()` will be
bigger than it should be for any redemption request after the first one.

```sol
    function fulfillMultipleRedeems(
        uint256[] memory shares,
        address[] memory controllers
    ) external returns (uint256 total) {
        // ...
        for (uint256 i; i < shares.length; i++) {
            // Using the lower bound totalAssets ensures that even with volatile strategies and market conditions we will have sufficient assets to cover the redeem
            uint256 assets = convertToLowBoundAssets(shares[i]);

            // Calculate the withdrawal incentive fee from the assets
            uint256 fees = assets.mulDivDown(
                uint256(fees_.withdrawalIncentive),
                1e18
            );

            // Fulfill the redeem request
            _fulfillRedeem(assets - fees, shares[i], controllers[i]);

            // Add to the total assets and fees
            total += assets;
            totalFees += fees;
        }

        // Send the withdrawal incentive fee to the fee recipient
        handleWithdrawalIncentive(totalFees, fees_.feeRecipient);

        return total;
    }
```

#### Recommended Mitigation Steps

Handle fees within the for loop.

**VaultCraft**: Acknowledged