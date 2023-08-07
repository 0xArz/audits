 # Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L154-L176
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L187-L211
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L125-L146


# Vulnerability details

In all liquidation functions, there is no check what getAssetPrice() returns so if the price falls to 0 or it just returns 0 because the oracle doesnt work some unexpected stuff may happen in the liquidation()/superLiquidation() functions because of incorrect calculations. The attacker can transfer all the collateral without repaying and burning EUSD/PeUSD.

## Impact

Because the liquidation functions use the assetPrice a lot to calculate other stuff, it will be a problem when the assetPrice will be 0.

The attacker can burn 0 EUSD/PeUSD because the eusdAmount will be 0 but he can transfer all the collateral to him and liquidate the user.

## Proof of Concept
Applies for both superLiquidation() and liquidation() in the EUSD Vault base

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L154 https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L187

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L125

```solidity

function liquidation(address provider, address onBehalfOf, uint256 assetAmount) external virtual {
     uint256 assetPrice = getAssetPrice();
     uint256 onBehalfOfCollateralRatio = (depositedAsset[onBehalfOf] * assetPrice *100)/borrowed[onBehalfOf];
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
Because there is no check if getAssetPrice() returns 0, the assetPrice will be 0 and that will cause problems.

If the assetPrice is 0 then the eusdAmount/peusdAmount will also be zero so the attacker will burn 0 EUSD/PeUSD tokens but transfer all the assetAmount to him

## Tools Used

Manual review

## Recommended Mitigation Steps

Require the assetPrice to be bigger than 0
```solidity

function liquidation(address provider, address onBehalfOf, uint256 assetAmount) external virtual {
    uint256 assetPrice = getAssetPrice();
    require(assetPrice > 0, "");
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
