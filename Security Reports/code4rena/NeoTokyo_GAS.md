## Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | State variables can be packed into fewer storage slots | 1 | 
| [GAS-02] | Setting the `constructor` to `payable` | 2 | 
| [GAS-03] | Usage of uint/int smaller than 32 bytes | 3 | 
| [GAS-04] | Using fixed bytes is cheaper than using `string` | 10 | 
| [GAS-05] | `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables | 24 | 

### [GAS-01] State variables can be packed into fewer storage slots
State variables can be packed in slot of 32 bytes to reduce storage usage.
See more: https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html

```Solidity
File: contracts/staking/NeoTokyoStaker.sol

// Layout of state variables BEFORE packing:
address public LP;
uint256 public VAULT_CAP;
uint256 public NO_VAULT_CAP;
mapping ( AssetType => mapping ( uint256 => uint256 )) public	timelockOptions; 
mapping ( AssetType => PoolData ) private _pools;
mapping ( address => mapping ( AssetType => uint256 )) public lastRewardTime;
mapping ( uint256 => mapping ( string => string )) public identityCreditYield; 
mapping ( string => uint256 ) public identityCreditPoints;
mapping ( string => uint256 ) public vaultCreditMultiplier;
mapping ( address => mapping( uint256 => StakedS1Citizen )) public stakedS1;
mapping ( address => uint256[] ) private _stakerS1Position;
mapping ( address => mapping( uint256 => StakedS2Citizen )) public stakedS2;
mapping ( address => uint256[] ) private _stakerS2Position;
mapping ( address => LPPosition ) public stakerLPPosition;
bool public lpLocked;

// Layout of state variables AFTER packing:
address public LP;
bool public lpLocked; // 1 byte bool (lpLocked) is packed with 20 bytes address (LP)
uint256 public VAULT_CAP;
uint256 public NO_VAULT_CAP;
mapping ( AssetType => mapping ( uint256 => uint256 )) public	timelockOptions; 
mapping ( AssetType => PoolData ) private _pools;
mapping ( address => mapping ( AssetType => uint256 )) public lastRewardTime;
mapping ( uint256 => mapping ( string => string )) public identityCreditYield; 
mapping ( string => uint256 ) public identityCreditPoints;
mapping ( string => uint256 ) public vaultCreditMultiplier;
mapping ( address => mapping( uint256 => StakedS1Citizen )) public stakedS1;
mapping ( address => uint256[] ) private _stakerS1Position;
mapping ( address => mapping( uint256 => StakedS2Citizen )) public stakedS2;
mapping ( address => uint256[] ) private _stakerS2Position;
mapping ( address => LPPosition ) public stakerLPPosition;

```

### [GAS-02] Setting the `constructor` to `payable`
Saves ~13 gas per instance

*Instances (2)*:
```solidity
File: contracts/staking/BYTES2.sol
75:	constructor (
		address _bytes,
		address _s1Citizen,
		address _staker,
		address _treasury
	) {

```

```solidity
File: contracts/staking/NeoTokyoStaker.sol
588:	constructor (
		address _bytes,
		address _s1Citizen,
		address _s2Citizen,
		address _lpToken,
		address _identity,
		address _vault,
		uint256 _vaultCap,
		uint256 _noVaultCap
	) {

```

### [GAS-03] Usage of uint/int smaller than 32 bytes
When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size. Each operation involving a uint8 costs an extra 22-28 gas (depending on whether the other operand is also a variable of type uint8) as compared to ones involving uint256, due to the compiler having to clear the higher bits of the memory word before operating on the uint8, as well as the associated stack operations of doing so. https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html<br>Use a larger size then downcast where needed.

*Instances (3)*:
```solidity
File: contracts/staking/NeoTokyoStaker.sol
1205:		if (uint8(_assetType) > 4) {

1668:		if (uint8(_assetType) == 2 || uint8(_assetType) > 4) {

1668:		if (uint8(_assetType) == 2 || uint8(_assetType) > 4) {

```

### [GAS-04] Using fixed bytes is cheaper than using `string`
Use `bytes` for arbitrary-length raw byte data and string for arbitrary-length `string` (UTF-8) data. If you can limit the length to a certain number of bytes, always use one of `bytes1`to `bytes32` because they are much cheaper. Example:
```solidity
// Before
string a;
function add(string str) {
	a = str;
}

// After
bytes32 a;
function add(bytes32 str) public {
	a = str;
}
```

*Instances (10)*:
```solidity
File: contracts/staking/NeoTokyoStaker.sol
326:	mapping ( uint256 => mapping ( string => string )) public identityCreditYield;

326:	mapping ( uint256 => mapping ( string => string )) public identityCreditYield;

329:	mapping ( string => uint256 ) public identityCreditPoints;

332:	mapping ( string => uint256 ) public vaultCreditMultiplier;

628:	) public view returns (string memory) {

639:		string memory vaultMultiplier = (_vaultId != 0)

665:		string memory vaultMultiplier = (_vaultId != 0)

939:		string memory citizenCreditYield = getCreditYield(

949:			string memory class = IGenericGetter(IDENTITY).getClass(identityId);

950:			if (_stringEquals(class, "Hand of Citadel")) {

```

### [GAS-05] `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables
Using the addition operator instead of plus-equals saves **[113 gas](https://gist.github.com/MiniGlome/f462d69a30f68c89175b0ce24ce37cae)**
Same for `-=`, `*=` and `/=`.

*Instances (24)*:
```solidity
File: contracts/staking/NeoTokyoStaker.sol
977:			pool.totalPoints += citizenStatus.points;

1029:			pool.totalPoints += citizenStatus.points;

1078:				citizenStatus.stakedBytes += amount;

1079:				citizenStatus.points += bonusPoints;

1080:				pool.totalPoints += bonusPoints;

1099:				citizenStatus.stakedBytes += amount;

1100:				citizenStatus.points += bonusPoints;

1101:				pool.totalPoints += bonusPoints;

1164:			pool.totalPoints += points;

1515:			pool.totalPoints -= stakedCitizen.points;

1580:			pool.totalPoints -= stakedCitizen.points;

1626:			lpPosition.amount -= amount;

1627:			lpPosition.points -= points;

1630:			pool.totalPoints -= points;

```

