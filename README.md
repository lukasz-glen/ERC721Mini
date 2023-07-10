# ERC721 Mini

Experimental implementation of ERC721.
Gas cost can be significantly lowered if we remove `balanceOf()` function.
As the conclusion, the ERC721 standard could be restricted or new restricted ERC721Mini standard could be provided,
in order to develop gas efficient NFTs.

## Gas cost

You can often hear that ERC1155 is cheaper than ERC721. 
Some projects choose ERC1155 standard over ERC721 even if every token is unique by design, 
i.e. token amount is 1 for every token id.
This is quite surprising because ERC1155 seems to have extended token structure comparing to ERC721.
Let's check why is that so.

Costly operations for a ERC721 token transfer are typically as follows.
- check approvals (0, 1, or 2 slots read),
- change ownership (1 slot update),
- update sender balance (1 slot update or deletion),
- update receiver balance (1 slot update or creation),
- clear a single approval (1 slot deletion even if is zero),
- emit Transfer event - 1k gas.

Costly operations for a ERC1155 token transfer are typically as follows.
- check approvals (0, or 1 slots read),
- update sender balance (1 slot update or deletion),
- update receiver balance (1 slot update or creation),
- emit Transfer event - 1k gas.

## balanceOf() - reasoning

Do we actually need this function in ERC721?

In particular, do other contracts use this function? Worth to be checked. 
I would take that risk and say that in my opinion they do not.
For instance, marketplaces are token centric, token ownership is relevant.
Maybe if we consider benefits for token holders, then it would be important
to know that an address is a token holder. 
Even then, `balanceOf()` is not enough - usually a snapshot is needed in order to avoid double spend.

Note that `totalSupply()` is not a part of ERC721 but ERC721Enumerable.
The function `balanceOf()` more suits to be in ERC721Enumerable also, than in the core ERC721.
So if anyone would need on-chain enumeration, then `balanceOf()` would be a must.

Let us consider how gas cost is changed when `balanceOf()` is removed from the core ERC721.
Clearing a single token approval can be omitted thanks to some trick described below.
Costly operations for a ERC721Mini token transfer would be then as follows.
- check approvals (0, 1, or 2 slots read),
- change ownership (1 slot update),
- emit Transfer event - 1k gas.
  
Now, it is visible that ERC721 would be more gas efficient if we would remove `balanceOf()` function.
And this is the statement of this work.
If we could rewrite ERC721 from the beginning, `balanceOf()` should be moved to ERC721Enumerable.
Or it would be convenient to provide a restriction (as the opposition to extension) of ERC721,
namely ERC721Mini.

## Single approve

A typical implementation of ERC721 clears `_tokenApprovals` upon a transfer by default.
We provide a little gas optimisation.
An approval is not deleted even if it is read during a transfer.
Instead, a token's transfer counter is attached to an approval.
A current token's transfer counter is stored with token's ownership and incremented everytime a token is transferred.

```markdown
|                  _owners[tokenId]                    |
|------------------------------------------------------|
| user address (160 bits) | transfer counter (96 bits) |
```

It does not cost any additional storage operation since a counter is appended to an ownership slot.
An approval is valid if its saved counter equals current token's counter.

## Test

Test cases against compliance with ERC721 standard are taken from OpenZeppelin. 
Tests with `balanceOf()` are skipped.

## Implementations

We provide 4 implementations.
All of them are ERC721, balances are not supported the way that `balanceOf()` returns an error.
- ERC721CutA. The OZ implementation with dummy `balanceOf()`. 
- ERC721CutB. The OZ implementation with dummy `balanceOf()` and the Single Approve trick. 
- ERC721MiniA. The minimal implementation with dummy `balanceOf()`. 
- ERC721MiniB. The minimal implementation with dummy `balanceOf()` and the Single Approve trick. 

## Benchmark

Gas cost depends on storage state.
On the compiler version and compiler optimisations also.
As declared in `hardhat.config.js`, we use `0.8.13` and 10k runs.
A reference case is a transfer from an owner of two tokens to an address with one token.
Of course, this is somehow arbitrary choice, but it is enough for comparison.

```
  Contract: ERC721
Basic transferFrom gas cost: 43884

  Contract: ERC721CutA
Basic transferFrom gas cost: 33661

  Contract: ERC721CutB
Basic transferFrom gas cost: 31180

  Contract: ERC721MiniA
Basic transferFrom gas cost: 32700

  Contract: ERC721MiniB
Basic transferFrom gas cost: 30393
```

## Feedback

Any feedback is appreciated. [Here](https://github.com/lukasz-glen/ERC721Mini/issues/1). 
