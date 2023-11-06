Quiet Sandstone Osprey

medium

# The winning agent continues to be transferrable even after the grand prize has been claimed.
## Summary

The winning agent carries substantial financial value, which would be attractive on secondary marketplaces.

However, even after claiming the grand prize, [the winner continues to be transferrable](https://github.com/sherlock-audit/2023-10-looksrare/blob/86e8a3a6d7880af0dc2ca03bf3eb31bc0a10a552/contracts-infiltration/contracts/Infiltration.sol#L924), meaning sales could be frontrun by a malicious victor in order to procure **both** the grand prize and the sale value.

## Vulnerability Detail

The `transferFrom` function inherited from `ERC721A` was specifically overrided to prevent transfers of agents whose status is not one of `Active` or `Wounded`.

From the sponsor in the Sherlock [__Discord__](https://discord.com/channels/812037309376495636/1168564996897243246/1170045482534436915):

> escaped and dead agents are out of the game and carry no financial value, healing agents might be out of the game next round if healing fails so there is a risk of sales frontrunning the kill. wounded agents for the most time are still in the game. theoretically wounded agents in their last round carry the same risk but this is an accepted risk to us because the alternative would be to add more checks in transferFrom, which makes trading more expensive. We are trying to strike a balance between the cost of trade and safety. [sic]

All secondary prizes are awarded to dead agents which have no risk of being traded, which is consistent with this logic. However, the winning agent may still continue to be traded even after withdrawing the prize, meaning transactions wishing to purchase the winning agent on a secondary marketplace could be frontrun by a malicious victor:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ITransferManager} from "@looksrare/contracts-transfer-manager/contracts/interfaces/ITransferManager.sol";
import {console} from "forge-std/console.sol";
import {IOwnableTwoSteps} from "@looksrare/contracts-libs/contracts/interfaces/IOwnableTwoSteps.sol";
import {VRFConsumerBaseV2} from "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";
import {VRFCoordinatorV2Interface} from "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";

import {IInfiltration} from "../../contracts/interfaces/IInfiltration.sol";
import {Infiltration} from "../../contracts/Infiltration.sol";

import {TestHelpers} from "./TestHelpers.sol";

import {MockERC20} from "../mock/MockERC20.sol";
import {AssertionHelpers} from "./AssertionHelpers.sol";
import {ChainlinkHelpers} from "./ChainlinkHelpers.sol";

import {TestParameters} from "./TestParameters.sol";

contract Infiltration_Sherlock_TransferWinner is TestHelpers {

    function setUp() public {
        _forkMainnet();
        _deployInfiltration();
        _setMintPeriod();
    }

    function test_sherlock_transferWinner() public {

        _downTo1ActiveAgent();

        IInfiltration.Agent memory agent = infiltration.getAgent(1);
        address winner = _ownerOf(agent.agentId);

        expectEmitCheckAll();
        emit PrizeClaimed(agent.agentId, address(0), 425 ether);

        vm.prank(winner);
        infiltration.claimGrandPrize();

        assertEq(winner.balance, 425 ether);
        assertEq(address(infiltration).balance, 0);

        (, , , , , , , , uint256 prizePool, , ) = infiltration.gameInfo();
        assertEq(prizePool, 0);

        invariant_totalAgentsIsEqualToTotalSupply();

        address unsuspectingUser = address(696969);

        vm.prank(winner);
        infiltration.transferFrom(winner, unsuspectingUser, agent.agentId);
        assertTrue(_ownerOf(agent.agentId) == unsuspectingUser);
        
    }
}
```

Yields the following console output:

```shell
[⠢] Compiling...
[⠆] Compiling 19 files with 0.8.20
[⠔] Solc 0.8.20 finished in 200.07s
Compiler run successful with warnings:

Running 1 test for test/foundry/Infiltration.Sherlock.TransferWinner.t.sol:Infiltration_Sherlock_TransferWinner
[PASS] test_sherlock_transferWinner() (gas: 805601025)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.03s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

This sequence of operations indeed proves that `Infiltration`'s protections against sales frontrunning do not apply to the victorious agent, which is inconsistent with the remainder of the protection logic.

This issue arises from the fact that the victor is the last agent remaining `Active` (and therefore transferrable), though other positions carrying financial value are not transferrable because they have been `Killed`.

## Impact

Trade safety of the winning agent on secondary marketplaces is not offered the necessary protections that are applied to secondary prizewinners or agents-in-play.

## Code Snippet

```solidity
/**
 * @notice Only active and wounded agents are allowed to be transferred or traded.
 * @param from The current owner of the token.
 * @param to The new owner of the token.  * @param tokenId The token ID.
 */
 function transferFrom(address from, address to, uint256 tokenId) public payable override {
    AgentStatus status = agents[agentIndex(tokenId)].status;
    if (status > AgentStatus.Wounded) {
        revert InvalidAgentStatus(tokenId, status);
    }
    super.transferFrom(from, to, tokenId);
}
```

## Tool used

Manual Review, Visual Studio Code, Discord

## Recommendation

Once the grand prize has been claimed, the winning agent should be `Killed` in order to prevent future transfers.

Please note, this issue also applies (though in slightly lesser extent) to the winning agent's ability to withdraw a secondary prize - ideally the outstanding prizes should be settled atomically for the winning agent. This would also improve user experience by reducing the required number of transactions.
