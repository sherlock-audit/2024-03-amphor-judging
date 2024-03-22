Mythical Cyan Bat

high

# Claiming a deposit in the same Epoch in which the deposit was requested will lead to loss of funds

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

