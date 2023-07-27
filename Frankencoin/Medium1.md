# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L309


# Vulnerability details

## Impact
## Brief/Intro
The restructureCapTable() has a loop issue that only burns tokens from the first address in the array. Because of this the cap table isn't restructured as intended.

## Explanation
restructureCapTable() is a function in Equity.sol that can be called when the system is at risk and qualified FPS holders want to restructure the system. This function burns FPS tokens from the qualified FPS holders that want to restructure the system. However because the current address in the loop is always set to the first address in the array, this only burns FPS tokens from the first address in the array instead of every single address in the array.  If there is a devastating loss in the system, this function doesnt help resolve the issue

## Impact

The impact of this issue is that the restructureCapTable() function is not functioning as intended. Since the loop always sets the current address to the first address in the array, the function only burns FPS tokens from the first address in the array, while leaving the rest of the addresses untouched. This means that the function is not restructuring the cap table as it should, and the system remains at risk.

## Code
```
function restructureCapTable(address[] calldata helpers, address[] calldata addressesToWipe) public {
 require(zchf.equity() < MINIMUM_EQUITY);
 checkQualified(msg.sender, helpers);
 for (uint256 i = 0; i<addressesToWipe.length; i++){
    address current = addressesToWipe[0];
    _burn(current, balanceOf(current));
 }
}
```


## Recommended Mitigation Steps
Set address current = addressesToWipe[i] so that it burns tokens from all of the addresses
