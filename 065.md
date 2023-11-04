Funny Daffodil Stallion

high

# The number of activeAgents could be zero and this will lead to loss of fund in the contract
## Summary

Via VRF request, in the fullfillRandomWord function of Infiltration.sol, activeAgents will be 0 and the final winner will not get the Grand Prize and the ETH(gameInfo.prizePool) will be stuck in the contract forever.

## Vulnerability Detail

After receiving random words from VRF, in fulfillRandomWords function the activeAgents can be 0 and the final winner can't claim the Grand Prize and also the owner can't withdraw the ETH in the contract.
```solidity
if (activeAgents > NUMBER_OF_SECONDARY_PRIZE_POOL_WINNERS) {
    uint256 woundedAgents = _woundRequestFulfilled(
        currentRoundId,
        currentRoundAgentsAlive,
        activeAgents,
        currentRandomWord
    );

    uint256 deadAgentsFromKilling;
    if (currentRoundId > ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD) {
        deadAgentsFromKilling = _killWoundedAgents({
            roundId: currentRoundId.unsafeSubtract(ROUNDS_TO_BE_WOUNDED_BEFORE_DEAD),
            currentRoundAgentsAlive: currentRoundAgentsAlive
        });
    }

    // We only need to deduct wounded agents from active agents, dead agents from killing are already inactive.

    // This is equivalent to
    // unchecked {
    //     gameInfo.activeAgents = activeAgents - woundedAgents;
    //     gameInfo.woundedAgents = gameInfo.woundedAgents + woundedAgents - deadAgentsFromKilling;
    //     gameInfo.deadAgents += (deadAgentsFromHealing + deadAgentsFromKilling);
    // }
    // SSTORE is called in _incrementRound
    uint256 gameInfoSlot0Value;
    assembly {
        gameInfoSlot0Value := sload(gameInfo.slot)

        let currentWoundedAgents := and(
            shr(GAME_INFO__WOUNDED_AGENTS_OFFSET, gameInfoSlot0Value),
            TWO_BYTES_BITMASK
        )
        let currentDeadAgents := and(shr(GAME_INFO__DEAD_AGENTS_OFFSET, gameInfoSlot0Value), TWO_BYTES_BITMASK)

        gameInfoSlot0Value := and(
            gameInfoSlot0Value,
            // This is equivalent to
            // not(
            //     or(
            //         TWO_BYTES_BITMASK,
            //         or(
            //             shl(GAME_INFO__WOUNDED_AGENTS_OFFSET, TWO_BYTES_BITMASK),
            //             shl(GAME_INFO__DEAD_AGENTS_OFFSET, TWO_BYTES_BITMASK)
            //         )
            //     )
            // )
            0xffffffffffffffffffffffffffffffffffffffffffffffff0000ffff00000000
        )
        gameInfoSlot0Value := or(gameInfoSlot0Value, sub(activeAgents, woundedAgents))

        gameInfoSlot0Value := or(
            gameInfoSlot0Value,
            shl(
                GAME_INFO__WOUNDED_AGENTS_OFFSET,
                sub(add(currentWoundedAgents, woundedAgents), deadAgentsFromKilling)
            )
        )

        gameInfoSlot0Value := or(
            gameInfoSlot0Value,
            shl(
                GAME_INFO__DEAD_AGENTS_OFFSET,
                add(currentDeadAgents, add(deadAgentsFromHealing, deadAgentsFromKilling))
            )
        )
    }
    _incrementRound(currentRoundId, gameInfoSlot0Value);
} else {
    uint256 killedAgentIndex = (currentRandomWord % activeAgents).unsafeAdd(1);
    Agent storage agentToKill = agents[killedAgentIndex];
    uint256 agentId = _agentIndexToId(agentToKill, killedAgentIndex);
    _swap({
        currentAgentIndex: killedAgentIndex,
        lastAgentIndex: currentRoundAgentsAlive,
        agentId: agentId,
        newStatus: AgentStatus.Dead
    });

    unchecked {
        --activeAgents;
    }

    // This is equivalent to
    // unchecked {
    //     gameInfo.activeAgents = activeAgents;
    //     gameInfo.deadAgents = gameInfo.deadAgents + deadAgentsFromHealing + 1;
    // }
    // SSTORE is called in _incrementRound
    uint256 gameInfoSlot0Value;
    assembly {
        gameInfoSlot0Value := sload(gameInfo.slot)
        let deadAgents := and(shr(GAME_INFO__DEAD_AGENTS_OFFSET, gameInfoSlot0Value), TWO_BYTES_BITMASK)

        gameInfoSlot0Value := and(
            gameInfoSlot0Value,
            // This is equivalent to not(or(TWO_BYTES_BITMASK, shl(GAME_INFO__DEAD_AGENTS_OFFSET, TWO_BYTES_BITMASK)))
            0xffffffffffffffffffffffffffffffffffffffffffffffff0000ffffffff0000
        )
        gameInfoSlot0Value := or(gameInfoSlot0Value, activeAgents)
        gameInfoSlot0Value := or(
            gameInfoSlot0Value,
            shl(GAME_INFO__DEAD_AGENTS_OFFSET, add(add(deadAgents, deadAgentsFromHealing), 1))
        )
    }

    uint256[] memory killedAgentId = new uint256[](1);
    killedAgentId[0] = agentId;
    emit Killed(currentRoundId, killedAgentId);

    _emitWonEventIfOnlyOneActiveAgentRemaining(activeAgents);

    _incrementRound(currentRoundId, gameInfoSlot0Value);
}
```
Suppose activeAgents = 1 and the VRF request is called, since activeAgent is 1 and this will reduced by 1 in L1209
```solidity
unchecked {
      --activeAgents;
  }
```
And this means activeAgents is now 0.
This game status will save in L1218~1233.
```solidity
uint256 gameInfoSlot0Value;
    assembly {
        gameInfoSlot0Value := sload(gameInfo.slot)
        let deadAgents := and(shr(GAME_INFO__DEAD_AGENTS_OFFSET, gameInfoSlot0Value), TWO_BYTES_BITMASK)

        gameInfoSlot0Value := and(
            gameInfoSlot0Value,
            // This is equivalent to not(or(TWO_BYTES_BITMASK, shl(GAME_INFO__DEAD_AGENTS_OFFSET, TWO_BYTES_BITMASK)))
            0xffffffffffffffffffffffffffffffffffffffffffffffff0000ffffffff0000
        )
        gameInfoSlot0Value := or(gameInfoSlot0Value, activeAgents)
        gameInfoSlot0Value := or(
            gameInfoSlot0Value,
            shl(GAME_INFO__DEAD_AGENTS_OFFSET, add(add(deadAgents, deadAgentsFromHealing), 1))
        )
    }
```
Since activeAgents is 0, the winner can't claim his reward.
```solidity
function claimGrandPrize() external nonReentrant {
    _assertGameOver();
    ...
}
function _assertGameOver() private view {
  if (gameInfo.activeAgents != 1) {
      revert GameIsStillRunning();
  }
}
```
Furthermore the owner of the contract can't not withdraw the funds using emergencyWithdraw.
Because in case of currentRoundId >0, activeAgents = 0, the conditionOne, conditionTwo, conditionThree all are false.
```solidity
bool conditionOne = currentRoundId != 0 &&
    activeAgents + woundedAgents + healingAgents + escapedAgents + deadAgents != totalSupply();

// 50 blocks per round * 216 = 10,800 blocks which is roughly 36 hours
// Prefer not to hard code this number as BLOCKS_PER_ROUND is not always 50
bool conditionTwo = currentRoundId != 0 && // @audit-info this checks for the time elapsed since the game started.
    activeAgents > 1 &&
    block.number > currentRoundBlockNumber + BLOCKS_PER_ROUND * 216;

// Just in case startGame reverts, we can withdraw the ETH balance and redistribute to addresses that participated in the mint.
bool conditionThree = currentRoundId == 0 && block.timestamp > uint256(mintEnd).unsafeAdd(36 hours);

if (conditionOne || conditionTwo || conditionThree) {
    uint256 ethBalance = address(this).balance;
    _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, ethBalance, gasleft());

    uint256 looksBalance = IERC20(LOOKS).balanceOf(address(this));
    _executeERC20DirectTransfer(LOOKS, msg.sender, looksBalance);

    emit EmergencyWithdrawal(ethBalance, looksBalance);
}
```
So the rescue of funds in the contract is impossible.
## Impact
If VRF request => Infiltration.sol#fulfillRandomWords => gameInfo.activeAgents = 0 happens, the final winner of the game can't claim the prize and the funds will be locked the contract forever.
## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L1096
## Tool used

Manual Review

## Recommendation
In the function fulfillRandomWords, there has to a check for activeAgents is not 0.

```solidity
unchecked {
      --activeAgents; // L1209
  }

require(activeAgents > 0, "Zero active agents");  // L1212 +
```
