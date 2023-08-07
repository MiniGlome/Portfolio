## Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | Functions guaranteed to revert when called by normal users can be marked `payable` | 5 | 
| [GAS-02] | Increments can be `unchecked` | 6 | 
| [GAS-03] | Setting the `constructor` to `payable` | 4 | 
| [GAS-04] | Usage of uint/int smaller than 32 bytes | 30 |  

### [GAS-01] Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier or require such as onlyOwner/onlyX is used, the function will revert if a normal user tries to pay the function. Marking the function as payable will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are `CALLVALUE`(2),`DUP1`(3),`ISZERO`(3),`PUSH2`(3),`JUMPI`(10),`PUSH1`(3),`DUP1`(3),`REVERT`(0),`JUMPDEST`(1),`POP`(2), which costs an average of about 21 gas per call to the function, in addition to the extra deployment cost.

*Instances (5)*:
```solidity
File: canto-namespace-protocol/src/Namespace.sol
196:    function changeNoteAddress(address _newNoteAddress) external onlyOwner {

204:    function changeRevenueAddress(address _newRevenueAddress) external onlyOwner {

```

```solidity
File: canto-namespace-protocol/src/Tray.sol
210:    function changeNoteAddress(address _newNoteAddress) external onlyOwner {

218:    function changeRevenueAddress(address _newRevenueAddress) external onlyOwner {

225:    function endPrelaunchPhase() external onlyOwner {

```

### [GAS-02] Increments can be `unchecked`
Increments in for loops as well as some uint256 iterators cannot realistically overflow as this would require too many iterations, so this can be `unchecked`.
The `unchecked` keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas PER LOOP.

*Instances (6)*:
```solidity
File: canto-bio-protocol/src/Bio.sol
59:            bytesOffset++;

90:                    strLines[insertedLines++] = string(bytesLines);

124:        uint256 tokenId = ++numMinted;

```

```solidity
File: canto-namespace-protocol/src/Namespace.sol
115:        uint256 namespaceIDToMint = ++nextNamespaceIDToMint;

154:                uniqueTrays[numUniqueTrays++] = trayID;

```

```

```solidity
File: canto-pfp-protocol/src/ProfilePicture.sol
80:        uint256 tokenId = ++numMinted;

```

### [GAS-03] Setting the `constructor` to `payable`
Saves ~13 gas per instance

*Instances (4)*:
```solidity
File: canto-bio-protocol/src/Bio.sol
32:    constructor() ERC721("Biography", "Bio") {

```

```solidity
File: canto-namespace-protocol/src/Namespace.sol
73:    constructor(
        address _tray,
        address _note,
        address _revenueAddress
    ) ERC721("Namespace", "NS") Owned(msg.sender) {

```

```solidity
File: canto-namespace-protocol/src/Tray.sol
98:    constructor(
        bytes32 _initHash,
        uint256 _trayPrice,
        address _revenueAddress,
        address _note,
        address _namespaceNFT
    ) ERC721A("Namespace Tray", "NSTRAY") Owned(msg.sender) {

```

```solidity
File: canto-pfp-protocol/src/ProfilePicture.sol
57:    constructor(address _cidNFT, string memory _subprotocolName) ERC721("Profile Picture", "PFP") {

```

### [GAS-04] Usage of uint/int smaller than 32 bytes
When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size. Each operation involving a uint8 costs an extra 22-28 gas (depending on whether the other operand is also a variable of type uint8) as compared to ones involving uint256, due to the compiler having to clear the higher bits of the memory word before operating on the uint8, as well as the associated stack operations of doing so. https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html<br>Use a larger size then downcast where needed.

*Instances (30)*:
```solidity
File: canto-bio-protocol/src/Bio.sol
77:                            uint8(bioTextBytes[i + 4]) >= 187 &&

78:                            uint8(bioTextBytes[i + 4]) <= 191) ||

```

```solidity
File: canto-namespace-protocol/src/Namespace.sol
34:        uint8 tileOffset;

36:        uint8 skinToneModifier;

125:            uint8 tileOffset = _characterList[i].tileOffset;

134:            uint8 characterModifier = tileData.characterModifier;

```

```solidity
File: canto-namespace-protocol/src/Tray.sol
59:        uint8 fontClass;

61:        uint16 characterIndex;

63:        uint8 characterModifier;

195:    function getTile(uint256 _trayId, uint8 _tileOffset) external view returns (TileData memory tileData) {

250:            tileData.characterIndex = uint16(charRandValue % NUM_CHARS_EMOJIS);

252:            tileData.characterIndex = uint16(charRandValue % NUM_CHARS_LETTERS);

255:                tileData.characterIndex = uint16(charRandValue % NUM_CHARS_LETTERS_NUMBERS);

259:                tileData.fontClass = 3 + uint8((res - 80) / 8);

261:                tileData.fontClass = 5 + uint8((res - 96) / 4);

263:                tileData.fontClass = 7 + uint8((res - 104) / 2);

267:                    tileData.characterModifier = uint8(zalgoSeed % 256);

```

```solidity
File: canto-namespace-protocol/src/Utils.sol
12:    error EmojiDoesNotSupportSkinToneModifier(uint16 emojiIndex);

74:        uint8 _fontClass,

75:        uint16 _characterIndex,

131:            uint8 asciiStartingIndex = 97; // Starting index for (lowercase) characters

135:            return abi.encodePacked(bytes1(asciiStartingIndex + uint8(_characterIndex)));

138:            uint8 asciiStartingIndex = 97;

145:            bytes memory character = abi.encodePacked(bytes1(asciiStartingIndex + uint8(_characterIndex)));

215:            return _getUtfSequence(startingSequence, uint8(_characterIndex));

267:    function _getUtfSequence(bytes memory _startingSequence, uint8 _characterIndex)

272:        uint8 lastByteVal = uint8(_startingSequence[3]);

272:        uint8 lastByteVal = uint8(_startingSequence[3]);

273:        uint8 lastByteSum = lastByteVal + _characterIndex;

278:            _startingSequence[2] = bytes1(uint8(_startingSequence[2]) + 1);

```