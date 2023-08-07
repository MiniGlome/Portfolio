## Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | Add non-zero check before calling function inside loop | 1 | 
| [GAS-02] | Use `== 0` instead of `<= 0` for uint | 2 | 
| [GAS-03] | Increments can be `unchecked` | 2 | 
| [GAS-04] | Setting the `constructor` to `payable` | 2 | 

### [GAS-01] Add non-zero check before calling function inside loop
The function `getAccountCollateralValue()` calls `getUsdValue()` which is very gas consuming for each token of the `s_collateralTokens` array even if the amount is 0 which case always returns 0.

*Instance (1)*:
```solidity
File: DSCEngine.sol
350:        function getAccountCollateralValue(address user) public view returns (uint256 totalCollateralValueInUsd) {
        // loop through each collateral token, get the amount they have deposited, and map it to
        // the price, to get the USD value
        for (uint256 i = 0; i < s_collateralTokens.length; i++) {
            address token = s_collateralTokens[i];
            uint256 amount = s_collateralDeposited[user][token];
            totalCollateralValueInUsd += getUsdValue(token, amount); // @audit - [GAS] Add non 0 check
        }
        return totalCollateralValueInUsd;
    }

```
Apply this edit to avoid calling `getUsdValue()` on 0 amount

```diff
File: DSCEngine.sol
350:        function getAccountCollateralValue(address user) public view returns (uint256 totalCollateralValueInUsd) {
        // loop through each collateral token, get the amount they have deposited, and map it to
        // the price, to get the USD value
        for (uint256 i = 0; i < s_collateralTokens.length; i++) {
            address token = s_collateralTokens[i];
            uint256 amount = s_collateralDeposited[user][token];
+			if (amount != 0) {
				totalCollateralValueInUsd += getUsdValue(token, amount);
+			}            
        }
        return totalCollateralValueInUsd;
    }

```

### [GAS-02] Use `== 0` instead of `<= 0` for uint
`uint` cannot be less than 0, so `== 0` is a sufficient check

*Instances (2)*:
```solidity
File: DecentralizedStableCoin.sol
48:        if (_amount <= 0) { // @audit - uint256 can not be < 0

61:        if (_amount <= 0) { // @audit - uint256 can not be < 0

```

### [GAS-03] Increments can be `unchecked`
Increments in for loops as well as some uint256 iterators cannot realistically overflow as this would require too many iterations, so this can be `unchecked`.
		The `unchecked` keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas PER LOOP.

*Instances (2)*:
```solidity
File: DSCEngine.sol
118:        for (uint256 i = 0; i < tokenAddresses.length; i++) {

353:        for (uint256 i = 0; i < s_collateralTokens.length; i++) {

```

### [GAS-04] Setting the `constructor` to `payable`
Saves ~13 gas per instance

*Instances (2)*:
```solidity
File: DecentralizedStableCoin.sol
44:    constructor() ERC20("DecentralizedStableCoin", "DSC") {}

```

```solidity
File: DSCEngine.sol
112:    constructor(address[] memory tokenAddresses, address[] memory priceFeedAddresses, address dscAddress) {

```
