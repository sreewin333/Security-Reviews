# [H-1]`currentWithheldETH` not updated correctly after unstaking in `stPlumeMinter.sol` contract

## Finding Description

In the `unstake(uint256 amount)` function in the `stPlumeMinter.sol` contract, when a user request to unstake ether from the contract,if the requested amount by the user can be covered with the `currentWithheldETH` a withdrawalRequest is made by the function,but the problem is that  the `currentWithheldETH` is not updated in the function call.The function correctly updates the `currentWithheldETH` if the contract cant cover the amount with its `currentWithheldETH` in the `else` part of the function but fails to do so in the above `if` block.
## Impact Explanation
A mismatch between the actual and outdated (non-updated) state of the `currentWithheldETH` variable can lead to internal accounting inconsistencies. For instance, if a user unstakes an amount that can be covered by `currentWithheldETH`, but the variable hasn't been properly updated, subsequent users may rely on the stale value. This can cause the contract to incorrectly assume it has sufficient liquidity to cover new unstake requests, leading to unexpected reverts when, in reality, the required ETH is not available.

## Likelihood Explanation
Likelihood is high because if the `currentWithheldETH` is greater than or equall to the amount that is being unstaked by a user this issue can happen.



## Recommendation
update the `currentWithheldETH` amount after a request is being made 

```diff
    function unstake(uint256 amount) external nonReentrant returns (uint256 amountUnstaked) {
     if (currentWithheldETH >= amount) {
            amountUnstaked = amount;
            cooldownTimestamp = block.timestamp + 1 days;
+             currentWithheldETH -= amount;
             }

    }
```
# [M-1]Immature return of validator when selecting next validator to stake

## Summary
The `getNextValidator(uint256 depositAmount)` function in the `stPlumeMinter.sol` contract prematurely returns a validator even if there is no staking capacity available for the validator.

## Finding Description

 inside this function, the main problem is this conditional statement `if (info.maxCapacity != 0 || totalStaked < info.maxCapacity)`.Because this is an or conditional, if the `info.maxCapacity != 0` is successful(which will be true for most validators),the function will prematurely return the validator without checking if there is sufficient capacity available for the validator for staking.so when a condition where the `totalStaked` amount of the validator is equall to the `maxCapacity`,the function will return `(validatorId,0)` without returning a validator that has the capacity to stake ether.

## Impact Explanation
The function prematurely returns a validator with no staking capacity eventually causing the staking mechanism to revert.

## Likelihood Explanation
This is very likely to happen if the totalStaked amount become equall to the maxCapacity of the validator, therefore the likelihood is high.

## Proof of Concept
imagine a scenario where a user is trying to deposit some amount of ether to stake to the validator and the `totalStaked` eth amount of the validator is filled and it reached the validator's `maxCapacity` making them equall.

1. A user sends the required amount of ether to the `stPlumeMinter.sol` contract 
2. the `receive` function is triggered and the internal `_submit(address recipient)` function is called which internally calls the `depositEther(uint256 _amount)` function.
3. inside that function, in this while loop `while (remainingAmount > 0) ` it calls the `getNextValidator(uint256 depositAmount)`
4. As the `totalStaked` and `maxCapacity` of the next validator is the same, the function prematurely returns a validator with no staking capacity (0).
5. the local variable`depositSize` in the function `depositEther(uint256 _amount)` is set to 0 and 0 amount is staked to the staking contract rather than the actual amount resulting in a revert. 

 the fork test had only one validator that that had max(uint256) value capacity.I couldn't add a Validator that had a legitimate capacity in order for the test to work so i had to create a `MockPlumStaking.sol` contract for testing.It inherits from the provide interface `IPlumeStaking.sol and shows a real world situation of a validator with a max capacity not set to max uint256.` 

these are the changes that i have done for this test in the `MockPlumStaking.sol` contract

```solidity 
contract MockPlumStaking is IPlumeStaking {
  error cannotbeZero();
//fake stake to just showcase the revert when the msg.value is 0
  function stake(uint16 validatorId) external payable returns (uint256) {
        if (msg.value == 0) revert cannotbeZero();
    }
//mock function to get a fake validator info with maximum capacity
    function getValidatorInfo(uint16 validatorId)
        external
        view
        returns (PlumeStakingStorage.ValidatorInfo memory info, uint256 totalStaked, uint256 stakersCount)
    {
        info = PlumeStakingStorage.ValidatorInfo(
            1, 0, 10e18, address(0), address(0), "address(0)", "address(0)", address(0), true, false, 10e18
        );
        totalStaked = 10e18;
        stakersCount = 0;
        (info, totalStaked, stakersCount);
    }
//mock function that retrives a validator stats
   function getValidatorStats(uint16 validatorId)
        external
        view
        returns (bool active, uint256 commission, uint256 totalStaked, uint256 stakersCount)
    {
        active = true;
        commission = 0;
        totalStaked = 10e18;
        stakersCount = 0;
        (active, commission, totalStaked, stakersCount);
    }

    function getMinStakeAmount() external view returns (uint256) {
        uint256 amount = 1e18;
        return amount;
    }
}

```
modified the `stPlumMinter.sol ` contract to use MockPlumStaking

```solidity 
import {MockPlumStaking} from "../../test/MockPlumStaking.sol";

contract StPlumeMinterForkTest is Test {

    function setUp() public {
        mockPlumeStaking = new MockPlumStaking();
//rest of the code
}
//paste the function here

    function test_RevetsWhenMaxCapacity() external {
        vm.expectRevert(MockPlumStaking.cannotbeZero.selector);
        (bool success,) = address(minter).call{value: 10 ether}("");
        require(success, "failed");
    }
}

```
Logs
```javascript
Ran 1 test for test/fork/stPlumeMinter.t.sol:StPlumeMinterForkTest
[PASS] test_RevetsWhenMaxCapacity()

```


## Recommendation
Handle the situation by skipping validators that has their `maxCapacity` filled to its maximum.

modify the `getNextValidator(uint256 depositAmount) `function to include these changes
```diff
    function getNextValidator(uint256 depositAmount) public returns (uint256 validatorId, uint256 capacity) {
            ////////////////////////////
            ///////////////////////////

            if (!active) continue;
            (PlumeStakingStorage.ValidatorInfo memory info, uint256 totalStaked,) =
                plumeStaking.getValidatorInfo(uint16(validatorId));
+            if (totalStaked == info.maxCapacity) continue;

            //additional logic under this 


    }

```
https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=86,86

https://cantina.xyz/code/c160af78-28f8-47f7-9926-889b3864c6d8/Liquid-Staking/src/stPlumeMinter.sol?lines=93,93


