# VaultCraft AnyToAny V2 Security Review by Ruhum

- [Summary](#summary)
- [High Risk](#high-risk)
  - [H-01: Keeper can steal a strategy's assets](#h-01-keeper-can-steal-a-strategys-assets)
- [Medium Risk](#medium-risk)
  - [M-01: Keeper can't pull funds when the strategy is paused](#m-01-keeper-cant-pull-funds-when-the-strategy-is-paused)
- [Low](#low)
  - [L-01: slippage calculation in `pushFunds()` and `pullFunds()` should round up](#l-01-slippage-calculation-in-pushfunds-and-pullfunds-should-round-up)
  - [L-02: owner can't cancel proposed call allowances](#l-02-owner-cant-cancel-proposed-call-allowances)


## Summary

**Author**: Ruhum

**Project**: VaultCraft AnyToAny V2

**Repo**: https://github.com/Popcorn-Limited/contracts/tree/audit/anyV2

**Commit**: 6db4d02cc943db4c91f8b6f799d828783debefcc

**Timeline**: 03/10/2024 - 

Scope:
- src/strategies/any/v2/AnyConverter2.sol
- src/strategies/any/v2/AnyCompounderNaiveV2.sol
- src/strategies/any/v2/AnyCompounderV2.sol
- src/strategies/any/v2/AnyDepositorV2.sol
- src/strategies/BaseStrategy.sol

**Issues found**:
| Severity | Count |
| -------- | ----- |
| High     | 1     |
| Medium   | 1     |
| Low      | 2     |
| QA       | 0     |

## High Risk

### H-01: Keeper can steal a strategy's assets

For the keeper to execute `pullFunds()` and `pushFunds()` the strategies owner has to allow the
keeper to execute specific contract functions. 

#### Vulnerability Details

For a keeper to execute `pullFunds()` and `pushFunds()` the strategy's owner has to allow them to
execute specific contract functions:

```sol
function _convert(
    bytes memory data
) internal returns (uint256, uint256, uint256, uint256, uint256, uint256) {
    // ...

    CallStruct[] memory calls = abi.decode(data, (CallStruct[]));

    for (uint256 i; i < calls.length; i++) {
        if (!isAllowed[calls[i].target][bytes4(calls[i].data)])
            revert("Not allowed");

        (bool success, ) = calls[i].target.call(calls[i].data);
        if (!success) revert("Call failed");
    }

    // ... 
}
```

As seen in the tests, the `approve()` function shall be allowed to be called for both the `asset` and `yieldToken`: [AnyBase.t.sol#L22](https://github.com/Popcorn-Limited/contracts/blob/audit/anyV2/test/strategies/any/v2/AnyBase.t.sol#L22C1-L41C12)

```sol
function _setUpBase() internal {
    exchange = new MockExchange();

    PendingCallAllowance[] memory changes = new PendingCallAllowance[](3);
    changes[0] = PendingCallAllowance({
        target: testConfig.asset,
        selector: bytes4(keccak256("approve(address,uint256)")),
        allowed: true
    });
    changes[1] = PendingCallAllowance({
        target: yieldToken,
        selector: bytes4(keccak256("approve(address,uint256)")),
        allowed: true
    });
    changes[2] = PendingCallAllowance({
        target: address(exchange),
        selector: bytes4(
            keccak256(
                "swapTokenExactAmountIn(address,uint256,address,uint256)"
            )
        ),
        allowed: true
    });

    AnyConverterV2(address(strategy)).proposeCallAllowance(changes);

    vm.warp(block.timestamp + 3 days + 1);
    AnyConverterV2(address(strategy)).changeCallAllowances();
}
```

By allowing the keeper to call `approve()` they can approve all the strategy's assets and yield tokens
to their own address to take out all of the funds.

Another approach, instead of approving the keeper to spend funds, is to allow them to transfer directly by calling `transferFrom()` on `asset` and `yieldToken`. But, that opens up another issue.
The vault has max-approved the strategy to spend `asset`. If you allow the keeper to execute `transferFrom()` calls, they can "steal" funds from the vault. The same thing applies to any other address
that approves the strategy to spend `asset`.

#### Recommended Mitigation Steps

The allowed contract calls should be stricter by limiting the allowed parameters for function calls.
For example, you could limit `transferFrom()` calls such that the `from` parameter is only allowed to
be the keeper's address.
Or you subtract the approved funds from the `asset`/`yieldToken` balance returned by the `_convert()` function. 

**VaultCraft**: Acknowledged. Keeper is a trusted party.

## Medium Risk

### M-01: Keeper can't pull funds when the strategy is paused

When the strategy is paused, the keeper isn't allowed to call `pullFunds()` anymore. 

#### Vulnerability Details

If the keeper can't trade yield tokens for the base asset, the vault won't be able
to withdraw any funds. That, in turn, means the user won't be able to withdraw their funds.

https://github.com/Popcorn-Limited/contracts/blob/audit/anyV2/src/strategies/any/v2/AnyConverterV2.sol#L192
```sol
function pullFunds(
    uint256,
    bytes memory data
) external override onlyKeeperOrOwner whenNotPaused {
    // ...
}
```

#### Recommended Mitigation Steps

Remove the `whenNotPaused` modifier from the `pullFunds()` function.

**VaultCraft**: Fixed in commit [599aeb8](https://github.com/Popcorn-Limited/contracts/commit/599aeb8bc695d62c6804d112fe62797e7dcb9300)
## Low 

### L-01: slippage calculation in `pushFunds()` and `pullFunds()` should round up

When verifying that the total assets after pushing/pulling funds haven't decreased in any significant way,
you should round up the slippage calculation in favor of the protocol:

https://github.com/Popcorn-Limited/contracts/blob/audit/anyV2/src/strategies/any/v2/AnyConverterV2.sol#L170
https://github.com/Popcorn-Limited/contracts/blob/audit/anyV2/src/strategies/any/v2/AnyConverterV2.sol#L208
```sol
// Total assets should stay the same or increase (with slippage)
if (
    postTotalAssets <
    preTotalAssets.mulDiv(
        10_000 - slippage,
        10_000,
        // @audit should round up to be more favorable for the protocol
        Math.Rounding.Floor
    )
) revert("Total assets decreased");
```

**VaultCraft**: Fixed in commit [599aeb8](https://github.com/Popcorn-Limited/contracts/commit/599aeb8bc695d62c6804d112fe62797e7dcb9300)

### L-02: owner can't cancel proposed call allowances

If the admin misconfigured the call allowances and proposed a call that they don't want to add,
there's no way to reset `proposedCallAllowances`. You have to execute the proposed calls.
To reset one proposal, you have to override by proposing it again and setting `allowed = false`.

https://github.com/Popcorn-Limited/contracts/blob/audit/anyV2/src/strategies/any/v2/AnyConverterV2.sol#L317
```sol
function proposeCallAllowance(
    PendingCallAllowance[] calldata callAllowances
) external onlyOwner {
    for (uint256 i; i < callAllowances.length; i++) {
        pendingCallAllowances.push(callAllowances[i]);

        emit CallAllowanceProposed(
            callAllowances[i].target,
            callAllowances[i].selector,
            callAllowances[i].allowed
        );
    }
    pendingCallAllowanceTime = block.timestamp + 3 days;
}
```

Instead, calling `proposeCallAllowance()` should delete the previously proposed calls.

**VaultCraft**: Acknowledged.