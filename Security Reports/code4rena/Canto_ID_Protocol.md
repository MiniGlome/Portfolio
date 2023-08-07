# [Canto ID Protocol](https://code4rena.com/contests/2023-01-canto-identity-protocol-contest#top)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Canto ID Protocol | [Website](https://www.cantoidentity.build/) | [Twitter](https://twitter.com/CantoIdentity) | $36,500 CANTO | 320 | 3 days | Jan 31, 2023 | Feb 3, 2023 |

## What is Canto Identity Protocol?

Canto Identity Protocol (CID) is a permissionless protocol that reinvents the concept of on-chain identity. With CID, the power to control one's on-chain identity is returned to users.
Within Canto Identity Protocol, ERC721 NFTs called cidNFTs represent individual on-chain identities. Users can mint CID NFTs for free by calling the mint method on CidNFT.sol.
Users must register a CID NFT as their canonical on-chain identity with the AddressRegistry.

## Audit scope

- [src/CidNFT.sol](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/CidNFT.sol)
- [src/SubprotocolRegistry.sol](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/SubprotocolRegistry.sol)
- [src/AddressRegistry.sol](https://github.com/code-423n4/2023-01-canto-identity/blob/main/src/AddressRegistry.sol)
  
## Issues found by MiniGlome

| Severity | Title | Count |
|:--|:--|:--:|
| High | Multiple people can register the same cidNFTID | [H-01] |

# [H-01] Multiple people can register the same cidNFTID

## Impact
Multiple people can register the same cidNFT in a way that the same "canonical on-chain identity" can be shared accross multiple real-life identities.

## Proof of Concept
`cidNFT`s can be transfered as any ERC721 token. After each transfer the new owner can `register` the `cidNFT` as his own within the `AddressRegistry` contract. Thus, several addresses can be registered under the same `cidNFTID`.
We can even imagine a malicious smart contract serving as a NFT flash loan reserve for anyone to register the same `cidNFTID`. This goes against the basic concept of the protocol since it allows a shared identity.
Furthermore, people sharing the same registered `cidNFTID` do not have to pay several times the fee to `add` traits from `Subprotocols` to there shared `CidNFT`.
Here is an example of a flash loan pool allowing anyone to share the same `cidNFTID` for free:
```solidity
// SPDX-License-Identifier: GPL-3.0-only
pragma solidity >=0.8.0;

import "./CidNFT.sol";

contract NFTFlahLoan {
	CidNFT nft;

	constructor(address _cidNFT) {
		nft = CidNFT(_cidNFT);
	}

	function flashLoan(uint _id, address _to) external payable {
		require(nft.ownerOf(_id) == address(this), "Wrong id");
		nft.transferFrom(address(this), _to, _id);
		(bool success, ) = address(_to).call{value:msg.value}(abi.encodeWithSignature("receiveFlashLoan(uint256)", _id));
		require(success, "call Failed");
		require(nft.ownerOf(_id) == address(this), "NFT must be returned");
	}
}
```

## Tools Used
Inspection.

## Recommended Mitigation Steps
Remove the ability to register the same CidNFTID twice.
In addition to the `cidNFTs` mapping that links addresses to `cidNFTID`, there could be a mapping that links `cidNFTID`s to owners
```solidity
File: AddressRegistry.sol

mapping(address => uint256) private cidNFTs;
mapping(uint256 => address) private cidNFTsOwners;	// Add this
//...
function register(uint256 _cidNFTID) external {
	if (ERC721(cidNFT).ownerOf(_cidNFTID) != msg.sender)
		revert NFTNotOwnedByUser(_cidNFTID, msg.sender);

	if (cidNFTsOwners[_cidNFTID] != address(0))	// Add this
		revert NFTAlreadyRegistered();		// Add this
	cidNFTsOwners[_cidNFTID] = msg.sender;		// Add this

	cidNFTs[msg.sender] = _cidNFTID;
	emit CIDNFTAdded(msg.sender, _cidNFTID);
}
```