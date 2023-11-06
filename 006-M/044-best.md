Scruffy Beige Mantaray

medium

# Wound agent can't invoke heal in the next round
## Summary
According to the document:

> if a user dies on round 12. The first round they can heal is round 13
However incorrect current round id check led to users being unable to invoke the `heal` function in the next round.

## Vulnerability Detail 
Assume players being marked as wounded in the round `12` , players cannot invoke  `heal` in the next round 13

```solidity
    function test_heal_in_next_round_v1() public {
        _startGameAndDrawOneRound();

        _drawXRounds(11);


        (uint256[] memory woundedAgentIds, ) = infiltration.getRoundInfo({roundId: 12});

        address agentOwner = _ownerOf(woundedAgentIds[0]);
        looks.mint(agentOwner, HEAL_BASE_COST);

        vm.startPrank(agentOwner);
        _grantLooksApprovals();
        looks.approve(TRANSFER_MANAGER, HEAL_BASE_COST);

        uint256[] memory agentIds = new uint256[](1);
        agentIds[0] = woundedAgentIds[0];

        uint256[] memory costs = new uint256[](1);
        costs[0] = HEAL_BASE_COST;

        //get gameInfo
        (,,,,,uint40 currentRoundId,,,,,) = infiltration.gameInfo();
        assert(currentRoundId == 13);

        //get agent Info
        IInfiltration.Agent memory agentInfo = infiltration.getAgent(woundedAgentIds[0]);
        assert(agentInfo.woundedAt == 12);

        //agent can't invoke heal in the next round.
        vm.expectRevert(IInfiltration.HealingMustWaitAtLeastOneRound.selector);
        infiltration.heal(agentIds);
    }
```

## Impact
User have to wait for 1 more round which led to the odds for an Agent to heal successfully start at 99% at Round 1 reduce to 98% at Round 2.

## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L843#L847

```solidity
    // No need to check if the heal deadline has passed as the agent would be killed
    unchecked {
        if (currentRoundId - woundedAt < 2) {
            revert HealingMustWaitAtLeastOneRound();
        }
    }
```

## Tool used

Manual Review

## Recommendation
```solidity
    // No need to check if the heal deadline has passed as the agent would be killed
    unchecked {
-       if (currentRoundId - woundedAt < 2) {
-       if (currentRoundId - woundedAt < 1) {
            revert HealingMustWaitAtLeastOneRound();
        }
    }
```