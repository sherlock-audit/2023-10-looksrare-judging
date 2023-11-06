Quick Silver Stallion

high

# New round might be not possible because of high number of loops
## Summary
When the `NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS` is reached the new round will start killing the wounded people either from the round 1 or the round from `currentRoundId.unsafeSubtract(ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD)` depending on the current round state. If the `ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD` is not reached but the active agents are lesser than 50 then the new round will start killing wounded agents from round 1 to current round. This process can be very gas extensive and cause the function to revert because the gas provided will never be enough. More details on the scenario will be provided in the below section. 
## Vulnerability Detail
Assume we have 10000 agents (total supply), `AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS = 20` and `ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD = 48` note that those constant values are directly taken from the actual test file so I am assuming these values are considered to be used. 

Now starting from the round 1 to round 48 the wounded agents will not be killed. So assuming the above values let's see how many wounded agents we will have at the round 48 assuming that there are no heals on the wounded agents.

First this is the formula for wounded agents in a given round:
```solidity
woundedAgentsCount =
            (activeAgents * AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS) /
            ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;
```
Now, let's plug this formula to Remix to see what would be the wounded agents from round 1 to round 48 assuming there are no heals.

```solidity
function someMath() external  {
      uint activeAgents = 10_000;
      uint AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS = 20;
      uint ONE_HUNDRED_PERCENT_IN_BASIS_POINTS = 10_000;
      for (uint i; i < 48; ++i) {
        uint res = (activeAgents * AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS) / ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;

        activeAgents -= res;

        r.push(res);
      }
  }
```
when we execute the above code snippet we will see the array result and the sum of wounded agents as follows:
uint256[]: 20,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,19,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18,18
sum = 892

That means that in 48 rounds there will be 892 wounded agents assuming none of them healed. Since the wounded agents are not counted as "active" agents the total active agents would be 10_000 - 892 = 9108

Now, let's assume at round 48 the active agent count is < 50, that can happen if agents are escapes in that interval + the wounded are not counted. That means we have 892 agents wounded and there are 9060 agents escaped. Which means 10000 - (892 + 9060) = 48 which is lesser than 50 and the next time we call the startNewRound() we will execute the following code:
```solidity
if (activeAgents <= NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS) {
            uint256 woundedAgents = gameInfo.woundedAgents;

            if (woundedAgents != 0) {
                uint256 killRoundId = currentRoundId > ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD
                    ? currentRoundId.unsafeSubtract(ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD)
                    : 1;

                // @audit totalSupply() - gameInfo.deadAgents - gameInfo.escapedAgents;
                uint256 agentsRemaining = agentsAlive();
                uint256 totalDeadAgentsFromKilling;
                while (woundedAgentIdsPerRound[killRoundId][0] != 0) {
                    uint256 deadAgentsFromKilling = _killWoundedAgents({
                        roundId: killRoundId,
                        currentRoundAgentsAlive: agentsRemaining // 200
                    });
                    unchecked {
                        totalDeadAgentsFromKilling += deadAgentsFromKilling; // 2
                        agentsRemaining -= deadAgentsFromKilling; // 148
                        ++killRoundId;
                    }
                }
```

As we can see the while loop will be executed starting from roundId 0 to 48 and starts killing the 892 wounded agents. However, this is extremely high number and the amount of SSTORE and SLOAD is too much. Calling this function will fail because it is not possible to do the loop 48 times for 892+ storage changes. This will result the game to stop and not be able to level up to the next round.
## Impact
The game will be impossible to play if the above condition is satisfied which is not an extreme case because escapes can happen randomly and easily. The values for ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD and total supply does not really change this because of the escapes are not limited. Therefore, I'll label this as high.
## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L579-L651
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1096-L1249
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1404-L1463
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1477-L1508
## Tool used

Manual Review

## Recommendation
