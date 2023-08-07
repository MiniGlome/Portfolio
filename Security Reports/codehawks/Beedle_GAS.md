# [INFO-01] - Wrong event parameters

## Summary
The `Repaid` event in the `refinance` functino emits the wrong parameters.

## Vulnerability Details
```solidity
File: Lender.sol

L676:	emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
                debt, // @audit - should be `loan.debt` before line 659 where `loans[loanId].debt = debt`
                collateral, // @audit - should be `loan.collateral`
                loan.interestRate,
                loan.startTimestamp
            );
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L676

The `debt` parameter should be replaced by the value of `loan.debt` before it is updated at line 659.
The `collateral` parameter should be replaced by `loan.collateral`.

## Impact
Events with wrong parameter can be misinterpreted by the frontend.

## Tools Used
Manual review

## Recommendations
The event should be emitted before the state change of Line 659 and apply the modifications indicated.

# Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | Increments can be `unchecked` | 25 |
| [GAS-02] | Increments can be `unchecked` | 6 | 
| [GAS-03] | Setting the `constructor` to `payable` | 5 | 
| [GAS-04] | `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables | 11 | 


### [GAS-01] State variables should be cached in memory rather than re-reading them from storage

The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (**100 gas**) with a much cheaper stack or memory read.

*Instances (25)*:
```solidity
File: Lender.sol
145:	if (p.outstandingLoans != pools[poolId].outstandingLoans) // @audit - [GAS] should save pools[poolId] to memory

148:	uint256 currentBalance = pools[poolId].poolBalance; // @audit - [GAS] should save pools[poolId] to memory

167:	if (pools[poolId].lender == address(0)) { // @audit - [GAS] should save pools[poolId] to memory

183:	if (pools[poolId].lender != msg.sender) revert Unauthorized(); // @audit - [GAS] should save pools[poolId] to memory

185:	_updatePoolBalance(poolId, pools[poolId].poolBalance + amount); // @audit - [GAS] should save pools[poolId] to memory

187:	IERC20(pools[poolId].loanToken).transferFrom( // @audit - [GAS] should save pools[poolId] to memory

199:	if (pools[poolId].lender != msg.sender) revert Unauthorized(); // @audit - [GAS] should save pools[poolId] to memory

202:	_updatePoolBalance(poolId, pools[poolId].poolBalance - amount); // @audit - [GAS] should save pools[poolId] to memory

205:	IERC20(pools[poolId].loanToken).transfer(msg.sender, amount); // @audit - [GAS] should save pools[poolId] to memory

265:	_updatePoolBalance(poolId, pools[poolId].poolBalance - debt); // @audit - [GAS] should use `pool` instead of `pools[poolId]`

429:	loans[loanId].debt, // @audit - [GAS] should use `loan` instead of `loans[loanId]`

430:	loans[loanId].collateral, // @audit - [GAS] should use `loan` instead of `loans[loanId]`

481:	if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh(); // @audit - [GAS] should save pools[poolId] to memory

489:	if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall(); // @audit - [GAS] should save pools[poolId] to memory

531:	loans[loanId].debt, // @audit - [GAS] should be `totalDebt`

532:	loans[loanId].collateral, // @audit - [GAS] should be `loan.collateral`

533:	pools[poolId].interestRate, // @audit - [GAS] should be `pool.interestRate`

600:	loans[loanId].lender, // @audit - [GAS] should save loans[loanId] to memory

601:	loans[loanId].loanToken, // @audit - [GAS] should save loans[loanId] to memory

602:	loans[loanId].collateralToken // @audit - [GAS] should save loans[loanId] to memory

639:	_updatePoolBalance(poolId, pools[poolId].poolBalance - debt); // @audit - [GAS] Can be replaced by `pool` 
```

```solidity
File: Staking.sol

65:	if (_balance > balance) { // @audit - [GAS] should cache balance value

66:	uint256 _diff = _balance - balance; // @audit - [GAS] should cache balance value

85:	supplyIndex[recipient] = index; // @audit - [GAS] should cache index value

86:	uint256 _delta = index - _supplyIndex; // @audit - [GAS] should cache index value
```

### [GAS-02] Increments can be `unchecked`
Increments in for loops as well as some uint256 iterators cannot realistically overflow as this would require too many iterations, so this can be `unchecked`.
		The `unchecked` keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas PER LOOP.

*Instances (6)*:
```solidity
File: Lender.sol
233:        for (uint256 i = 0; i < borrows.length; i++) {

293:        for (uint256 i = 0; i < loanIds.length; i++) {

359:        for (uint256 i = 0; i < loanIds.length; i++) {

438:        for (uint256 i = 0; i < loanIds.length; i++) {

549:        for (uint256 i = 0; i < loanIds.length; i++) {

592:        for (uint256 i = 0; i < refinances.length; i++) {

```

### [GAS-03] Setting the `constructor` to `payable`
Saves ~13 gas per instance

*Instances (5)*:
```solidity
File: Beedle.sol
11:    constructor() ERC20("Beedle", "BDL") ERC20Permit("Beedle") Ownable(msg.sender) {

```

```solidity
File: Fees.sol
19:    constructor(address _weth, address _staking) {

```

```solidity
File: Lender.sol
73:    constructor() Ownable(msg.sender) {

```

```solidity
File: Staking.sol
31:    constructor(address _token, address _weth) Ownable(msg.sender) {

```

```solidity
File: utils/Ownable.sol
14:    constructor(address _owner) {

```

### [GAS-04] `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables
Using the addition operator instead of plus-equals saves **[113 gas](https://gist.github.com/MiniGlome/f462d69a30f68c89175b0ce24ce37cae)**
Same for `-=`, `*=` and `/=`.

*Instances (11)*:
```solidity
File: Lender.sol
263:            pools[poolId].outstandingLoans += debt;

314:            pools[poolId].outstandingLoans -= loan.debt;

388:            pools[poolId].outstandingLoans += totalDebt;

400:            pools[oldPoolId].outstandingLoans -= loan.debt;

490:        pools[poolId].outstandingLoans += totalDebt;

502:        pools[oldPoolId].outstandingLoans -= loan.debt;

575:            pools[poolId].outstandingLoans -= loan.debt;

633:            pools[oldPoolId].outstandingLoans -= loan.debt;

637:            pools[poolId].outstandingLoans += debt;

698:            pools[poolId].poolBalance -= debt;

```

