Sunny Bronze Gecko

medium

# If a winner (primary or secondary) forgets to claim it's reward, money will be stuck undefinitely
## Summary
Inflation.sol contract is lacking a recover function in case there is a winner but for example 2 month after the game finished there are still `LOOKS` or `ETH` inside the contract, rendering these tokens stuck forever

## Vulnerability Detail
inflation.sol includes function that allow winners (either last Agent owner or up to 50 last agents) to take their prizes using : 
- `claimGrandPrize()`
- `claimSecondaryPrizes()`

and includes one emergency function that allows `owner` to withdraw funds stuck on contract in case game is stuck, game couldn't start or there is an accounting problem of agents
However `emergencyWithdraw` cannot be called if the game ends normally with one final winner and there is no other function that allow recovering funds if a winner (of the 1st or secondary prize) which either forgets, loses it's private key or is no longer able to claim it's prize which render LOOKS or ETH in inflation.sol stuck forever

## Impact

- money stuck if a winner either forget, lose it's private key or is no longer able to claim it's prize which render LOOKS or ETH in inflation.sol stuck forever

## Code Snippet

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L528

## Tool used

Manual Review

## Recommendation
Implement a `withdrawAll()` function that ,if for example 2 months pass and there is still money inside,  allows contract `owner` or final winner (`ownerOf(agents[1].agentId`) to recover all the funds in the contract 