## Issue 1 - User gets less tokens in V2Migrator.sol than he deposits

Contract: V2Migrator.sol
https://github.com/code-423n4/2023-04-rubicon/blob/511636d889742296a54392875a35e4c0c4727bb7/contracts/V2Migrator.sol#L55

When the user withdraws tokens from bathTokenV1 he pays a fee. The withdraw function then returns the (deposit amount - fee) which is then set as a variable amountWithdrawn in migrate(). Then the new bathTokenV2 are minted to the contract, but the amount that is minted is the amountWithdrawn so the user gets less than he deposits.

## Recommended Mitigation Steps
Mint the amount that he deposits

