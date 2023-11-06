Fun Aegean Halibut

high

# Incorrect bitmask used in _swap will lead to ids in agents mapping to be corrupted
## Summary
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