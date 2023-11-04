Howling Banana Shark

medium

# Missing approve before transferring of WETH to the recipient
## Summary
Missing approve before transferring of WETH to the recipient

## Vulnerability Detail
In the `_transferETHAndWrapIfFailWithGasLimit` function, if the original transfer fails, it should wrap the ETH into WETH and transfer the WETH to the recipient.

Firstly, the amount of ETH is deposited into the WETH contract and then transferred to the recipient. The depositing of ETH will be successful, but the transfer will revert because `msg.sender` has not approved the transfer of the amount.

When the `transfer()` function of the WETH contract is called, it directly calls the `transferFrom` function, which requires approval for the tokens before to be to be transferred.

https://etherscan.io/token/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2#code

```solidity
 function transfer(address dst, uint wad) public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }

    function transferFrom(address src, address dst, uint wad)
        public
        returns (bool)
    {
        require(balanceOf[src] >= wad);

        if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {
            require(allowance[src][msg.sender] >= wad);
            allowance[src][msg.sender] -= wad;
        }

        balanceOf[src] -= wad;
        balanceOf[dst] += wad;

        Transfer(src, dst, wad);

        return true;
    }
```

`LowLevelWETH` is not within the scope of the audit, but this will affect every part of the codebase where `_transferETHAndWrapIfFailWithGasLimit()` is used.


Also, if the original transfer fails because the gas is not enough, depositing and transferring WETH may not be possible.

## Impact
Impossible to transfer WETH to the recipient.

## Code Snippet
https://github.com/LooksRare/contracts-libs/blob/master/contracts/lowLevelCallers/LowLevelWETH.sol#L36

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L521

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L565

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L669

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L693

https://github.com/sherlock-audit/2023-10-looksrare/blob/main/contracts-infiltration/contracts/Infiltration.sol#L693

## Tool used
Manual Review

## Recommendation
Approve `_amount` to be transferred before calling of `transfer()` function.