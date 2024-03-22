Decent Malachite Dragon

medium

# The `ZapRouter` can leave a user's unspent tokens after a partial swap.

## Summary

The caller's entire transferred token amount is not guaranteed to be spent by the [`router`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L181) during a swap. These tokens are left on the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol), and can subsequently be stolen by an attacker.

## Vulnerability Detail

The sponsor intends to whitelist 1inch exchange on the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol). The 1inch exchange [supports partial fill swaps](https://help.1inch.io/en/articles/4610960-what-is-the-partial-fill-setting), which will return any unspent tokens back to the caller (in this instance, the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol) itself).

The [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol) provides the assurance that input tokens are at the very least not accrued between swaps:

```solidity
// internal function used only to execute the zap before request a deposit
function _zapIn(
    IERC20 tokenIn,
    address router,
    uint256 amount,
    bytes calldata data
)
internal
{
    uint256 expectedBalance;

    if (msg.value == 0) {
        expectedBalance = tokenIn.balanceOf(address(this)); /// @audit i.e. 1 ether
        _transferTokenInAndApprove(router, tokenIn, amount);
    } else {
        expectedBalance = address(this).balance - msg.value;
    }

    _executeZap(router, data); // zap

    uint256 balanceAfterZap = msg.value == 0
        ? tokenIn.balanceOf(address(this)) /// @audit i.e. 0.8 ether (partial fill)
        : address(this).balance;

    if (balanceAfterZap > expectedBalance) {
        revert InconsistantSwapData({
            expectedTokenInBalance: expectedBalance,
            actualTokenInBalance: balanceAfterZap
        });
    }
}
```

But does not provide such assurances for cases where the user's input amount of tokens have _not_ been fully utilized.

This leaves tokens on the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol), which may contribute to the input balance of an attacker as follows:

1. The [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol) allows the `owner` to configure maximum [`router`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L181) spend approvals for certain tokens via a call to [`approveTokenForRouter(address,address)`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L99).
2. An attacker can specify a nonzero `msg.value` (i.e. `1 wei`) to deactivate the call to [`_transferTokenInAndApprove(address,address,uint256)`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L160), ensuring the maximum spend approval for the asset is still maintained, alongside a swap which underlyingly transacts using the stuck token instead of the `msg.value`.

This would allow an adversary to spend leftover funds from a previous caller's partial fill.

## Impact

Medium, for two reasons:

1. Remaining unspent tokens can indeed be manually rescued by the `owner`, however the current implementation is lacking methods to easily reconcile which stuck tokens belong to which swapper. These functions are designed for the ad-hoc rescue of accidentally transferred tokens, and not the enshrined behaviour of routinely leaving tokens.
4. Stuck tokens can be spent by an attacker, since they are left exposed on the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol) for the interim period leading up to a rescue.

## Code Snippet

```solidity
// internal function used only to execute the zap before request a deposit
function _zapIn(
    IERC20 tokenIn,
    address router,
    uint256 amount,
    bytes calldata data
)
internal
{
    uint256 expectedBalance; // of tokenIn (currently)

    if (msg.value == 0) {
        expectedBalance = tokenIn.balanceOf(address(this));
        _transferTokenInAndApprove(router, tokenIn, amount);
    } else {
        expectedBalance = address(this).balance - msg.value;
    }

    _executeZap(router, data); // zap

    uint256 balanceAfterZap = msg.value == 0
        ? tokenIn.balanceOf(address(this))
        : address(this).balance;

    if (balanceAfterZap > expectedBalance) {
        // Our balance is higher than expected, we shouldn't have received
        // any token
        revert InconsistantSwapData({
            expectedTokenInBalance: expectedBalance,
            actualTokenInBalance: balanceAfterZap
        });
    }
}
```

## Tool used

Manual Review

## Recommendation

There are two possible remediations:

1. Disable partial swaps by forcing a `revert` if the caller's full token balance is not spent by the [`router`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L181) after executing a zap.
2. Ensure any non-zero balance of [`tokenIn`](https://github.com/sherlock-audit/2024-03-amphor/blob/a526990ad80ca80a7a154212cea2f312034b5cd1/asynchronous-vault/src/VaultZapper.sol#L128) left on the [`ZapRouter`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol) is returned to the caller upon completion of a swap.

