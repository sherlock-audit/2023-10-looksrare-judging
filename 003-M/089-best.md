Sunny Bronze Gecko

medium

# costToHeal() in InfiltrationPeriphery.sol allow easy sandwich and user paying more than healing cost
## Summary
In InfitrationPeriphery.sol any user can calculate how much he must sent eth to heal it's agents, allowing him tu use ETH instead of Looks.
To do this the easiest way and recommended way seems to be using `InfiltrationPeriphery.sol#costToHeal()` that returns the amount of ETH needed to heal some NFT agents.
However `costToHealInLOOKS` is first calculated by making a call to `QuoterV2`. This simulates the transaction directly on the target pool. This is the fundamental problem with this design. If the pool is sandwich attacked then the expected out will also fall.
Amount returned will be false and then when user will submit the false amount of ETH to effectively heal it's agent he will open door for another sandwich attack where surplus amount will be stolen.

## Vulnerability Detail

Example: Assume Alice wants to heal its 10 agents using ETH  : 
- current price of ETH is $1500
- 1 looks = 1$ 
- price to heal one agent = 150$ so price to heal 1 agent in eth should be **0.1 ETH**

There is no slippage added as only `costToHealInLooks` is added (`if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);`). 

This means there is no highest price Alice should get : 

1. Assume that an attacker is observing the mempool. They see the transaction and sandwich attack it. 
2. First the sell ETH into the pool lowering the price to $1300. 
3. When Alice transaction executes, QuoterV2 will quote the price of ETH as $1300 and therefore price to heal in eth will be 0,115 ETH instead of 0,1 for one agent
4. Alice send `0,115 ETH * 5 = 1.15` as msg.value in InfiltrationPeriphery.sol#heal() and she gets sandwiched as again only `amountOut` is needed for this transaction`sqrtPriceLimitX96: 0 //E if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);`
5. Alice paid 0.15 ETH more than what she should have.

Here is the related code : 
```solidity
    function heal(uint256[] calldata agentIds) external payable {
        uint256 costToHealInLOOKS = INFILTRATION.costToHeal(agentIds);
        IV3SwapRouter.ExactOutputSingleParams memory params = IV3SwapRouter.ExactOutputSingleParams({
            tokenIn: WETH,
            tokenOut: LOOKS,
            fee: POOL_FEE, //E 3_000
            recipient: address(this),
            amountOut: costToHealInLOOKS,
            amountInMaximum: msg.value,
            sqrtPriceLimitX96: 0 //E if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
        });

        uint256 amountIn = SWAP_ROUTER.exactOutputSingle{value: msg.value}(params);

        IERC20(LOOKS).approve(address(TRANSFER_MANAGER), costToHealInLOOKS);

        INFILTRATION.heal(agentIds);

        if (msg.value > amountIn) {
            SWAP_ROUTER.refundETH();
            unchecked {
                _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, msg.value - amountIn, gasleft());
            }
        }
    }

    function costToHeal(uint256[] calldata agentIds) external returns (uint256 costToHealInETH) {
        uint256 costToHealInLOOKS = INFILTRATION.costToHeal(agentIds);

        IQuoterV2.QuoteExactOutputSingleParams memory params = IQuoterV2.QuoteExactOutputSingleParams({
            tokenIn: WETH,
            tokenOut: LOOKS,
            amount: costToHealInLOOKS,
            fee: POOL_FEE,
            sqrtPriceLimitX96: uint160(0) //E if (sqrtPriceLimitX96 == 0) amountOutCached = params.amount = costToHealInLOOKS
        });

        (costToHealInETH, , , ) = QUOTER.quoteExactOutputSingle(params);
    }
```
## Impact
- Users healing payment can be sandwich attacked due to ineffective slippage controls

## Code Snippet
- https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/QuoterV2.sol#L197
- https://github.com/Uniswap/v3-periphery/blob/main/contracts/interfaces/IQuoterV2.sol
- https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/InfiltrationPeriphery.sol#L78
- 
## Tool used

Manual Review

## Recommendation
Allow users to specify an min/max sqrtPriceLimitX96. If the pool ever goes above/below that value then revert the call