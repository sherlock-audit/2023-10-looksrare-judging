Powerful Golden Bull

medium

# Index values selected in `_woundRequestFulfilled()` are not uniformly distributed.
## Summary
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
