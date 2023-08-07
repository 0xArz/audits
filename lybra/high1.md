# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L131-L136


# Vulnerability details

## Vulnerability Detail

grabEsLBR() in ProtocolRewardsPool.sol allows you to purchase the accumulated amount of pre-claimed lost ESLBR in the contract using LBR but an attacker can call grabEsLBR() with a small amount and this calculation can return 0:

```solidity
(amount * grabFeeRatio) / 10000
```

which means that the attacker will burn 0 LBR but mint a small amount of esLBR.

## Impact

The attacker can repeatedly call grabEsLBR() with small amounts so that he always burns 0 LBR tokens but mints a small amount of esLBR. This allows him to take all the grabableAmount without burning any LBR tokens.

## Proof of Concept

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L131-L136

```solidity
uint256 public grabFeeRatio = 3000;

function grabEsLBR(uint256 amount) external {
require(amount > 0, "QMG");
grabableAmount -= amount;
LBR.burn(msg.sender, (amount * grabFeeRatio) / 10000);
esLBR.mint(msg.sender, amount);
}
```
Because the grabFeeRatio is 3000, the attacker can mint 3 esLBR tokens but because (3 * 3000) / 10000 equals to 0, he will burn 0 LBR tokens. This can be repeated until the grabableAmount is 0
## Tools Used
Manual review

## Recommended Mitigation Steps

Check if the calculation doesnt equal to 0

```solidity

function grabEsLBR(uint256 amount) external {
require(amount > 0, "QMG");
grabableAmount -= amount;
uint256 amountToMint = (amount * grabFeeRatio) / 10000;
require(amountToMint != 0,"");
LBR.burn(msg.sender, amountToMint);
esLBR.mint(msg.sender, amount);
}

```
