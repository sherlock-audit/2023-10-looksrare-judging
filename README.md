# Issue H-1: _killWoundedAgents 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/21 

## Found by 
ast3ros, cergyk, klaus, mstpr-brainbot

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



## Discussion

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/148

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/164

# Issue H-2: Incorrect bitmask used in _swap will lead to ids in agents mapping to be corrupted 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/27 

## Found by 
cergyk
Infiltration smart contract uses a common pattern when handling deletion in arrays:
Say we want to delete element i in array of length N:

1/ We swap element at index i with element at index N-1
2/ We update the length of the array to be N-1

Infiltration uses `_swap()` method to do this with a twist:
It updates the status of the removed agent to `Killed`, and initializes id of last agent if it is uninitialized. 

There is a mistake in a bitmask value used for resetting the id of the last agent, which will mess up the game state.

Lines of interest for reference:
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L1582-L1592

## Vulnerability Detail

Let's take a look at the custom `assembly` block used to assign an id to the newly allocated `lastAgent`:

```solidity
assembly {
    let lastAgentCurrentValue := sload(lastAgentSlot)
    // Replace the last agent's ID with the current agent's ID.
--> lastAgentCurrentValue := and(lastAgentCurrentValue, not(AGENT__STATUS_OFFSET))
    lastAgentCurrentValue := or(lastAgentCurrentValue, lastAgentId)
    sstore(currentAgentSlot, lastAgentCurrentValue)

    let lastAgentNewValue := agentId
    lastAgentNewValue := or(lastAgentNewValue, shl(AGENT__STATUS_OFFSET, newStatus))
    sstore(lastAgentSlot, lastAgentNewValue)
}
```

The emphasized line shows that `not(AGENT__STATUS_OFFSET)` is used as a bitmask to set the `agentId` value to zero.
However `AGENT__STATUS_OFFSET == 16`, and this means that instead of setting the low-end 2 bytes to zero, this will the single low-end fifth bit to zero. Since bitwise or is later used to assign lastAgentId, if the id in lastAgentCurrentValue is not zero, and is different from lastAgentId, the resulting value is arbitrary, and can cause the agent to be unrecoverable. 

The desired value for the mask is `not(TWO_BYTES_BITMASK)`:

```solidity
    not(AGENT__STATUS_OFFSET) == 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffef
    not(TWO_BYTES_BITMASK)    == 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0000
```

## Impact
The ids of the agents will be mixed, this could mean that some agents will be unrecoverable, or duplicated.
Since the attribution of the grand prize relies on the `agentId` stored, we may consider that the legitimate winner can lose access to the prize.

## Code Snippet

## Tool used

Manual Review

## Recommendation
```diff
assembly {
    let lastAgentCurrentValue := sload(lastAgentSlot)
    // Replace the last agent's ID with the current agent's ID.
-   lastAgentCurrentValue := and(lastAgentCurrentValue, not(AGENT__STATUS_OFFSET))
+   lastAgentCurrentValue := and(lastAgentCurrentValue, not(TWO_BYTES_BITMASK))
    lastAgentCurrentValue := or(lastAgentCurrentValue, lastAgentId)
    sstore(currentAgentSlot, lastAgentCurrentValue)

    let lastAgentNewValue := agentId
    lastAgentNewValue := or(lastAgentNewValue, shl(AGENT__STATUS_OFFSET, newStatus))
    sstore(lastAgentSlot, lastAgentNewValue)
}
```



## Discussion

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/150

# Issue H-3: Winning agent id may be uninitialized when game is over, locking grand prize 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/31 

## Found by 
cergyk, detectiveking
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



## Discussion

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/153

# Issue H-4: Attacker can steal reward of actual winner by force ending the game 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/98 

## Found by 
0xReiAyanami, SilentDefendersOfDeFi, cergyk, mstpr-brainbot

A malicious user can force win the game by escaping all but one wounded agent, and steal the grand price.

## Vulnerability Detail

Currently following scenario is possible: 
There is an attacker owning some lower index agents and some higher index agents.
There is a normal user owing one agent with an index between the attackers agents.
If one of the attackers agents with an lower index gets wounded, he can escape all other agents and will instantly win the game,
even if the other User has still one active agent.

This is possible because because the winner is determined by the agent index, 
and escaping all agents at once wont kill the wounded agent because the game instantly ends.

Following check inside startNewRound prevents killing of wounded agents by starting a new round:

```solidity
uint256 activeAgents = gameInfo.activeAgents;
        if (activeAgents == 1) {
            revert GameOver();
        }
```

Following check inside of claimPrize pays price to first ID agent:

```solidity
uint256 agentId = agents[1].agentId;
_assertAgentOwnership(agentId);
```

See following POC:

## POC

Put this into Infiltration.mint.t.sol and run `forge test --match-test forceWin -vvv`

```solidity
function test_forceWin() public {
        address attacker = address(1337);

        //prefund attacker and user1
        vm.deal(user1, PRICE * MAX_MINT_PER_ADDRESS);
        vm.deal(attacker, PRICE * MAX_MINT_PER_ADDRESS);

        // MINT some agents
        vm.warp(_mintStart());
        // attacker wants to make sure he owns a bunch of agents with low IDs!!
        vm.prank(attacker);
        infiltration.mint{value: PRICE * 30}({quantity: 30});
        // For simplicity we mint only 1 agent to user 1 here, but it could be more, they could get wounded, etc.
        vm.prank(user1);
        infiltration.mint{value: PRICE *1}({quantity: 1});
        //Attacker also wants a bunch of agents with the highest IDs, as they are getting swapped with the killed agents (move forward)
        vm.prank(attacker);
        infiltration.mint{value: PRICE * 30}({quantity: 30});
    
        vm.warp(_mintEnd());

        //start the game
        vm.prank(owner);
        infiltration.startGame();

        vm.prank(VRF_COORDINATOR);
        uint256[] memory randomWords = new uint256[](1);
        randomWords[0] = 69_420;
        VRFConsumerBaseV2(address(infiltration)).rawFulfillRandomWords(_computeVrfRequestId(1), randomWords);
        // Now we are in round 2 we do have 1 wounded agent (but we can imagine any of our agent got wounded, doesn´t really matter)
        
        // we know with our HARDCODED RANDOMNESS THAT AGENT 3 gets wounded!!

        // Whenever we get in a situation, that we own all active agents, but 1 and our agent has a lower index we can instant win the game!!
        // This is done by escaping all agents, at once, except the lowest index
        uint256[] memory escapeIds = new uint256[](59);
        escapeIds[0] = 1;
        escapeIds[1] = 2;
        uint256 i = 4; //Scipping our wounded AGENT 3
        for(; i < 31;) {
            escapeIds[i-2] = i;
            unchecked {++i;}
        }
        //skipping 31 as this owned by user1
        unchecked {++i;}
        for(; i < 62;) {
            escapeIds[i-3] = i;
            unchecked {++i;}
        }
        vm.prank(attacker);
        infiltration.escape(escapeIds);

        (uint16 activeAgents, uint16 woundedAgents, , uint16 deadAgents, , , , , , , ) = infiltration.gameInfo();
        console.log("Active", activeAgents);
        assertEq(activeAgents, 1);
        // This will end the game instantly.
        //owner should not be able to start new round
        vm.roll(block.number + BLOCKS_PER_ROUND);
        vm.prank(owner);
        vm.expectRevert();
        infiltration.startNewRound();

        //Okay so the game is over, makes sense!
        // Now user1 has the only active AGENT, so he should claim the grand prize!
        // BUT user1 cannot
        vm.expectRevert(IInfiltration.NotAgentOwner.selector);
        vm.prank(user1);
        infiltration.claimGrandPrize();

        //instead the attacker can:
        vm.prank(attacker);
        infiltration.claimGrandPrize();
        
```

## Impact

Attacker can steal the grand price of the actual winner by force ending the game trough escapes.

This also introduces problems if there are other players with wounded agents but lower < 50 TokenID, they can claim prices 
for wounded agents, which will break parts of the game logic.

## Code Snippet

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L589-L592

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L656-L672
## Tool used

Manual Review

## Recommendation

Start a new Round before the real end of game to clear all wounded agents and reorder IDs.



## Discussion

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/154

# Issue M-1: Wound agent can't invoke heal in the next round 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/44 

## Found by 
coffiasd, lil.eth, mstpr-brainbot, pontifex
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



## Discussion

**nevillehuang**

@0xhiroshi Any reason why you disagree with the severity? It does mentioned in the docs that the first round user can heal is right after the round his agent is wounded. The above PoC also shows how a user cannot heal a wounded agent at round 13 when his agent was wounded at round 12.

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/151

# Issue M-2: Index values selected in `_woundRequestFulfilled()` are not uniformly distributed. 

Source: https://github.com/sherlock-audit/2023-10-looksrare-judging/issues/84 

## Found by 
0xGoodess, Tricko, cergyk, detectiveking
Index values selected in `_woundRequestFulfilled()` are not uniformly distributed. Indexes right next to wounded agents are more likely to be selected in the subsequent iterations, leading to bias in the distribution of wounded agents.

## Vulnerability Detail
At the end of each round, the function `_woundRequestFulfilled()` is called, which uses uses the `randomWord` obtained from the VRF to select which agents should be marked as wounded. This selection process is carried out by performing a modulo operation on the `randomWord` with respect to the number of agents currently alive in the round, and then adding 1 to the result. The resulting value corresponds to the index of the agent to be designated as wounded, as illustrated in the code snippet section.

However, if the resulting index corresponds to an agent who is already wounded, the `else` branch is executed, where 1 is added to the `randomWord` for the next iteration of the loop. **This is where the bias is introduced, because in the next iteration, the `woundedAgentIndex` will be the current `woundedAgentIndex` plus 1**. As can be seen below:

$$ (A + 1) \bmod M $$

$$ ((A \bmod  M) + (1 \bmod  M)) \bmod  M $$

As M > 1, we can simplify to

$$ ((A \bmod  M) + 1) \bmod  M $$

For $(A \bmod M) + 1$ less than $M$, we have 

$$ (A + 1) \bmod M  = (A \bmod M) + 1 $$

So with the exception of when `randomWord` overflows or (`randomWord` % `currentRoundAgentsAlive` + 1) >= `currentRoundAgentsAlive`,  (`randomWord` + 1) % `currentRoundAgentsAlive` will be equal to
(`randomWord` % `currentRoundAgentsAlive`) + 1.

Consequently, when the `else` branch is triggered, the next `woundedAgentIndex` will be ( previous `woundedAgentIndex`+ 1) from the last loop iteration (besides the two exceptions specified above). Therefore the agent at the next index will also be marked as wounded. As a result of this pattern, **agents whose indexes are immediately next to an already wounded agent are more likely to be wounded than the remaining agents**.

Consider the representative example below, albeit on a smaller scale (8 agents) to facilitate explanation. The initial state is represented in the table below:

| Index       | 1      | 2      | 3      | 4      | 5      | 6      | 7      | 8      |
|----------------|--------|--------|--------|--------|--------|--------|--------|--------|
| Agents         | Active | Active | Active | Active | Active | Active | Active | Active |
| Probabilities  | 0.125  | 0.125  | 0.125  | 0.125  | 0.125  | 0.125  | 0.125  | 0.125  |

In the initial iteration of `_woundRequestFulfilled()`, assume that index 2 is selected. As expected from function logic, for the next iteration a new `randomWord` will be generated, resulting in a new index within the range of 1 to 8, all with equal probabilities. However now not all agents have an equal likelihood of being wounded. This disparity arises because both 3 and 2 (due to the `else` branch in the code above) will lead to the agent at index 3 being wounded.

| Index       | 1      | 2       | 3      | 4      | 5      | 6      | 7      | 8      |
|----------------|--------|---------|--------|--------|--------|--------|--------|--------|
| Agents         | Active | Wounded | Active | Active | Active | Active | Active | Active |
| Probabilities  | 0.125  | -       | 0.25   | 0.125  | 0.125  | 0.125  | 0.125  | 0.125  |

Now suppose that agents at index 2 and 3 are wounded. Following from the explanations above, index 4 has three times the chance of being wounded.

| Index       | 1      | 2       | 3       | 4      | 5      | 6      | 7      | 8      |
|----------------|--------|---------|---------|--------|--------|--------|--------|--------|
| Agents         | Active | Wounded | Wounded | Active | Active | Active | Active | Active |
| Probabilities  | 0.125  | -       | -       | 0.375  | 0.125  | 0.125  | 0.125  | 0.125  |

## Impact
The distribution of wounded status among indexes is not uniform. Indexes subsequent to agents already wounded are more likely to be selected, introducing unfairness in the distribution of wounded agents. 

This is particularly problematic because indexes that are close to each other are more likely to owned by the same address, due to the fact that when minting multiple agents, they are created in sequential indexes. Consequently, if an address has one of its agents marked as wounded, it becomes more probable that additional agents owned by the same address will also be marked as wounded. This creates an unfair situation to users.

## Code Snippet
https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L1404-L1468

## Tool used
Manual Review

## Recommendation
Consider eliminating the `else` and calling `_nextRandomWord(randomWord)` at the end of the loop iteration, so a new `randomWord` is generated each time. As shown in the diff below:

```diff
diff --git a/Infiltration.sol b/Infiltration.mod.sol
index 31af961..1c43c31 100644
--- a/Infiltration.sol
+++ b/Infiltration.mod.sol
@@ -1451,15 +1451,9 @@ contract Infiltration is
                     ++i;
                     currentRoundWoundedAgentIds[i] = uint16(woundedAgentId);
                 }
-
-                randomWord = _nextRandomWord(randomWord);
-            } else {
-                // If no agent is wounded using the current random word, increment by 1 and retry.
-                // If overflow, it will wrap around to 0.
-                unchecked {
-                    ++randomWord;
-                }
             }
+
+            randomWord = _nextRandomWord(randomWord);
         }

         currentRoundWoundedAgentIds[0] = uint16(woundedAgentsCount);
```



## Discussion

**0xhiroshi**

https://github.com/LooksRare/contracts-infiltration/pull/152

