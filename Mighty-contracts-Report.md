
 # [H-1] Wrong price feed Calculation 

 ## Description
This is from the official pyth documentation
https://www.pyth.network/blog/pyth-root-cause-analysis

"The on-chain Pyth program uses a fixed-point number representation, where each number is represented as the combination of an integer plus a number of decimal points called the exponent. For example, $52.21 might be represented as 52210 and an exponent of 10^-3".

The `getPythprice()` function in the `PrimaryOracle.sol` returns the `price` of the token without accounting in the `exponent` and `decimals precision` and is used in multiple functions in the `ShadowRangeVault.sol` contract to fetch the value of tokens which will result in completely wrong calculations.

## impact

results in completely wrong calculations when calculating the token values essentially breaking the core functionality of the protocol.

can cause incorrect liquidations.

## Poc

suppose,

token0=ETH and token1=DAI and 1ETH=10,000$ and expo = -6 for both the tokens.
a user opens a position with amount0=100 ETH and amount1=100DAI.

the `getpositionvalue()` will calculate the value of the position as, 
(100*10000000000)/1e18 + (100*1000000)/1e18= 0.000001+0.0000000001=0.0000010001
which is completely wrong 

 it should be,

 (100*10000000000*1e12)/1e18 + (100*1000000*1e12)/1e18 = 1000000+100 =1000100. 

 the difference is `1e12X` below the actual value. 

 # Reccomendation

 change this in `PrimaryPriceOracle.sol`

 ```diff
 function getPythPrice(address token) public view returns (uint256) {
        bytes32 priceId = pythPriceIds[token];
        if (priceId == bytes32(0)) {
            revert("PriceId not set");
        }
        PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);

        require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");

        uint256 price = uint256(uint64(priceStruct.price));

        return price;
    }
```
 # [H-2] Ignoring confidence level in price feed lead to wrong liquidations.

## Description
we are well aware that crypto is known for its high volatality markets and price of the asset can change in an instant.
The pyth price Oracle submits the price of an asset with certain uncertainity with its confidence intervals.This is from the official pyth price feeds docs 
"At every point in time, Pyth publishes both a price and a confidence interval for each product. For example, Pyth may publish the current price of bitcoin as $50000 ± $10. Pyth publishes a confidence interval because, in real markets, there is no one single price for a product".

In the `getPythPrice()` function in the `PrimaryOracle.sol` contract, the price of a token is checked without accounting for its confidence interval.

## impact

lets say price of ETH=2000$ and Alice deposits 1ETH in the `ShadowRangeVault.sol` contract for ETH/DAI pair and borrows 1500 DAI.

the liquidation threshold from the docs are for "Stablecoin pairs: 80% Debt Ratio","General asset pairs: 70% Debt Ratio".
The pyth price oracle will report a price of ETH=2000$ and confidence:+- $300. (actual price in between 1700$ and 2300$).

this means max borrow is 2000*80/100=1600$. which means that alice position is healthy.(100$ above threshold).

Now suppose the value of ETH suddenly drops to 1700$ due to market conditions.
Pyth would still read the price of ETH=2000$ with the same confidence ie,=-300$

this means that the collateral value of Alice now becomes 1700*80/100=1360$. which means Alice's position is not healthy and should be liquidated (140$ below threshold).Alice closes her position by giving back the pool tokens and getting back the 1 ETH collateral.In reality Alice should'nt have received back the full amount of collateral that she deposited that is because confidence not being checked results in Alice's position to still being considered as healthy in which it isn't.This results in the accumulation of Bad debt in the protocol and the protocol or other users have to lose thier funds to remove this bad debt.

This could also result in the liquidation of genuine safe positions

consider the value of ETH now is 1875$ and now the loan to value becomes 1500/1875*100=80% which is the borderline threshold for liquidation and if the price of ETH drops below 1875 lets say about 1860$ even though this price is within the confidence range Alice's position becomes 80.6% and liquidation is triggered,but in reality the price could be way above 1860$.This results in alice getting liquidated in an unfair manner and her collateral is seized for a discount for the liquidators.

## Reccomendation

check for the price confidence and revert if the price confidence is way above or below some threshold (eg:-10% of the returned price).

make these changes in the `getPythPrice()` function in the `PrimaryPriceOracle.sol` contract.
```diff
+   //maxmaxAllowedConfidence as required by the protocol 
+        require(priceStruct.conf>0,"confidence is zero");
+        require(priceStruct.confidence <= maxAllowedConfidence, "Confidence too high");
+     uint256 safePrice = uint256(uint64(priceStruct.price)) - uint256(uint64(priceStruct.conf));
+        return safePrice;
```
