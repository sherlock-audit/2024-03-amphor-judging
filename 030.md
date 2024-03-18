Ripe Fiery Snail

high

# Funds locked forever in the Silos contracts

## Summary

Funds can be locked forever in the Silos contracts in tokens like WBTC, which reduces allowance on every transferFrom, even if allowance is set uint256 max.

## Vulnerability Detail

In the Silos contracts we [approve uint256 max](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L95) the Vault contract. The assumption is that the Vault then will be able to always transfer from Silos' contract. Indeed some contracts (like WETH) don't decrease allowance when it is set to uint256 max. However, some contracts (like [WBTC](https://etherscan.io/address/0x2260fac5e5542a773aa44fbcfedf7c193bc2c599#code), which is listed as one of the vault's underlying) decrease allowance even if allowance is set to uint256. This means every `transferFrom` will decrease allowance, and eventually, it will go to 0, which would cause locking funds forever.

The situation is even worse, because a malicious actor can transfer back and forth, to and from Silo, tokens by repeatedly calling [requestDeposit](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L439), and [decreaseDepositRequest](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L234). Every round of such cycle would decrease allowance, and would lead at the end to DoS of the Silo, which would lock funds forever in it.

## Impact

Funds locked forever inside the Silos contracts, with no way to recover them. DoS attack possible.

## Code Snippet

```solidity
contract Silo {
    constructor(IERC20 underlying) {
        underlying.forceApprove(msg.sender, type(uint256).max);
    }
}
```

## Tool used

Manual Review

## Recommendation

There should be a way to increase the allowance for SIlo contracts. I recommend adding `approveVault` function, which might be called by anyone, and it will set approval to uint256 max again.

```solidity
contract Silo {
    address public immutable vault;
    IERC20 public immutable underlying;

    constructor(IERC20 _underlying) {
        _underlying.forceApprove(msg.sender, type(uint256).max);

        underlying = _underlying;
        vault = msg.sender;
    }

    function approveVault() external {
        underlying.forceApprove(vault, type(uint256).max);
    }
}


```
