  # Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L154
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L187
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L125


# Vulnerability details

In LybraEUSDVaultBase.sol and LybraPeUSDVaultBase.sol there is no check if a user can self liquidate himself. The provider and onBehalfOf can be the exact same address so a user can self liquidate himself which will burn his EUSD/PeUSD, transfer his collateral back to him and he will also benefit from this because he receives a small reward

## Impact
The user can self liquidate himself and he wont lose any of his collateral and he will also receive a small reward for liquidating himself. The user can abuse this and he can keep self liquidating himself to avoid losses and receive rewards at the same time.

## Proof of Concept
Applies for both superLiquidation() and liquidation() in the EUSD Vault base

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L154 https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L187

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L125

```solidity

function liquidation(address provider, address onBehalfOf, uint256 assetAmount) external virtual {
    uint256 assetPrice = getAssetPrice();
    uint256 onBehalfOfCollateralRatio = (depositedAsset[onBehalfOf] * assetPrice * 100) / borrowed[onBehalfOf];
    require(onBehalfOfCollateralRatio < badCollateralRatio, "Borrowers collateral ratio should below badCollateralRatio");

    require(assetAmount * 2 <= depositedAsset[onBehalfOf], "a max of 50% collateral can be liquidated");
    require(EUSD.allowance(provider, address(this)) > 0, "provider should authorize to provide liquidation EUSD");
    uint256 eusdAmount = (assetAmount * assetPrice) / 1e18;

    _repay(provider, onBehalfOf, eusdAmount);
    uint256 reducedAsset = (assetAmount * 11) / 10;
    totalDepositedAsset -= reducedAsset;
    depositedAsset[onBehalfOf] -= reducedAsset;
    uint256 reward2keeper;
    if (provider == msg.sender) {
        collateralAsset.transfer(msg.sender, reducedAsset);
    } else {
        reward2keeper = (reducedAsset * configurator.vaultKeeperRatio(address(this))) / 110;
        collateralAsset.transfer(provider, reducedAsset - reward2keeper);
        collateralAsset.transfer(msg.sender, reward2keeper);
    }
    emit LiquidationRecord(provider, msg.sender, onBehalfOf, eusdAmount, reducedAsset, reward2keeper, false, block.timestamp);
}
```
As you can see there is no check if provider == onBehalfOf so a user can self liquidate himself which allows users to avoid losses and gain rewards at the same time

## Tools Used
Manual review

## Recommended Mitigation Steps
Check if provider == onBehalfOf

```
function liquidation(address provider, address onBehalfOf, uint256 assetAmount) external virtual {
    require(provider != onBehalfOf, "Cant self liquidate");
    uint256 assetPrice = getAssetPrice();
    uint256 onBehalfOfCollateralRatio = (depositedAsset[onBehalfOf] * assetPrice * 100) / borrowed[onBehalfOf];
    require(onBehalfOfCollateralRatio < badCollateralRatio, "Borrowers collateral ratio should below badCollateralRatio");

    require(assetAmount * 2 <= depositedAsset[onBehalfOf], "a max of 50% collateral can be liquidated");
    require(EUSD.allowance(provider, address(this)) > 0, "provider should authorize to provide liquidation EUSD");
    uint256 eusdAmount = (assetAmount * assetPrice) / 1e18;

    _repay(provider, onBehalfOf, eusdAmount);
    uint256 reducedAsset = (assetAmount * 11) / 10;
    totalDepositedAsset -= reducedAsset;
    depositedAsset[onBehalfOf] -= reducedAsset;
    uint256 reward2keeper;
    if (provider == msg.sender) {
        collateralAsset.transfer(msg.sender, reducedAsset);
    } else {
        reward2keeper = (reducedAsset * configurator.vaultKeeperRatio(address(this))) / 110;
        collateralAsset.transfer(provider, reducedAsset - reward2keeper);
        collateralAsset.transfer(msg.sender, reward2keeper);
    }
    emit LiquidationRecord(provider, msg.sender, onBehalfOf, eusdAmount, reducedAsset, reward2keeper, false, block.timestamp);
}

```
