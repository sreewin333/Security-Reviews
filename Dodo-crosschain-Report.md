# [H-1]Anyone can claim refunds in GatewayTransferNative.sol
## Summary
A user can withdraw tokens to their native chain with the help of this function withdrawToNativeChain in the GatewayTransferNative.sol.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L530

This triggers the gateway contract and thus withdrawAndCall()
invoked.The gateway contract has a mechanism so that if the call failed and callOnRevert is true, the Gateway invokes the onRevert() function(if the onRevert() itself fails,it calls the onAbor()function).Thus a refund is processed and it is set inside this mapping refundInfos[externalId] = refundInfo so that a verified bot or the user can claim the refund. But because of a faulty check inside the claimRefund() function, anyone can claim the refunds of any users.

## Root Cause
The main problem is the claimRefund() function.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L661

First it sets the receiver as the msg.sender.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L664

It then checks if the wallet address length is 20 bytes long. If it is, it then sets the receiver as the wallet address.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L665

But the main problem is that the contract also process solana transactions and solana addresses are 32 bytes long so this check is bypassed.

Then the function requires that caller is a verifed bot or the receiver.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L668

This check is flawed because, if the receiver is a Solana wallet address, there's no verification that the caller is the owner of that Solana address. The function merely checks if the receiver equals msg.sender, which has already been set earlier.

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewayTransferNative.sol#L664

Therefore any person can call the claimRefund with a valid externalId and steal the tokens of the failed transcation from the users.

## Internal Pre-conditions
The cross-chain transaction must revert
The refund wallet address must be a solana address.
External Pre-conditions
nil

## Attack Path
Alice initiates a cross chain transaction of transferring 100 usdt.
The transaction fails and the Gateway contract initiates the withdrawal process by calling the onRevert function and alice's refund info is stored in the refundInfos[externalId] mapping.
Bob sees that Alice's transaction has failed by monitoring the blockchain and gets the externalId of Alice's transaction.
Bob calls the claimRefund() function with alice's externalId and steals the tokens of alice.
Impact
This results in the direct loss of funds to the users as anyone can monitor the blockchain for failed transcation of users and also by monitoring the blockchain, any user can get the externalId of users and claim the refunds of these failed transactions.

## PoC
modify the test inside the GatewayTransferNative.t.sol::test_ZOnAbort slightly for this test.

to run this test ,run forge test --mt test_ZOnAbort_TransferNative --fork-url https://zetachain-evm.blockpi.network/v1/rpc/public -vv

```solidity
   function test_ZOnAbort_TransferNative() public {  
        bytes32 externalId = keccak256(abi.encodePacked(block.timestamp));  
        uint256 amount = 100 ether;  
        token1Z.mint(address(gatewayTransferNative), amount);  
        //modified this test so that the refund wallet address is a solana Address  
        bytes memory solAddress = abi.encodePacked("DrexsvCMH9WWjgnjVbx1iFf3YZcKadupFmxnZLfSyotd");  
        vm.prank(address(gatewayZEVM));  
        gatewayTransferNative.onAbort(  
            AbortContext({  
                sender: abi.encode(address(this)),  
                asset: address(token1Z),  
                amount: amount,  
                outgoing: false,  
                chainID: 7000,  
                revertMessage: bytes.concat(externalId, solAddress)  
            })  
        );  
  
        // vm.expectRevert();  
        // gatewayTransferNative.claimRefund(externalId);  
  
        // vm.expectRevert();  
        // vm.prank(user2);  
        // gatewayTransferNative.claimRefund(bytes32(0));  
  
        //vm.prank(user2);  
  
        address attacker = makeAddr("attacker");  
        vm.prank(attacker);  
        gatewayTransferNative.claimRefund(externalId);  
        console.log("balance of attacker:", token1Z.balanceOf(attacker));  
  
        // vm.expectRevert();  
        // vm.prank(user2);  
        // gatewayTransferNative.claimRefund(externalId);  
  
        // assertEq(token1Z.balanceOf(user2), amount);  
    } 
    ``` 
```javascript
Logs:

Ran 1 test for test/GatewayTransferNative.t.sol:GatewayTransferNativeTest  
[PASS] test_ZOnAbort_TransferNative() (gas: 182784)  
Logs:  
  balance of attacker: 100000000000000000000  
  ```
As you can see from the test an attacker was able to claim the refunds from a valid externalId as demonstrated in the above test.

# Mitigation
modify the claimRefund() function so that only a verified bot or the owner of the wallet address to claim the refund rather than allowing anyone to call the claimRefund() function.


 # [M-1] Use of transfer() instead of safeTransfer() can cause certain Transactions to fail.
## Description
In the GatewaySend.sol, the gateway contract calls the onCall() function to transfer eth or tokens to the evm wallet address. But the function uses regular transfer() method instead of using safeTransfer() to transfer the tokens.some tokens like usdt in ethereum does'nt return bool values Thus causes the transaction to revert.

## Root cause
https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewaySend.sol#L372

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewaySend.sol#L359

https://github.com/Skyewwww/omni-chain-contracts/blob/2fe44d3da76b721e4d32addfecb04ca97a39cb0d/contracts/GatewaySend.sol#L317

## Impact
Because the gateway contract does not properly handle non-compliant tokens like USDT, all transfers involving these tokens will fail. This means the protocol will be unable to support one of the most widely used stablecoins (USDT), severely limiting its usability.

## mitigation
uses openzeppelin's SafeERC20 library to handle these cases.