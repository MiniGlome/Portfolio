### [S-01] Misleading modifier name
Modifier called `onlyAdminWithdrawAfterEnd` should be called `onlyAfterEnd` because it only checks `endTime`, and has nothing to do with admin privileges.
```solidity
File: Quest.sol

L76:	modifier onlyAdminWithdrawAfterEnd() {
		if (block.timestamp < endTime) revert NoWithdrawDuringClaim();
		_;
	}
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L76

### [L-01] `Erc1155Quest`'s tokens can be withdrawn before every reward has been claimed
#### Impact
The owner can withdraw all the remaining tokens after the Quest `endTime`. Thus, users who have not claimed their reward at the end of the quest may not be able to do so because the tokens can be withdrawn by the owner beforehand.
#### Proof Of Concept
The `withdrawRemainingTokens()` function withdraws all token balance whithout checking unclaimed tokens. 
```solidity
File: Erc1155Quest.sol

L56:	IERC1155(rewardToken).safeTransferFrom(
		address(this),
		to_,
		rewardAmountInWeiOrTokenId,
		IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
		'0x00'
	);
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L56

## Summary
_This report does not include publicly known issues_
### Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | Comparing strings is not gas efficient | 2 | 
| [GAS-02] | `<x> += <y>` costs more gas than `<x> = <x> + <y>` for state variables | 1 | 
| [GAS-03] | `++<x>`/`<x>++` should be `unchecked` when it is no possible for them to overflow | 10 |
| [GAS-04] | Optimize names to save gas | 4 |
| [GAS-05] | `internal` functions only called once can be inlined to save gas | 1 |
| [GAS-06] | Setting the constructor to `payable` | 6 |
| [GAS-07] | Use `</>` instead of `>=/>=` | 3 |
| [GAS-08] | Don't compare boolean expressions to boolean literals | 3 |

Total: 30 instances over 8 issues


## Gas Optimizations
### [GAS-01] Comparing strings is not gas efficient
It is not gas efficient to compare `contractType_` as a string, as the keccak256 function is used twice to convert the string to a bytes32 value, and then to compare the two bytes32 values. It would be more efficient to compare `contractType_` as an integer, which requires less computation and thus less gas.
```solidity
File: QuestFactory.sol
L72:            if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc20')))
L105:           if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc1155')))
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L72
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L105

**Recommended Mitigation Steps**
```solidity
// Costs around 2000 less gas
if (contractType_ == 1)
```
Alternatively `keccak256(abi.encodePacked('erc20')))` and `keccak256(abi.encodePacked('erc1155')))` could be pre-computed.
### [GAS-02]  `<x> += <y>` costs more gas than `<x> = <x> + <y>` for state variables
Using the addition operator instead of plus-equals saves **[113 gas](https://gist.github.com/MiniGlome/f462d69a30f68c89175b0ce24ce37cae)**
```solidity
File: Quest.sol

115:         redeemedTokens += redeemableTokenCount;
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L115

### [GAS-03]  `++<x>`/`<x>++` should be `unchecked` when it is no possible for them to overflow
The unchecked keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas PER LOOP
```solidity
File: QuestFactory.sol

L101:           ++questIdCount;
L132:           ++questIdCount;
L226:           quests[questId_].numberMinted++;
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L101
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L132
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L226

```solidity
File: Quest.sol

L70:            for (uint i = 0; i < tokenIds_.length; i++) {
L104:           for (uint i = 0; i < tokens.length; i++) {
L106:           redeemableTokenCount++;
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L70
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L104
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L106
```solidity
File: RabbitHoleReceipt.sol

L117:           for (uint i = 0; i < msgSenderBalance; i++) {
L121:           foundTokens++;
L128:           for (uint i = 0; i < msgSenderBalance; i++) {
L131:           filterTokensIndexTracker++;
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L117
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L121
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L128
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L131

### [GAS-04]  Optimize names to save gas
Contracts most called functions could simply save gas by function ordering via Method ID. Calling a function at runtime will be cheaper if the function is positioned earlier in the order (has a relatively lower Method ID) because 22 gas are added to the cost of a function for every position that came before it. The caller can save on gas if you prioritize most called functions.

See more [here](https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92)
```
File: Quest.sol
File: Erc20Quest.sol
File: Erc1155Quest.sol
File: RabbitHoleReceipt.sol
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol

**Recommended Mitigation Steps**
Find a lower method ID name for the most called functions for example b_A6Q() is cheaper than b().

### [GAS-05]  `internal` functions only called once can be inlined to save gas
Not inlining costs 20 to 40 gas because of two extra JUMP instructions and additional stack operations needed for function calls.
```solidity
File: QuestFactory.sol

L152:           function grantDefaultAdminAndCreateQuestRole(address account_) internal {
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L152

### [GAS-06] Setting the constructor to `payable`
Saves ~13 gas per instance
```solidity
File: QuestFactory.sol

L35:            constructor() initializer {}
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L35
```solidity
File: Quest.sol

L26:            constructor([...]) {
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L26
```solidity
File: Erc20Quest.sol

L17:            constructor([...]) Quest([...]){
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L17
```solidity
File: Erc1155Quest.sol

L13:            constructor([...]) Quest([...]){
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L13
```solidity
File: RabbitHoleReceipt.sol

L39:            constructor(){
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L39
```solidity
File: RabbitHoleTickets.sol

L28:            constructor(){
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol#L28

### [GAS-07] Use `</>` instead of `>=/>=` 
In Solidity, there is no single op-code for ≤ or ≥ expressions. What happens under the hood is that the Solidity compiler executes the LT/GT (less than/greater than) op-code and afterwards it executes an ISZERO op-code to check if the result of the previous comparison (LT/ GT) is zero and validate it or not. Example:

```solidity
// Gas cost: 21391
function check() exernal pure returns (bool) {
                return 3 > 2;
}
```
```solidity
// Gas cost: 21394
function check() exernal pure returns (bool) {
                return 3 >= 3;
}
```

The gas cost between these contract differs by 3 which is the cost executing the ISZERO op-code, **making the use of < and > cheaper than ≤ and ≥.**
```solidity
File: Quest.sol

L35:            if (endTime_ <= block.timestamp) revert EndTimeInPast();
L36:            if (startTime_ <= block.timestamp) revert StartTimeInPast();
L37:            if (endTime_ <= startTime_) revert EndTimeLessThanOrEqualToStartTime();
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L35
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L36
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L37

### [GAS-08] Don't compare boolean expressions to boolean literals
`if (<x> == true)` => `if (<x>)`, `if (<x> == false)` => `if (!x>)`
```solidity
File: QuestFactory.sol

L73:            if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();
L221:           if (quests[questId_].addressMinted[msg.sender] == true) revert AddressAlreadyMinted();
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L73
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L221
```solidity
File: Quest.sol

L136:           return claimedList[tokenId_] == true;
```
- https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L136

