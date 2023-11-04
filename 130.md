Rare Turquoise Bear

high

# [H-01] '_swap' can break things while in a loop.
## Summary

The [_swap()](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1557C3-L1593C6) function alters the storage and can potentially cause issues when it is used.

## Vulnerability Detail

The [_swap()](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1557C3-L1593C6) is called when we want to `kill` or `escape`. This is done by swapping the indexes and ids of the last Agent in the array by the index and ids of the agent we want to `kill` or `escape`. The issue is that when executed in a loop with pre-fed values, the next iteration is called by the previous value, where it might be potentially altered. 

That means, suppose we swap with two agents, `Agent001`  (index = 69 , id = 7) and `Agent002` (index = 78, id = 10), where Agent001 is the one to be set to `kill` or `escaped` and Agent002 is the agent in last index . The issue is when `Agent002 ` is also next in iteration or coming up in iteration of setting its status as `kill` or escaped. Lets go have a walkthrough when heal is called with both these agents;

When `heal` is called with an array of agents to heal with Agent001 and Agent002 , and ultimately [_healRequestFulfilled](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1335C4-L1339C104) is called, `healingAgentIds` is already set and those ids are used to `heal` agents, suppose Agent001 heal failed and `_swap` is called with Agent002 at `lastIndex`. Now after `_swap`;

Agent001 index = 78, id = 10
Agent002 index = 69, id = 7

So in the next iteration or upcoming iterations, the ` Agent storage agent = agents[78];`, not that 78 was the index of Agent002 originally, but now it is the index of Agent001, so Agent001 gets another shot at redemption and Agent002 is left out, and the amount spent by the ownerOf(Agent002) is wasted. 

This is one example, there can be issues when [_killWoundedAgents](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1477C4-L1480C50) is also called.

## Impact

Loss of amount spent for user in case of heal.

## Code Snippet

https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1557C3-L1594C1

```solidity
 assembly {
            let lastAgentCurrentValue := sload(lastAgentSlot)
            // Replace the last agent's ID with the current agent's ID.
            lastAgentCurrentValue := and(lastAgentCurrentValue, not(AGENT__STATUS_OFFSET))
            lastAgentCurrentValue := or(lastAgentCurrentValue, lastAgentId)
            sstore(currentAgentSlot, lastAgentCurrentValue)

            let lastAgentNewValue := agentId
            lastAgentNewValue := or(lastAgentNewValue, shl(AGENT__STATUS_OFFSET, newStatus))
            sstore(lastAgentSlot, lastAgentNewValue)
        }
```
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1335C3-L1395C6

Notice the loop in which healRequestFulfilled executed
```solidity
  function _healRequestFulfilled(
        uint256 roundId,
        uint256 currentRoundAgentsAlive,
        uint256 randomWord
    ) private returns (uint256 healedAgentsCount, uint256 deadAgentsCount, uint256 currentRandomWord) {
        uint16[MAXIMUM_HEALING_OR_WOUNDED_AGENTS_PER_ROUND_AND_LENGTH]
            storage healingAgentIds = healingAgentIdsPerRound[roundId];
        uint256 healingAgentIdsCount = healingAgentIds[0];

        if (healingAgentIdsCount != 0) {
            HealResult[] memory healResults = new HealResult[](healingAgentIdsCount);

            for (uint256 i; i < healingAgentIdsCount; ) {
                uint256 healingAgentId = healingAgentIds[i.unsafeAdd(1)];
                uint256 index = agentIndex(healingAgentId);
                Agent storage agent = agents[index];

                healResults[i].agentId = healingAgentId;

                // 1. An agent's "healing at" round ID is always equal to the current round ID
                //    as it immediately settles upon randomness fulfillment.
                //
                // 2. 10_000_000_000 == 100 * PROBABILITY_PRECISION
                if (randomWord % 10_000_000_000 <= healProbability(roundId.unsafeSubtract(agent.woundedAt))) {
                    // This line is not needed as HealOutcome.Healed is 0. It is here for clarity.
                    // healResults[i].outcome = HealOutcome.Healed;
                    uint256 lastHealCount = _healAgent(agent);
                    _executeERC20DirectTransfer(
                        LOOKS,
                        0x000000000000000000000000000000000000dEaD,
                        _costToHeal(lastHealCount) / 4
                    );
                } else {
                    healResults[i].outcome = HealOutcome.Killed;
                    _swap({
                        currentAgentIndex: index,
                        lastAgentIndex: currentRoundAgentsAlive - deadAgentsCount,
                        agentId: healingAgentId,
                        newStatus: AgentStatus.Dead
                    });
                    unchecked {
                        ++deadAgentsCount;
                    }
                }

                randomWord = _nextRandomWord(randomWord);

                unchecked {
                    ++i;
                }
            }

            unchecked {
                healedAgentsCount = healingAgentIdsCount - deadAgentsCount;
            }

            emit HealRequestFulfilled(roundId, healResults);
        }

        currentRandomWord = randomWord;
    }
    
   ```
   
   
## Tool used

Manual Review

## Recommendation

Implement checks to see if the agent at `lastIndex`  is called in the upcoming iterations or better would be to cache  and update the storage by `_swap` after execution or after loop is completed.
