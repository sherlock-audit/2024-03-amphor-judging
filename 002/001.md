Main Bone Iguana

medium

# The `_zapIn` function may unexpectedly revert due to the incorrect implementation of `_transferTokenInAndApprove`

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