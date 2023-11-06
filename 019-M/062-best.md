Quick Silver Stallion

medium

# If all agents are owned by the same owner then there is no point to play the game
## Summary
At any stage of the game, if all agents are owned by the same owner, there is no motivation or financial incentive to continue the game. These additional rounds would result in unnecessary losses of funds for the LOOKS admin, as compensation in LINK tokens for the Chainlink VRF will continue to accumulate as long as the game persists until completion
## Vulnerability Detail
Assume that at a stage of the game, there are 70 active agents, some wounded, and healing agents. If all the active agents are owned by the same owner, the owner can escape 69 of the agents and finish the game in the next round.

Alternatively, if there are 70 active agents, and all 70 are alive, meaning there are no wounded or healing agents, and these 70 agents are controlled by the same owner, then this owner can choose to stall the game as much as possible. This means they do not perform any escapes or healing actions, resulting in the Chainlink VRF continuing to work until the last round. This unnecessarily costs LINK tokens for the LOOKS admin
## Impact
Although the LOOKS admin is paid enough for the keep game played until the last round the above scenario is just causing the LOOKS admin to lose unnecessary funds. It's like playing a game that has already over. Because of this unnecessary LINK token cost for the LOOKS admin I will label this as medium
## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L716-L796
## Tool used

Manual Review

## Recommendation
If there are no healers (or maybe even wounded agents?) and all the active/alive agents are owned by the same address, then make a shortcut and end the game