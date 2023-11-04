Damaged Ocean Mantaray

high

# Agents with Healing Opportunity Will Be Terminated Directly if The `escape` Reduces activeAgents to the Number of `NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS` or Fewer
## Summary

Wounded Agents face the risk of losing their last opportunity to heal and are immediately terminated if certain Active Agents decide to escape.

## Vulnerability Detail

In each round, agents have the opportunity to either `escape` or `heal` before the `_requestForRandomness` function is called. However, the order of execution between these two functions is not specified, and anyone can be executed at any time just before `startNewRound`. Typically, this isn't an issue. However, the problem arises when there are only a few Active Agents left in the game.

On one hand, the `heal` function requires that the number of `gameInfo.activeAgents` is greater than `NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS`.

```solidity
    function heal(uint256[] calldata agentIds) external nonReentrant {
        _assertFrontrunLockIsOff();
//@audit If there are not enough activeAgents, heal is disabled
        if (gameInfo.activeAgents <= NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS) {
            revert HealingDisabled();
        }
```


On the other hand, the `escape` function will directly set the status of agents to "ESCAPE" and reduce the count of `gameInfo.activeAgents`.

```solidity
   function escape(uint256[] calldata agentIds) external nonReentrant {
        _assertFrontrunLockIsOff();

        uint256 agentIdsCount = agentIds.length;
        _assertNotEmptyAgentIdsArrayProvided(agentIdsCount);

        uint256 activeAgents = gameInfo.activeAgents;
        uint256 activeAgentsAfterEscape = activeAgents - agentIdsCount;
        _assertGameIsNotOverAfterEscape(activeAgentsAfterEscape);

        uint256 currentRoundAgentsAlive = agentsAlive();

        uint256 prizePool = gameInfo.prizePool;
        uint256 secondaryPrizePool = gameInfo.secondaryPrizePool;
        uint256 reward;
        uint256[] memory rewards = new uint256[](agentIdsCount);

        for (uint256 i; i < agentIdsCount; ) {
            uint256 agentId = agentIds[i];
            _assertAgentOwnership(agentId);

            uint256 index = agentIndex(agentId);
            _assertAgentStatus(agents[index], agentId, AgentStatus.Active);

            uint256 totalEscapeValue = prizePool / currentRoundAgentsAlive;
            uint256 rewardForPlayer = (totalEscapeValue * _escapeMultiplier(currentRoundAgentsAlive)) /
                ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;
            rewards[i] = rewardForPlayer;
            reward += rewardForPlayer;

            uint256 rewardToSecondaryPrizePool = (totalEscapeValue.unsafeSubtract(rewardForPlayer) *
                _escapeRewardSplitForSecondaryPrizePool(currentRoundAgentsAlive)) / ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;

            unchecked {
                prizePool = prizePool - rewardForPlayer - rewardToSecondaryPrizePool;
            }
            secondaryPrizePool += rewardToSecondaryPrizePool;

            _swap({
                currentAgentIndex: index,
                lastAgentIndex: currentRoundAgentsAlive,
                agentId: agentId,
                newStatus: AgentStatus.Escaped
            });

            unchecked {
                --currentRoundAgentsAlive;
                ++i;
            }
        }

        // This is equivalent to
        // unchecked {
        //     gameInfo.activeAgents = uint16(activeAgentsAfterEscape);
        //     gameInfo.escapedAgents += uint16(agentIdsCount);
        // }
```

Threrefore, if the `heal` function is invoked first then the corresponding Wounded Agents will be healed in function `fulfillRandomWords`. If the `escape` function is invoked first and the number of `gameInfo.activeAgents` becomes equal to or less than `NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS`, the `heal` function will be disable. This obviously violates the fairness of the game.

**Example**

Consider the following situation:

After Round N, there are 100 agents alive. And, **1** Active Agent wants to `escape` and **10** Wounded Agents want to `heal`.
- Round N: 
  - Active Agents: 51
  - Wounded Agents: 49
  - Healing Agents: 0

According to the order of execution, there are two situations.
**Please note that the result is calculated only after `_healRequestFulfilled`, so therer are no new wounded or dead agents**

First, invoking `escape` before `heal`. 
`heal` is disable and all Wounded Agents are killed because there are not enough Active Agents.
- Round N+1: 
  - Active Agents: 50
  - Wounded Agents: 0
  - Healing Agents: 0
 
Second, invoking `heal` before `escape`.
Suppose that `heal` saves **5** agents, and we got:
- Round N+1:
  - Active Agents: 55
  - Wounded Agents: 39
  - Healing Agents: 0

Obviously, different execution orders lead to drastically different outcomes, which affects the fairness of the game.

## Impact

If some Active Agents choose to escape, causing the count of `activeAgents` to become equal to or less than `NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS`, the Wounded Agents will lose their final chance to heal themselves. 

This situation can significantly impact the game's fairness. The Wounded Agents would have otherwise had the opportunity to heal themselves and continue participating in the game. However, the escape of other agents leads to their immediate termination, depriving them of that chance.

## Code Snippet

Heal will be disabled if there are not enout activeAgents.
[Infiltration.sol#L804](https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L804)

Escape will directly reduce the activeAgents.
[Infiltration.sol#L769](https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L769)

## Tool used

Manual Review

## Recommendation

It is advisable to ensure that the `escape` function is always called after the `heal` function in every round. This guarantees that every wounded agent has the opportunity to heal themselves when there are a sufficient number of `activeAgents` at the start of each round. This approach can enhance fairness and gameplay balance.