Fancy Tortilla Elk

medium

# High Costs Incurred Due to Lack of Business Logic in the Protocol
## Summary
In the provided docs, a specific game logic is outlined: "Every 50 blocks (approximately 10 minutes), 0.2% of all active agents are randomly set to a 'wounded' status."

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L1404-L1416C10

This logic implies that if no one heals or escapes, it takes 2,271 rounds to finish the game. During the last 49 rounds, only one agent will be wounded each round. So in the remaining 2,222 rounds, the outcomes are the following:
- in 72 rounds 7 agents wounded
- in 83 rounds 6 agents wounded
- in 100 rounds 5 agents wounded
- in 125 rounds 4 agents wounded
- in 166 rounds 3 agents wounded
- in 250 rounds 2 agents wounded
- in 949 rounds 1 agents wounded

As shown above, in 949 rounds, only one agent is wounded. The cost for each round includes LINK and transaction fees,
These 949 rounds constitute 41.79% of the total rounds and will cost 41.79% of all costs.
If we assume each round costs 3 LINK each LINK $10 (I don't consider transaction fees), the total rounds (2271) will cost 68130, and the 949 rounds cost 28470. If in 949 rounds just one agent heals the cost will be double, around 56940.

## Vulnerability Detail
https://docs.google.com/spreadsheets/d/1gbaUikPH-q3v-pje9rw__UiHbAt7KNPmoAFj0I9sjS4/edit?usp=sharing
## Impact
The protocol incurs significant costs due to this logic.
## Code Snippet
```solidity
        woundedAgentsCount =
            (activeAgents * AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS) /
            ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;
        // At some point the number of agents to wound will be 0 due to round down, so we set it to 1.
        if (woundedAgentsCount == 0) {
            woundedAgentsCount = 1;
        }
```
## Tool used

Manual Review

## Recommendation
To reduce costs and improve efficiency, it is suggested to increase 0.2% after some rounds, but if it isn't feasible, it is recommended to modify the code to select 2 wardens instead of 1. By making this change, the number of rounds will decrease from 949 rounds to 474 rounds. This change will lead to substantial cost savings.
```diff
        woundedAgentsCount =
            (activeAgents * AGENTS_TO_WOUND_PER_ROUND_IN_BASIS_POINTS) /
            ONE_HUNDRED_PERCENT_IN_BASIS_POINTS;
        
+        if (activeAgents < 1000 && activeAgents > 51){ //@audit for agents unders 1000 pick 2 agents
+            woundedAgentsCount = 2;
+        }
        // At some point the number of agents to wound will be 0 due to round down, so we set it to 1.
        if (woundedAgentsCount == 0) {
            woundedAgentsCount = 1;
        }
```

