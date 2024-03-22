Dancing Magenta Deer

medium

# Issue with phantom function permit in certain ERC20 tokens like WETH

## Summary
The `permit` function implementation in both [`VaultZapper.sol`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L367-L384) and [`SyncSynthVault.sol`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/SyncSynthVault.sol#L550-L566) may lead to security vulnerabilities when interacting with ERC20 tokens that implement a fallback mechanism, such as WETH (Wrapped Ethereum). This could potentially open up an attack vector if not handled properly.

## Vulnerability Detail
The current implementation directly calls the `permit` function on the ERC20 token contracts without ensuring that the token contract safely handles calls to non-existent functions (i.e., does not have a problematic fallback function). This can be particularly concerning for tokens like WETH, where the fallback function's behavior could be exploited in certain scenarios.

## Impact
This vulnerability could allow attackers to exploit the fallback function in ERC20 tokens that do not safely handle unexpected function calls. This can lead to unintended interactions with the contract, potentially resulting in loss of funds or unauthorized actions being taken.

## Code Snippet
https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L367-L384
https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/SyncSynthVault.sol#L550-L566

## Tool used
Manual Review

## Recommendation
It is recommended to after the `permit` function calls to check that the allowance has change to the expected value. Implementing this change will help secure the contract against potential vulnerabilities associated with tokens that have fallback mechanisms.
Reference links:
- https://dedaub.com/blog/phantom-functions-and-the-billion-dollar-no-op