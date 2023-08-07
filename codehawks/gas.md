## Summary

All optimizations were benchmarked using the protocol's tests using the following config: solc version 0.8.18, optimizer on, 200 runs, viaIR = true. In most cases forge test --gas-report was used

Each optimization was submitted individually.

### Gas Optimizations



| |Issue|Instances|Total Gas Saved|
|-|:-|:-:|:-:|
| [G&#x2011;01] | Consider using clones | * |  -70% cheaper deployment gas |
| [G&#x2011;02] | ReentrancyGuard can be safely removed | 1 |  42725 |
| [G&#x2011;03] | computeEscrowAddress() can be internal instead of public | 1 |  55863 + 193  |
| [G&#x2011;04] | Redundant zero address checks | 2 |  237 |
| [G&#x2011;05] | Input validation should be done in the beginning | 2 |  110649(in the revert case) | 
| [G&#x2011;06] | Emit after the transfer has been made in case it fails | 2 | 1381(in the revert case)  |
| [G&#x2011;07] | The bytecode can be removed from the function params | 1 |  27 |
| [G&#x2011;08] | Nested ifs are cheaper than && | 1 |  19 |


### [G&#x2011;01]  Consider using clones

For each escrow, the factory has to deploy a new copy of the entire contract which is very expensive. But because we are deploying exactly the same contract repeatedly, using minimal proxies is a much more gas efficient way to do it. 

### What is a minimal proxy?

Minimal Proxy Contracts are used to create lightweight and gas-efficient instances of a smart contract by reusing the logic of an existing contract. Instead of deploying a new copy of the entire contract each time like the factory does, minimal proxy contracts act as a thin layer that forwards calls to an already deployed implementation contract

A clone contract is like a proxy that cannot be upgraded. Because proxies are very small relative to the implementation contract, they are cheap to deploy.

Like the proxy pattern, clones delegate all calls to the implementation contract but keep state in their own storage.

A good example is Manifold or Gnosis, both use the clone pattern when creating a new safe/project and when you are interacting with the contract, you are actually interacting with a clone of it.

### Gas savings

So i decided to rewrite the contracts to use clones to see what the gas savings are. A gist containing all the differences can be found here. It didnt take me less than 5 minutes to rewrite everything.


*Gas Savings for newEscrow() obtained via protocol's tests(median was used because the avg isnt accurate here): 431540 gas*

|        |   MED  |   
| ------ | ------ | 
| Before |  627057  | 
| After  |  195517  |  


As you can see that is almost a -70% gas reduction for deploying a new escrow which is a lot. 

Please note that the only problem is that runtime gas will be a bit bigger because we are using delegatecall and we cant use immutables because there is no constructor(however that is still possible using [this library](https://github.com/wighawag/clones-with-immutable-args). 
However even if the runtime gas is a bit bigger, that is not a problem because only one person is interacting with that contract and deployment gas is more important here.


### [G&#x2011;02]  ReentrancyGuard can safely be removed

The resolveDispute() has a reentrancy guard just in case a token had a callback function and wanted to reenter the contract. However, because CEI is followed the nonReentrant and the reentrancy guard only wastes gas here(both deployment and runtime).

*Deployment Gas Savings for Escrow.sol obtained via protocol's tests(not using newEscrow): 40525 gas*

|        |        |
| ------ | ------ | 
| Before |  591900  | 
| After  |  551375  | 

*The runtime gas savings for resolveDispute() couldnt be calculated with foundry because it warms up the slots in the tests but when using Remix the gas savings were about 2200 gas*

*There is 1 instance of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L9
https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L109


```solidity
File: Escrow.sol

109: function resolveDispute(uint256 buyerAward) external onlyArbiter nonReentrant inState(State.Disputed) {
110:     uint256 tokenBalance = i_tokenContract.balanceOf(address(this));
111:     uint256 totalFee = buyerAward + i_arbiterFee; // Reverts on overflow
112:     if (totalFee > tokenBalance) {
113:         revert Escrow__TotalFeeExceedsBalance(tokenBalance, totalFee);
114:     }
115:
116:     s_state = State.Resolved;
117:     emit Resolved(i_buyer, i_seller);
118:
119:     if (buyerAward > 0) {
120:         i_tokenContract.safeTransfer(i_buyer, buyerAward);
121:     }
122:     if (i_arbiterFee > 0) {
123:         i_tokenContract.safeTransfer(i_arbiter, i_arbiterFee);
124:     }
125:     tokenBalance = i_tokenContract.balanceOf(address(this));
126:     if (tokenBalance > 0) {
127:         i_tokenContract.safeTransfer(i_seller, tokenBalance);
128:     }
129: }

```
As you can see here the state is set to State.Resolved before the transfer is made so if the transfer wanted to reenter then the inState() modifier would revert because the state is no longer State.Disputed. The reentrancy guard can be safely removed here.


### [G&#x2011;03]  computeEscrowAddress() can be internal instead of public

The computeEscrowAddress() here is set to public however because it is only used in the newEscrow() function and users arent going to call this function then it can be marked as internal and this will save us both deployment gas of the EscrowFactory and runtime gas when calling newEscrow()

*Deployment Gas Savings for EscrowFactory.sol obtained via protocol's tests: 55863 gas*

|        |        |
| ------ | ------ | 
| Before |  1720616  | 
| After  |  1664753  | 

*Runtime Gas Savings for newEscrow() obtained via protocol's tests(median was used because the avg isnt accurate here): 193 gas*

|        |    MED    |
| ------ |  ------   | 
| Before |  627057   | 
| After  |  626864   | 


*There is 1 instance of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L66

```solidity
File: EscrowFactory.sol

56: function computeEscrowAddress(
57:     bytes memory byteCode,
58:     address deployer,
59:     uint256 salt,
60:     uint256 price,
61:     IERC20 tokenContract,
62:     address buyer,
63:     address seller,
64:     address arbiter,
65:     uint256 arbiterFee
66: ) public pure returns (address) {

```
Use internal instead of public

### [G&#x2011;04]  Redundant zero address checks

When a new escrow is deployed, in the constructor there are 2 zero address checks that are redundant and that only waste gas. The first one is the zero token address check and the second one is zero address buyer check. 

The zero token address is redundant because safeTransferFrom() is used and if the address is zero then this will revert and later it also uses balanceOf which would also revert. 

The buyer zero address check is redundant because its impossible for the msg.sender to be address(0) when calling newEscrow().

*Gas Savings for newEscrow() obtained via protocol's tests(median was used because the avg isnt accurate here): 237 gas*

|        |    MED   |
| ------ |  ------  | 
| Before |  627057  | 
| After  |  626820  | 


*There are 2 instances of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L39-L47

```solidity
File: EscrowFactory.sol

39: tokenContract.safeTransferFrom(msg.sender, computedAddress, price);
40: Escrow escrow = new Escrow{salt: salt}(
41:     price,
42:     tokenContract,
43:     msg.sender, 
44:     seller,
45:     arbiter,
46:     arbiterFee
47: );

```
As you can see here safeTransferFrom is called so if the tokenContract is address(0) then this would revert. And the buyer can be address(0) here because msg.sender is used.

### [G&#x2011;05]  Input validation should be done in the beginning

Input validations should be always done in the beginning because we want out tx to revert as early as possible so no operations were made before the REVERT opcode was hit and no gas was wasted on them. 

When deploying a new escrow we are first computing an address, transferring tokens and only then during deployment we have input validations like zero address seller or arbiterFee < price check. These validations should be done in the beginning before any other operations are done so no gas is wasted in the revert case.

*Gas Savings for testSellerZeroReverts() obtained via forge snapshot(the gas costs here are for the entire test but only the difference is important to us): 55330 gas in the revert case*

|        |          |
| ------ |  ------  | 
| Before |  140272  | 
| After  |  84942  | 

*Gas Savings for testRevertIfFeeGreaterThanPrice() obtained via forge snapshot(the gas costs here are for the entire test but only the difference is important to us): 55319 gas in the revert case*

|        |    MED   |
| ------ |  ------  | 
| Before |  141012  | 
| After  |  85693  | 


*There are 2 instances of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L42-L43

```solidity
File: Escrow.sol

42: if (seller == address(0)) revert Escrow__SellerZeroAddress();
43: if (arbiterFee >= price) revert Escrow__FeeExceedsPrice(price, arbiterFee);

```
These checks can be done in newEscrow() before any other operations are done. In case that the seller is a zero address or arbiterFee >= price then placing this in the beginning would save a lot of gas.

### [G&#x2011;06]  Emit after the transfer has been made in case it fails

In confirmReceipt() and resolveDispute() an event is emitted right before a transfer has been made. It would be better to emit the event after the transfer because the transfer can easily fail and you would have to pay for everything before the REVERT opcode was hit so you would have to pay for emitting that event.

*Gas Savings for confirmReceipt() obtained via protocol's tests: avg 803 gas in the revert case where the transfer fails*

|        |  AVG  |  MED  |   MAX  |
| ------ | ------| ----- | ------ | 
| Before | 18329 | 26017 | 26237  |
| After  | 17526 | 24874 | 25094  |

*Gas Savings for resolveDispute() obtained via protocol's tests: avg 578 gas in the revert case where the transfer fails*

|        |  AVG  |  MED  |   MAX  |
| ------ | ------| ----- | ------ | 
| Before |  8249 | 3915  | 45127  |
| After  |  7671 | 3146  | 45118  |


*There are 2 instances of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L94-L99
https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L109-L121
```solidity
File: Escrow.sol

94: function confirmReceipt() external onlyBuyer inState(State.Created) {
95:     s_state = State.Confirmed;
96:     emit Confirmed(i_seller);
97:
98:     i_tokenContract.safeTransfer(i_seller, i_tokenContract.balanceOf(address(this)));
99: }

```

```solidity
109: function resolveDispute(uint256 buyerAward) external onlyArbiter nonReentrant inState(State.Disputed) {
110:     uint256 tokenBalance = i_tokenContract.balanceOf(address(this));
111:     uint256 totalFee = buyerAward + i_arbiterFee; // Reverts on overflow
112:     if (totalFee > tokenBalance) {
113:         revert Escrow__TotalFeeExceedsBalance(tokenBalance, totalFee);
114:     }
115:
116:     s_state = State.Resolved;
117:     emit Resolved(i_buyer, i_seller);
118:
119:     if (buyerAward > 0) {
120:         i_tokenContract.safeTransfer(i_buyer, buyerAward);
121:     }

```

The events are emitted before the transfer has been made, however if the transfer fails then you would have to pay for emitting that event so its better to emit on the end.


### [G&#x2011;07] The bytecode can be removed from the function params

Instead of having computeEscrowAddress() accept the bytecode as an argument it would more efficient if it was already embedded in the keccak256 function. This will allow us to save a small amount of gas.

*Gas Savings for newEscrow() obtained via protocol's tests(median was used because the avg isnt accurate here): 27 gas*

|        |    MED   |
| ------ |  ------  | 
| Before |  627057  | 
| After  |  627030  | 


*There is 1 instance of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L57

```solidity
File: EscrowFactory.sol

56: function computeEscrowAddress(
57:     bytes memory byteCode,
58:     address deployer,
59:     uint256 salt,
60:     uint256 price,
61:     IERC20 tokenContract,
62:     address buyer,
63:     address seller,
64:     address arbiter,
65:     uint256 arbiterFee
66: ) public pure returns (address) {
67:     address predictedAddress = address(
68:         uint160(
69:             uint256(
70:                 keccak256(
71:                     abi.encodePacked(
72:                         bytes1(0xff),
73:                         deployer,
74:                         salt,
75:                         keccak256(
76:                             abi.encodePacked(
77:                                 byteCode, abi.encode(price, tokenContract, buyer, seller, arbiter, arbiterFee)
78:                             )
79:                         )

```
Instead of passing in the byteCode into the function, it can be removed and you can put type(Escrow).creationCode in the abi.encodePacked()

### [G&#x2011;08] Nested IFs are cheaper than &&

Nested IFs are cheaper than a single && statement because the short-circuiting rules are removed.

*Gas Savings for initiateDispute() obtained via protocol's tests: avg 19 gas*

|        |  AVG  |  MED  |   MAX  |
| ------ | ------| ----- | ------ | 
| Before | 16855 | 23603 | 23603  |
| After  | 16836 | 23584 | 23584  |


*There is 1 instance of this issue:*

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L67

```solidity
File: Escrow.sol

66: modifier onlyBuyerOrSeller() {
67:     if (msg.sender != i_buyer && msg.sender != i_seller) {
68:         revert Escrow__OnlyBuyerOrSeller();
69:     }
70:     _;
71: }

```
Use a nested ifs instead of &&
