 # Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/PeUSDMainnetStableVision.sol#L129-L139


# Vulnerability details

The executeFlashloan() in PeUSDMainnetStableVision.sol allows users to execute flash loans but the problem is that the **receiver doesnt have to be the msg.sender** so an attacker can do 2 things:

1. Execute other users flash loans

2. If a user is a **smart contract that has a fallback function**(for example a multisig wallet), then the attacker can call executeFlashloan() with the user contract address as the receiver and receiver.onFlashLoan() is not going to revert because the contract has a fallback function which is a special function that is executed when a function that does not exist is called

Because there is a fee for executing a flash loan, the attacker will burn shares of other users(either the flash loan contract or any contract that has a fallback function)


## Impact

The attacker can burn shares of other users(either the flash loan contract or any contract that has a fallback function) by continously calling executeFlashloan() or he can just make the shareAmount so big that the fee will be huge and it will burn all the users shares.

## Proof of Concept
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/PeUSDMainnetStableVision.sol#L129-L139

```solidity

function executeFlashloan(FlashBorrower receiver, uint256 eusdAmount, bytes calldata data) public payable {
uint256 shareAmount = EUSD.getSharesByMintedEUSD(eusdAmount);
EUSD.transferShares(address(receiver), shareAmount);
receiver.onFlashLoan(shareAmount, data);
bool success = EUSD.transferFrom(address(receiver), address(this), EUSD.getMintedEUSDByShares(shareAmount));
require(success, "TF");

uint256 burnShare = getFee(shareAmount);
EUSD.burnShares(address(receiver), burnShare);
emit Flashloaned(receiver, eusdAmount, burnShare);
}

```
As you can see above, there is no check if msg.sender == receiver so an attacker abuse this

Example 1

1. A multisig wallet that is a smart contract that has a fallback function(that does nothing) deposits to the Lybra protocol.
2. The attacker sees this and then calls executeFlashloan() where receiver is the multisig wallet.
3. Because the multisig wallet has a fallback function, receiver.onFlashLoan() call is not going to revert
4. Because of the fee, shares from the multisig wallet will be burned and the attacker can make this fee so big that the whole balance is burned

Example 2

1. A user has created a flash loan contract
2. The attacker knows about this flash loan and decides to executeFlashloan() where receiver is the flash loan contract
3. If nothing reverts in the flash loans onFlashLoan() function then the flash loan is charged a fee


## Tools Used
Manual review

## Recommended Mitigation Steps



Use msg.sender instead of receiver so that flash loans that want to execute this function execute it themselves.

Or the second solution might be burning shares from msg.sender instead of the receiver so that flash loan contracts dont have to hold any shares.

```solidity
function executeFlashloan(uint256 eusdAmount, bytes calldata data) public payable {
uint256 shareAmount = EUSD.getSharesByMintedEUSD(eusdAmount);
EUSD.transferShares(msg.sender, shareAmount);
FlashBorrower(msg.sender).onFlashLoan(shareAmount, data);
bool success = EUSD.transferFrom(msg.sender, address(this), EUSD.getMintedEUSDByShares(shareAmount));
require(success, "TF");

uint256 burnShare = getFee(shareAmount);
EUSD.burnShares(msg.sender, burnShare);
emit Flashloaned(FlashBorrower(msg.sender), eusdAmount, burnShare);
}

```
