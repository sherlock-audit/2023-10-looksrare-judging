Fun Aegean Halibut

high

# Winning agent id may be uninitialized when game is over, locking grand prize
## Summary
In the `Infiltration` contract, the `agents` mapping holds all of the agents structs, and encodes the ranking of the agents (used to determine prizes at the end of the game).

This mapping records are lazily initialized when two agents are swapped (an agent is either killed or escapes):
- The removed agent goes to the end of currently alive agents array with the status `Killed/Escaped` and its `agentId` is set  
- The last agent of the currently alive agents array is put in place of the previously removed agent and its `agentId` is set

This is the only moment when the `agentId` of an agent record is set.

This means that if the first agent in the array ends up never being swapped, it keeps its agentId as zero, and the grand prize is unclaimable. 

## Vulnerability Detail
We can see in the implementation of `claimGrandPrize` that:
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L658

The field `Agent.agentId` of the struct is used to determine if the caller can claim. Since the id is zero, and it is and invalid id for an agent, there is no owner for it and the condition:

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L1666-L1670

Always reverts.

## Impact
The grand prize ends up locked/unclaimable by the winner

## Code Snippet

## Tool used

Manual Review

## Recommendation
In `claimGrandPrize` use 1 as the default if `agents[1].agentId == 0`:

```diff
function claimGrandPrize() external nonReentrant {
    _assertGameOver();
    uint256 agentId = agents[1].agentId;
+   if (agentId == 0)
+       agentId = 1;
    _assertAgentOwnership(agentId);

    uint256 prizePool = gameInfo.prizePool;

    if (prizePool == 0) {
        revert NothingToClaim();
    }

    gameInfo.prizePool = 0;

    _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, prizePool, gasleft());

    emit PrizeClaimed(agentId, address(0), prizePool);
}
```