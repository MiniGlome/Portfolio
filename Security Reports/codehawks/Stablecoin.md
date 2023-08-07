<img src="https://res.cloudinary.com/droqoz7lg/image/upload/q_90/dpr_2.0/c_fill,g_auto,h_320,w_320/f_auto/v1/company/clgyfdhp5rqfd5f6yh4t?_a=BATCr5AA0" width=100 height=100>

# [CodeHawks Stablecoin](https://www.codehawks.com/contests/cljyfxlc40003jq082s0wemya)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| CodeHawks Stablecoin | [Website](https://www.codehawks.com/) | [Twitter](https://twitter.com/CodeHawks) | $15,000 USDC | 236 | 12 days | Jul 24, 2023 | Aug 5, 2023 |

## What is CodeHawks Stablecoin?

This project is meant to be a stablecoin where users can deposit WETH and WBTC in exchange for a token that will be pegged to the USD. The system is meant to be such that someone could fork this codebase, swap out WETH & WBTC for any basket of assets they like, and the code would work the same.

## Audit scope

- [DSCEngine](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DSCEngine.sol)
- [DecentralizedStableCoin](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/DecentralizedStableCoin.sol)
- [OracleLib](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/main/src/libraries/OracleLib.sol)
  
## Issues found by Mlome

| Severity | Title | Count |
|:--|:--|:--:|
| High | Mutli tokens collateralization can block the liquidation process | [H-01] |
| Medium | Liquidation can be frontrun | [M-01] |

# [H-01] - Mutli tokens collateralization can block the liquidation process

## Summary
In some cases where the collateral is spread across multiple tokens, the liquidation may become impossible only after a small price drop.

## Vulnerability Details
To liquidate a position the liquidator has to take care that the `healthFactor` is broken before the liquidation and has recovered at the end of the transaction for it to go through. So the `healthFactor` should not be broken anymore after a liquidation transaction, that means the value of the liquidated position's collateral is back to at least 200% of the value of the underlying stablecoin.
However, the `liquidate()` function only allows the liquidation of one token at a time which might not be enough to recover the full debt if the collateral is composed of several tokens. Because the debt cannot be recovered in one call to the `liquidate()` function, the position is not liquidatable anymore.

```solidity
File: DSCEngine.Sol

229: function liquidate(address collateral, address user, uint256 debtToCover) // @audit - possible frontrun
        external
        moreThanZero(debtToCover)
        nonReentrant
    {
        // need to check health factor of the user
        uint256 startingUserHealthFactor = _healthFactor(user);
        if (startingUserHealthFactor >= MIN_HEALTH_FACTOR) {
            revert DSCEngine__HealthFactorOk();
        }
        // We want to burn their DSC "debt"
        // And take their collateral
        // Bad User: $140 ETH, $100 DSC
        // debtToCover = $100
        // $100 of DSC == ??? ETH?
        // 0.05 ETH
        uint256 tokenAmountFromDebtCovered = getTokenAmountFromUsd(collateral, debtToCover);
        // And give them a 10% bonus
        // So we are giving the liquidator $110 of WETH for 100 DSC
        // We should implement a feature to liquidate in the event the protocol is insolvent
        // And sweep extra amounts into a treasury
        // 0.05 * 0.1 = 0.005. Getting 0.055
        uint256 bonusCollateral = (tokenAmountFromDebtCovered * LIQUIDATION_BONUS) / LIQUIDATION_PRECISION;
        uint256 totalCollateralToRedeem = tokenAmountFromDebtCovered + bonusCollateral;
        _redeemCollateral(user, msg.sender, collateral, totalCollateralToRedeem);
        // We need to burn the DSC
        _burnDsc(debtToCover, user, msg.sender);

        uint256 endingUserHealthFactor = _healthFactor(user);
        if (endingUserHealthFactor <= startingUserHealthFactor) {
            revert DSCEngine__HealthFactorNotImproved();
        }
        _revertIfHealthFactorIsBroken(msg.sender);
    }
```

## Impact
Some <200% collateralized positions are not possible to liquidate, breaking the core mechanism of the stablecoin.
This can happen even with quite a small debt.

## Proof Of Concept

(1) A user uses 5 tokens as collateral, worth \$1000 in total, to mint \$500 of DSC.

Collateral:
- \$200 TokenA
- \$200 TokenB
- \$200 TokenC
- \$200 TokenD
- \$200 TokenE
- TOTAL = \$1000

Minted:
- \$500 DSC

HealthFactor = (1000/2) / 500 = 1

(2) All prices go down by 15%

Collateral:
- \$170 TokenA
- \$170 TokenB
- \$170 TokenC
- \$170 TokenD 
- \$170 TokenE
- TOTAL = \$850

Minted:
- \$500 DSC

HealthFactor = (850/2) / 500 = 0.85

There is a liquidation opportunity now that HealthFactor = 0.85 < 1

(3) A liquidator burns the maximum possible amount of \$154.54 DSC and gets \$170 worth of Token0 (\$154.54 + 10%)
Remaining:
- \$0   TokenA
- \$170 TokenB
- \$170 TokenC
- \$170 TokenD
- \$170 TokenE
- TOTAL = \$680

Minted:
- 500 - 154.54 = \$345.46 DSC

HealthFactor = (680/2) / 345.46 = 0.96

The liquidation transaction FAILS because the HealthFactor = 0.96 < 1

(4) The liquidation process will always revert and the position will remain <200% collateralized. 

## Tools Used

Manual review

## Recommendations
Allow the liquidation of several tokens at the same time.


# [M-01] - Liquidation can be frontrun

## Summary
Liquidation transaction can be frontrun to steal liquidator's reward.

## Vulnerability Details
When a position is no longer sufficiently collateralized, the `liquidate()` function can be called to liquidate the position and earn a 10% bonus. However, when someone finds a liquidation opportunity and calls this function, anyone can frontrun it and execute the liquidation first to get the bonus reward.

```solidity
File: DSCEngine.Sol

229: function liquidate(address collateral, address user, uint256 debtToCover) // @audit - possible frontrun
        external
        moreThanZero(debtToCover)
        nonReentrant
    {
        // need to check health factor of the user
        uint256 startingUserHealthFactor = _healthFactor(user);
        if (startingUserHealthFactor >= MIN_HEALTH_FACTOR) {
            revert DSCEngine__HealthFactorOk();
        }
        // We want to burn their DSC "debt"
        // And take their collateral
        // Bad User: $140 ETH, $100 DSC
        // debtToCover = $100
        // $100 of DSC == ??? ETH?
        // 0.05 ETH
        uint256 tokenAmountFromDebtCovered = getTokenAmountFromUsd(collateral, debtToCover);
        // And give them a 10% bonus
        // So we are giving the liquidator $110 of WETH for 100 DSC
        // We should implement a feature to liquidate in the event the protocol is insolvent
        // And sweep extra amounts into a treasury
        // 0.05 * 0.1 = 0.005. Getting 0.055
        uint256 bonusCollateral = (tokenAmountFromDebtCovered * LIQUIDATION_BONUS) / LIQUIDATION_PRECISION;
        uint256 totalCollateralToRedeem = tokenAmountFromDebtCovered + bonusCollateral;
        _redeemCollateral(user, msg.sender, collateral, totalCollateralToRedeem);
        // We need to burn the DSC
        _burnDsc(debtToCover, user, msg.sender);

        uint256 endingUserHealthFactor = _healthFactor(user);
        if (endingUserHealthFactor <= startingUserHealthFactor) {
            revert DSCEngine__HealthFactorNotImproved();
        }
        _revertIfHealthFactorIsBroken(msg.sender);
    }
```