Amateur Cinnabar Raccoon

high

# Exploiting Funds from ZapperVault Contract Due to Approval to the Router

## Summary

If the VaultZapper.sol approves the 1inch router agreegator for any specific token, there's an high chance of pulling out the tokens from the VaultZapper contract. 

## Vulnerability Detail

##### Brief Info about ZapDeposits:
- Zap deposits automate the process of buying assets to automatically enter into the respective vaults. The main use of this function is, user can send the "approved tokens" tokens directly to the VaultZapper which actually converts these tokens into required underlying tokens via 1inch, in order deposit them into the vaults. 

Nevertheless, As an example,  the important aspect here is the data that we're passing into the VaultZapper.sol contract on the function 'zapAndDeposit'. 

https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L178-L183

```solidity
function zapAndDeposit(
        IERC20 tokenIn,
        IERC4626 vault,
        address router,
        uint256 amount,
        bytes calldata data
    ) public
        payable
        onlyAllowedRouter(router)
        onlyAllowedVault(vault)
        whenNotPaused
        returns (uint256)
```

The important aspect here is, the "tokenIn" isn't actually verified/approved by the ZapperVault along with the calldata. Here, an attacker can pass his own controlled token and pass any arbitary call data. As the contract for now, allows 1inch Router aggregator v4, one can actually use the swap function on the v4 to exploit this issue. 

If in future, the contract supports 1inch router aggregator v5, there's an direct exploition by calling the 'arbitaryStaticCall' function to directly transfer out the funds. 

### Proof-Of-Concept 


>> Note that here there are two pre-requisities:
- The VaultZapper contract should approve the 1inch router to spend the USDC tokens using the 'approveTokenForRouter()' function. 


> For 1inch Router Aggregator V4: 

- Exploiter will call the 'zapAndDeposit()' function on VaultZapper with the following values: 
--> tokenIn: User Controlled ERC20 Token
--> vault: Amphor Vault address(let's say underlying token is USDC)
--> router: 1inch Router V4 address 
--> amount: X (let's say 10 for simplicity)

- Now, the VaultZapper checks for total number of USDC tokens that it holds at present. And in the consecutive line,  the contract makes an internal call to "_zapIn()". 

- In "_zapIn()" function, as we didn't send any msg.value, it checks for "how many tokens that it holds currently". (Note: Here the contract checks, how many user-controlled tokens does this contract holds.)

- And it call the "_transferTokenAndApprove()" function, which transfers the user-controlled tokens from the Exploiter to the VaultZapper contract and also VaultZapper approves the router to spend all the user-controlled tokens. 

- Next, it calls the "_executeZap()" function, where it makes an low-level call to the router contract with the user given 'arbitary calldata'.  (https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L351-L365) 

```solidity
function _executeZap(
        address target,
        bytes memory data
    )
        internal
        returns (bytes memory response)
    {
        (bool success, bytes memory _data) =
            target.call{ value: msg.value }(data);
        if (!success) {
            if (data.length > 0) revert SwapFailed(string(_data));
            else revert SwapFailed("Unknown reason");
        }
        return _data;
    }

```

- At this instance, user can give the calldata such as "swap()" with respective parameters and mainly "srcToken" as USDC. If you look into the 1inch router V4 code, on line 2341, it actually transfers the USDC from msg.sender(VaultZapper) to the desc.srcReciever(Exploiter) thus successfully claiming the funds.  (https://etherscan.io/address/0x1111111254fb6c44bac0bed2854e76f90643097d#code) 

> For 1inch Router Aggregator V5:

- This follows the same as previous steps, but here, exploit turns out so easy as the function is interacting with the 'arbitaryStaticCall' (https://etherscan.io/address/0x1111111254EEB25477B68fb85Ed929f73A960582#code) - Check line 4018. 

## Impact

By allowing the router to spend the tokens of VaultZapper, you're giving an scope of looting/stelaing the funds directly. 

## Code Snippet

- https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L127
- https://github.com/sherlock-audit/2024-03-amphor/blob/main/asynchronous-vault/src/VaultZapper.sol#L351

## Tool used

Manual Review

## Recommendation

FIrst and foremost, try not to give token approval to the Router as you're already giving the router to spend the tokens within the "_transferTokensAndApprove" function.

Not allowing, zero amount of tokens transfer will likely to fix this solution as the "_zapIn()" function checks the post-token balance. 