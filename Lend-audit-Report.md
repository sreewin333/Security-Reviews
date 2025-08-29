 # [M-1] use of transfer() instead of safeTransfer() causes certain token transfer to fail.
## Summary
The CoreRouter.sol contract allows users to supply token through the supply() function and it transfers the tokens using safeTransferFrom() but when redeeming the tokens through the redeem() function, the function uses the regular transfer() method instead of using safeTransfer(). This can cause non compatible ERC20's like usdt transfer to fail making the redeem function to revert. The team has confirmed the use of usdt in the protocol Whitelisted only (e.g BTC, ETH, USDC, DAI, USDT). ERC20 standard.

## Root Cause
the root cause is the use of using transfer() instead of using safeTransfer() in the redeem function.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CoreRouter.sol#L124

## Internal Pre-conditions
The underlying token should be token like (Non-compatible ERC20) like usdt.
External Pre-conditions
nil

## Attack Path
nil

## Impact
As the contract has no function to recover these tokens and the redeem function reverting, this results in the tokens to stuck permanently in the contract.

## PoC
alice deposits 100 usdt(which is the underlying token) in the CoreRouter.sol contract using the supply function and receives lets say about 100 lTokens
alice decides to redeem the token and calls the function redeem().
The function reverts because it is trying to transfer the usdt(which is non-compatibe ERC20) to alice using the transfer() method instead os using safeTransfer().
Mitigation
use safeTransfer() instead of transfer().