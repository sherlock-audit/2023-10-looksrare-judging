Polite Rose Beaver

high

# _killWoundedAgents - re-wounded healed agent die when ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD passed because of not checking woundedAt
## Summary

If an agent healed is wounded again, the agent will die from the previous wound that was healed. The user spends LOOKS tokens to heal and success to heal, but as the result, the agent will die.

## Vulnerability Detail

The `_killWoundedAgents` function only checks the status of the agent, not when it was wounded. 

```solidity
    function _killWoundedAgents(
        uint256 roundId,
        uint256 currentRoundAgentsAlive
    ) private returns (uint256 deadAgentsCount) {
        ...
        for (uint256 i; i < woundedAgentIdsCount; ) {
            uint256 woundedAgentId = woundedAgentIdsInRound[i.unsafeAdd(1)];

            uint256 index = agentIndex(woundedAgentId);
@>          if (agents[index].status == AgentStatus.Wounded) {
                ...
            }

            ...
        }

        emit Killed(roundId, woundedAgentIds);
    }
```

So when `fulfillRandomWords` kills agents that were wounded and unhealed at round `currentRoundId - ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD`, it will also kill the agent who was healed and wounded again after that round.

Also, since `fulfillRandomWords` first draws the new wounded agents before kills agents, in the worst case scenario, agent could die immediately after being wounded in this round.

```solidity
if (activeAgents > NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS) {
@>  uint256 woundedAgents = _woundRequestFulfilled(
        currentRoundId,
        currentRoundAgentsAlive,
        activeAgents,
        currentRandomWord
    );

    uint256 deadAgentsFromKilling;
    if (currentRoundId > ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD) {
@>      deadAgentsFromKilling = _killWoundedAgents({
            roundId: currentRoundId.unsafeSubtract(ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD),
            currentRoundAgentsAlive: currentRoundAgentsAlive
        });
    }
```

This is the PoC test code. You can add it to the Infiltration.fulfillRandomWords.t.sol file and run it.

```solidity
function test_poc() public {

    _startGameAndDrawOneRound();

    uint256[] memory randomWords = _randomWords();
    uint256[] memory woundedAgentIds;

    for (uint256 roundId = 2; roundId <= ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD + 1; roundId++) {

        if(roundId == 2) { // heal agent. only woundedAgentIds[0] dead.
            (woundedAgentIds, ) = infiltration.getRoundInfo({roundId: 1});
            assertEq(woundedAgentIds.length, 20);

            _drawXRounds(1);

            _heal({roundId: 3, woundedAgentIds: woundedAgentIds});

            _startNewRound();

            // everyone except woundedAgentIds[0] is healed
            uint256 agentIdThatWasKilled = woundedAgentIds[0];

            IInfiltration.HealResult[] memory healResults = new IInfiltration.HealResult[](20);
            for (uint256 i; i < 20; i++) {
                healResults[i].agentId = woundedAgentIds[i];

                if (woundedAgentIds[i] == agentIdThatWasKilled) {
                    healResults[i].outcome = IInfiltration.HealOutcome.Killed;
                } else {
                    healResults[i].outcome = IInfiltration.HealOutcome.Healed;
                }
            }

            expectEmitCheckAll();
            emit HealRequestFulfilled(3, healResults);

            expectEmitCheckAll();
            emit RoundStarted(4);

            randomWords[0] = (69 * 10_000_000_000) + 9_900_000_000; // survival rate 99%, first one gets killed

            vm.prank(VRF_COORDINATOR);
            VRFConsumerBaseV2(address(infiltration)).rawFulfillRandomWords(_computeVrfRequestId(3), randomWords);

            for (uint256 i; i < woundedAgentIds.length; i++) {
                if (woundedAgentIds[i] != agentIdThatWasKilled) {
                    _assertHealedAgent(woundedAgentIds[i]);
                }
            }

            roundId += 2; // round 2, 3 used for healing
        }

        _startNewRound();

        // Just so that each round has different random words
        randomWords[0] += roundId;

        if (roundId == ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD + 1) { // wounded agents at round 1 are healed, only woundedAgentIds[0] was dead.
            (uint256[] memory woundedAgentIdsFromRound, ) = infiltration.getRoundInfo({
                roundId: uint40(roundId - ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD)
            });

            // find re-wounded agent after healed
            uint256[] memory woundedAfterHeal = new uint256[](woundedAgentIds.length);
            uint256 totalWoundedAfterHeal;
            for (uint256 i; i < woundedAgentIds.length; i ++){
                uint256 index = infiltration.agentIndex(woundedAgentIds[i]);
                IInfiltration.Agent memory agent = infiltration.getAgent(index);
                if (agent.status == IInfiltration.AgentStatus.Wounded) {
                    woundedAfterHeal[i] = woundedAgentIds[i]; // re-wounded agent will be killed
                    totalWoundedAfterHeal++;
                }
                else{
                    woundedAfterHeal[i] = 0; // set not wounded again 0
                }

            }
            expectEmitCheckAll();
            emit Killed(roundId - ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD, woundedAfterHeal);
        }

        expectEmitCheckAll();
        emit RoundStarted(roundId + 1);

        uint256 requestId = _computeVrfRequestId(uint64(roundId));
        vm.prank(VRF_COORDINATOR);
        VRFConsumerBaseV2(address(infiltration)).rawFulfillRandomWords(requestId, randomWords);
    }
}
```

## Impact

The user pays tokens to keep the agent alive, but agent will die even if agent success to healed. The user has lost tokens and is forced out of the game.

## Code Snippet

[https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1489](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1489)

[https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1130-L1143](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1130-L1143)

## Tool used

Manual Review

## Recommendation

Check woundedAt at `_killWoundedAgents` 

```diff
    function _killWoundedAgents(
        uint256 roundId,
        uint256 currentRoundAgentsAlive
    ) private returns (uint256 deadAgentsCount) {
        ...
        for (uint256 i; i < woundedAgentIdsCount; ) {
            uint256 woundedAgentId = woundedAgentIdsInRound[i.unsafeAdd(1)];

            uint256 index = agentIndex(woundedAgentId);
-           if (agents[index].status == AgentStatus.Wounded) {
+           if (agents[index].status == AgentStatus.Wounded && agents[index].woundedAt == roundId) {
                ...
            }

            ...
        }

        emit Killed(roundId, woundedAgentIds);
    }
```