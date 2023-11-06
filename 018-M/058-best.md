Quick Silver Stallion

high

# Race condition on escaping
## Summary
At any point in the game, agents are free to 'escape' from the game by claiming a portion of the prize, as long as the game is not awaiting the Chainlink call. Escapes can only occur after the last Chainlink fulfillment call has been made, and a time interval of approximately one day has passed since then. During this time interval, agents who are still active in the game can choose to escape. The prize they can claim upon escaping is calculated based on the number of 'activeAgents.' The fewer active agents there are, the larger the escape prize will be. However, there is a race condition during this interval. The first agent to exit will receive a smaller reward, which may incentivize users who wish to escape that round to wait until the very last moment to ensure they can receive a larger prize.
## Vulnerability Detail
Let's illustrate this with an example: Suppose there are 40 active agents in the game, and the Chainlink fulfill function has just been called. Since `BLOCKS_PER_ROUND` is approximately 1 day and also randomness is checked that at least 1 day is passed in terms of "block.timestamp", the next time the 'startNewRound()' function will be called is in one day. During this time interval, Alice decides to escape the game. However, Alice wants to take advantage of the fact that the fewer active agents there are, the larger the prize she can claim upon escaping. To achieve this, Alice runs a bot that front-runs the 'startNewRound()' function, ensuring that she is the last person to exit that round and, therefore, potentially receives a larger escape prize.

Additionally, since Alice is among the top 50 surviving agents, she will receive a larger share of the secondary prize when she escapes. Therefore, it is in Alice's best interest to wait as long as possible before exiting a round.
## Impact
Users that are technically able to run a front-running bot or monitor the time to execute the escape will always have more advantage. This will make non technical users to always get the worse outcome when escaping in a round. Since this is applicable for any round I will label this as high because it will create unfairness for every round for the agents that wants to escape.
## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L579-L596
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L677-L711
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L740-L752
## Tool used

Manual Review

## Recommendation
Check the round id when escaping, agents escaping in same round should be receiving the escaping reward same in order to prevent the gaming of escaping.