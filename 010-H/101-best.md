Soaring Saffron Nuthatch

high

# Incorrect share calculation leads to users receiving less shares if they use SyncSynthVault methods compared to AsyncSynthVault methods.

## Summary

The deposit functions in the `AsyncSynthVault` contract correctly calculate share by using `epochs[requestId].totalAssetsSnapshotForRedeem`for totalAssets, which is the contract balance **minus** fees. 

The  deposit functions in the `SyncSynthVault` contract however, use `totalAssets()` for totalAssets, which is the contract balance **including** the fees. 

As a consequence, users who deposit through the `SyncSynthVault` contract will receive less shares than they should.   


## Vulnerability Detail


The are two ways users can deposit in a vault: 

### AsyncSynthVault
A user can request a deposit through various functions whenever the Vault is closed. After the request has been processed, users can claim their shares by calling `claimAndRequestDeposit`,  `claimAndRequestDepositWithPermit` or `claimDeposit`.  
This calls the internal function `_claimDeposit` which calls `previewClaimDeposit`, which in turn calls `_convertToShares` to calculate the amount of shares.
In `_convertToShares`, we find: 
   
```solidity
        uint256 totalAssets = epochs[requestId].totalAssetsSnapshotForDeposit + 1;
        uint256 totalSupply = epochs[requestId].totalSupplySnapshotForDeposit + 1;
        return assets.mulDiv(totalSupply, totalAssets, rounding);

```
The `totalAssetsSnapshotForDeposit` is calculated in the `previewSettle` function:
```solidity
        // taking fees if positive yield
        _lastSavedBalance = newSavedBalance - fees;

        address pendingSiloAddr = address(pendingSilo);
        uint256 pendingRedeem = balanceOf(pendingSiloAddr);
        uint256 pendingDeposit = _asset.balanceOf(pendingSiloAddr);

        uint256 sharesToMint = pendingDeposit.mulDiv(
            totalSupply + 1, _lastSavedBalance + 1, Math.Rounding.Floor
        );

        uint256 totalAssetsSnapshotForDeposit = _lastSavedBalance + 1;
        uint256 totalSupplySnapshotForDeposit = totalSupply + 1;

```
So the `totalAssets` used in the calculation is the balance of the contract **minus** fees, which is correct and consistent with [EIP4626](https://eips.ethereum.org/EIPS/eip-4626#methods) which states that convertToShares MUST NOT be inclusive of any fees that are charged against assets in the Vault.

Now let's look at the  `SyncSynthVault` contract.


### SyncSynthVault

A user can perform a deposit in the vault at any time by calling `deposit` or `depositWithPermit`. The amount of shares to be returned is obtained from the `previewDeposit` function, which calls `_convertToShares` 

In `_convertToShares`, we find: 
   
```solidity
    function _convertToShares(
        uint256 assets,
        Math.Rounding rounding
    )
        internal view returns (uint256)
    {
        return assets.mulDiv(totalSupply() + 1, totalAssets() + 1, rounding);

    }

```
The `totalAssets` is obtained by calling the `totalAssets()` function:
```solidity
    function totalAssets() public view returns (uint256) {
        if (vaultIsOpen) return _asset.balanceOf(address(this));
        else return _asset.balanceOf(address(this)) + lastSavedBalance;
    }

```

So the `totalAssets` used in the calculation is either the current or stored **total** balance of the contract. 
The `totalAssets`function is consistent with EIP4626, since it states that totalAssets MUST be inclusive of any fees that are charged against assets in the vault.

Yet this also means that `_convertToShares()` cannot be correct, since EIP4626 states that it MUST NOT be inclusive of any fees that are charged against assets in the Vault. So using `totalAssets` in `_convertToShares()` can only be correct if there are **ZERO** fees, which here is not the case. 

As a result, the totalAssets used in this calculation is greater then it should and the users will receive less shares then they should. 

Example: 

Bob deposits 100
totalSupply = 1000
totalAssets = 1500
fee = 20%

In AsyncSynthVault:
shares = 100 * 1000 / (1500-200) = 76 shares

In SyncSynthVault:
shares = 100 * 1000 / 1500 = 66 shares

If Bob uses the SyncSynthVault, he loses 10 shares. 

## Impact

Direct loss of funds to users. 

## Code Snippet

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/SyncSynthVault.sol#L277-L317

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/SyncSynthVault.sol#L449-L451

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/SyncSynthVault.sol#L576-L585

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L204-L213

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L327-L337

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L508-L514

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L532-L545

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L567-L571

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L626-L684

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L742-L756

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L895-L911

## Tool used

Manual Review

## Recommendation

Apply the same pattern use in AsyncSynthVault to SyncSynthVault and do not use totalAssets() for share calculation. 