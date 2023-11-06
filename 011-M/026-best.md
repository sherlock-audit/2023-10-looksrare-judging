Handsome Metal Gorilla

high

# `InfiltrationPeriphery.sol` is useless since the swap router parameters are wrong and the functions will revert all the time
## Summary
The two functions used in the `InfiltrationPeriphery.sol` will revert all the time because the parameters used for the swap router are wrong.
## Vulnerability Detail
`InfiltrationPeriphery.sol` has two functions, `costToHeal` used to see how much it will cost to heal an agent in WETH, by calling `QuoterV2` and `heal` that is used to heal an agent by using  WETH, swapped by using the UniswapV3 router. The problem relies in the fact that the parameters used in the `heal` function to call `exactOutputSingle` in the swap router are wrong. In the `InfiltrationPeriphery.sol` contract the parameters used look like this 
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/interfaces/IV3SwapRouter.sol#L7-L15
while the one from the UniswapV3 docs look like this 
![image](https://github.com/sherlock-audit/2023-10-looksrare-VagnerAndrei26/assets/111457602/5ba1d2d8-9660-4b82-8d52-fcfb419815e9)
as you can see, it is missing the deadline parameters. Because of that, anytime the contract will try to call `exactOutputSingle` with those parameters the call will revert, because the decoding will expect a bigger number of parameters, and it will not find the deadline one to be checked here 
https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/SwapRouter.sol#L207
making the function, and pretty much the whole contract unusable.
## Impact
Impact is a high one since the contract is rendered unusable.
## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/InfiltrationPeriphery.sol#L48-L56
## Tool used

Manual Review

## Recommendation
Add the deadline parameter, so the function will not revert.