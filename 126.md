Micro Crimson Scorpion

high

# IERC20.transfer wil fail for USDT

## Summary

Some tokens do not return bool on transfer, e.g. USDT on mainnet

## Vulnerability Detail

USDT on mainnet do not return bool on transfer: https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code#L126

But but because of IERC20 interface solidity will try to parse bool from nothing thus reverting.

## Impact

Unable to claim requested redeem if underlying asset is USDT

## Code Snippet

https://github.com/sherlock-audit/2024-03-amphor/blob/6c797025ffe296e04607abf74400ff2bb36a7de3/asynchronous-vault/src/AsyncSynthVault.sol#L771

## Tool used

Manual Review

## Recommendation

Instead transfer directly to `receiver`

```diff
_asset.safeTransferFrom(address(claimableSilo), receiver, assets);
- _asset.transfer(receiver, assets);
```
