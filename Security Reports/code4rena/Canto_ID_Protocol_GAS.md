## Summary
_This report does not include publicly known issues_
### Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | `++i/i++` Should Be `unchecked{++i}`/`unchecked{i++}` When It Is Not Possible For Them To Overflow, As Is The Case When Used In For- And While-loops | 2 | 
| [GAS-02] | Using fixed bytes is cheaper than using `string` | 25 |
| [GAS-03] | Setting the `constructor` to `payable` | 3 |
| [GAS-04] | Using `unchecked` blocks to save gas | 5 |
| [GAS-05] | `require()`/`revert()` statements that check input arguments should be at the top of the function | 5 |

Total: 40 instances over 5 issues

## Gas Optimizations

### [GAS-01] `++i/i++` Should Be `unchecked{++i}`/`unchecked{i++}` When It Is Not Possible For Them To Overflow, As Is The Case When Used In For- And While-loops
The unchecked keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas PER LOOP
```solidity
File: CidNFT.sol
L148:		_mint(msg.sender, ++numMinted);
L150:		for (uint256 i = 0; i < _addList.length; ++i) {
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/CidNFT.sol)

### [GAS-02]  Using fixed bytes is cheaper than using `string`
Use `bytes` for arbitrary-length raw byte data and string for arbitrary-length `string` (UTF-8) data. If you can limit the length to a certain number of bytes, always use one of `bytes1` to `bytes32` because they are much cheaper.
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
Consider limiting Subprotocols name to 32 bytes and replace name `string` to `bytes32`
```solidity
File: SubprotocolRegistry.sol
L40:		mapping(string => SubprotocolData) private subprotocols;
L47:		string indexed name,
L58:		error SubprotocolAlreadyExists(string name, address owner);
L59:		error NoTypeSpecified(string name);
L84:		string calldata _name,
L106:		function getSubprotocol(string calldata _name) external view returns (SubprotocolData memory) {
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/SubprotocolRegistry.sol)
```solidity
File: CidNFT.sol
L70:		mapping(uint256 => mapping(string => SubprotocolData)) internal cidData;
L77:		string indexed subprotocolName,
L80:		event PrimaryDataAdded(uint256 indexed cidNFTID, string indexed subprotocolName, uint256 subprotocolNFTID);
L84:		string indexed subprotocolName,
L90:		string indexed subprotocolName,
L96:		event PrimaryDataRemoved(uint256 indexed cidNFTID, string indexed subprotocolName, uint256 subprotocolNFTID);
L95:		event ActiveDataRemoved(uint256 indexed cidNFTID, string indexed subprotocolName, uint256 subprotocolNFTID);
L102:		error SubprotocolDoesNotExist(string subprotocolName);
L104:		error AssociationTypeNotSupportedForSubprotocol(AssociationType associationType, string subprotocolName);
L107:		error ActiveArrayAlreadyContainsID(uint256 cidNFTID, string subprotocolName, uint256 nftIDToAdd);
L108:		error OrderedValueNotSet(uint256 cidNFTID, string subprotocolName, uint256 key);
L109:		error PrimaryValueNotSet(uint256 cidNFTID, string subprotocolName);
L110:		error ActiveArrayDoesNotContainID(uint256 cidNFTID, string subprotocolName, uint256 nftIDToRemove);
L167:		string calldata _subprotocolName,
L239:		string calldata _subprotocolName,
L297:		string calldata _subprotocolName,
L307:		function getPrimaryData(uint256 _cidNFTID, string calldata _subprotocolName)
L319:		function getActiveData(uint256 _cidNFTID, string calldata _subprotocolName)
L333:		string calldata _subprotocolName,
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/CidNFT.sol)

### [GAS-03] Setting the `constructor` to `payable`
Saves ~13 gas per instance
```solidity
File: AddressRegistry.sol
L36:		constructor(address _cidNFT) {
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/AddressRegistry.sol#L36)
```solidity
File: CidNFT.sol
L119:		constructor(
        string memory _name,
        string memory _symbol,
        string memory _baseURI,
        address _cidFeeWallet,
        address _noteContract,
        address _subprotocolRegistry
    ) ERC721(_name, _symbol) {
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/CidNFT.sol#L119)
```solidity
File: SubprotocolRegistry.sol
L65:		constructor(address _noteContract, address _cidFeeWallet) {
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/SubprotocolRegistry.sol#L65)

### [GAS-04] Using `unchecked` blocks to save gas
Solidity version 0.8+ comes with implicit overflow and underflow checks on unsigned integers. When an overflow or an underflow isn’t possible (as an example, when a comparison is made before the arithmetic operation), some gas can be saved by using an `unchecked` block.
```solidity
File: CidNFT.sol
L148:		_mint(msg.sender, ++numMinted);
L150:		for (uint256 i = 0; i < _addList.length; ++i) {
L179:		uint256 befSwapLastNFTID = activeData.values[arrayLength - 1];
L180:		activeData.values[arrayPosition - 1] = befSwapLastNFTID;
L225:		activeData.positions[_nftIDToAdd] = lengthBeforeAddition + 1;
```

### [GAS-05] `require()`/`revert()` statements that check input arguments should be at the top of the function
Checks that involve constants should come before checks that involve state variables, function calls, and calculations. By doing these checks first, the function is able to revert before wasting a Gcoldsload (2100 gas) in a function that may ultimately revert in the unhappy case.
```solidity
File: CidNFT.sol
L178:		if (
            cidNFTOwner != msg.sender &&
            getApproved[_cidNFTID] != msg.sender &&
            !isApprovedForAll[cidNFTOwner][msg.sender]
        ) revert NotAuthorizedForCIDNFT(msg.sender, _cidNFTID, cidNFTOwner);
L183:		if (_nftIDToAdd == 0) revert NFTIDZeroDisallowedForSubprotocols();
L250:		if (
            cidNFTOwner != msg.sender &&
            getApproved[_cidNFTID] != msg.sender &&
            !isApprovedForAll[cidNFTOwner][msg.sender]
        ) revert NotAuthorizedForCIDNFT(msg.sender, _cidNFTID, cidNFTOwner);
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/CidNFT.sol)
```solidity
File: SubprotocolRegistry.sol
L88:		if (!(_ordered || _primary || _active)) revert NoTypeSpecified(_name);
L94:		if (!ERC721(_nftAddress).supportsInterface(type(CidSubprotocolNFT).interfaceId))
            revert NotASubprotocolNFT(_nftAddress);
```
[Link to code](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/SubprotocolRegistry.sol)
