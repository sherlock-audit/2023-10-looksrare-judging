Warm Concrete Mole

high

# Frontrunning with startNewRound()
## Summary

Because anyone can call `startNewRound`, an attacker can front-run calls to `heal` or `escape` with `startNewRound`. 

## Vulnerability Detail

Whenever `startNewRound` is called, a front-running lock is enabled (in `_requestForRandomness`), so during that period of time (until randomness request is fulfilled), no `heal` or `escape` will be allowed. 

Ironically enough, an attacker can abuse this front-run lock to make innocent calls to `heal` or `escape` revert by calling `startNewRound` when they observe such `heal` or `escape` requests in the mempool. This is particularly bad for healing, because it can either lead to your agent just getting killed (if time is up as a result) or you getting a worse healing probability. This is a PVP game so attackers are incentivized to do this. 

Of course, in order for this to work, the conditions to call `startNewRound` must be cleared. 

## Impact

You can adversely force calls to `heal` or `escape` to revert

## Code Snippet

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L579

## Tool used

Manual Review

## Recommendation
Make `startNewRound` only callable by owner. 