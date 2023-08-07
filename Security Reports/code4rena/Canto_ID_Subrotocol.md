# [Canto ID Subprotocols](https://code4rena.com/contests/2023-03-canto-identity-subprotocols-contest#top)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| Canto ID Subprotocols | [Website](https://www.cantoidentity.build/) | [Twitter](https://twitter.com/CantoIdentity) | $36,500 USDC | 687 | 3 days | Mar 17, 2023 | Mar 20, 2023 |

## What is Canto Identity Subprotocols?

The audit covers three subprotocols for the Canto Identity Protocol:

Canto Bio Protocol: Allows the association of a biography to an identity
Canto Profile Picture Protocol: Allows the association of a profile picture (arbitrary NFT) to an identity
Canto Namespace Protocol: A subprotocol for minting names from tiles (characters in a specific font).
Each subprotocol is contained in a folder (canto-bio-protocol, canto-namespace-protocol, canto-pfp-protocol) and there is a README in every folder that describes the protocol in more detail.

## Audit scope

- [canto-pfp-protocol/src/ProfilePicture.sol](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-pfp-protocol/src/ProfilePicture.sol)
- [canto-bio-protocol/src/Bio.sol](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-bio-protocol/src/Bio.sol)
- [canto-namespace-protocol/src/Namespace.sol](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/src/Namespace.sol) 
- [canto-namespace-protocol/src/Tray.sol](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/src/Tray.sol) 
- [canto-namespace-protocol/src/Utils.sol](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/src/Utils.sol)
  
## Issues found by MiniGlome

| Severity | Title | Count |
|:--|:--|:--:|
| Medium | XSS via code injection in SVG | [M-01] |

# [M-01] XSS via code injection in SVG

## Impact
SVG is a unique type of image file format that is often susceptible to Cross-site scripting. If a malicious user is able to inject malicious Javascript into a SVG file, then any user who views the SVG on a website will be susceptible to XSS. This can lead stolen cookies, Denial of Service attacks, and more.

The `Bio` contract generates a SVG via the `tokenURI` function. One of the variables used by the `tokenURI` function is `bioText` which is the input given by the user when minting the NFT. When minting a `Bio` NFT, a malicious user can set malicious XSS as the bio text.

These set of circumstances leads to XSS when the SVG is loaded on any website.

## Proof of Concept
- Hacker mint a `Bio` NFT including malicious JavaScript code in the `_bio` argument
- When the SVG is loaded on any site by calling the NFT's `tokenURI` function, any user viewing that SVG will load the malicious Javascript from within the SVG and result in a possible XSS attack if the content is not sanitized by the frontend.

## Tools Used
N/A

## Recommended Mitigation Steps
Creating a SVG file inside of a Solidity contract requires the entity creating a SVG file to sanitize any potential user-input that goes into generating the SVG file.
As of this time there are no known Solidity libraries that sanitize text to prevent an XSS attack. The easiest solution is to remove all user-input data from the SVG file or not generate the SVG at all.