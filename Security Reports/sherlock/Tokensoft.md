![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Ftokensoft.jpg&w=96&q=75)

# [Tokensoft](https://audits.sherlock.xyz/contests/100)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| TokenSoft | [Website](https://www.tokensoft.io/) | [Twitter](https://twitter.com/TokensoftInc) | 14,000 USDC | 650 | 4 days | July 17, 2023 | July 21, 2023 |

## What is Tokensoft?

Tokensoft stands as a prominent provider of enterprise services and tools for blockchain foundations and projects. Our ultimate goal is to restore integrity to financial markets by seamlessly merging existing regulations with decentralized technology. In the dynamic realm of digital assets, security, efficiency, and compliance take center stage. Tokensoft equips teams with robust digital asset solutions that empower them to confidently navigate their challenges with convenience.

## Audit scope


- [contracts/claim/BasicDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/BasicDistributor.sol)
- [contracts/claim/CrosschainContinuousV.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/CrosschainContinuousVestingMerkle.sol)
- [contracts/claim/CrosschainTrancheVesti.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/CrosschainTrancheVestingMerkle.sol)
- [contracts/claim/Satellite.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/Satellite.sol)
- [contracts/claim/abstract/AdvancedDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/AdvancedDistributor.sol)
- [contracts/claim/abstract/CrosschainDist.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/CrosschainDistributor.sol)
- [contracts/claim/abstract/CrosschainMerkleDistributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/CrosschainMerkleDistributor.sol)
- [contracts/claim/abstract/Distributor.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/Distributor.sol)
- [contracts/claim/abstract/MerkleSet.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/MerkleSet.sol)
- [contracts/claim/ContinuousVestingMerkle.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/ContinuousVestingMerkle.sol)
- [contracts/claim/PriceTierVestingMerkle.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingMerkle.sol)
- [contracts/claim/PriceTierVestingSale_2_0.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/PriceTierVestingSale_2_0.sol)
- [contracts/claim/abstract/TrancheVesting.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/claim/abstract/TrancheVesting.sol)
- [contracts/utilities/Sweepable.sol](https://github.com/SoftDAO/contracts/blob/291df55ddb0dbf53c6ed4d5b7432db0c357ca4d3/contracts/utilities/Sweepable.sol)
  
## Issues found by Mlome

| Severity | Title | Count |
|:--|:--|:--:|
| Medium | Airdrop amount superior than type(uint120).max will result in stucked voting tokens | [M-01] |

# [M-01] - Airdrop amount superior than type(uint120).max will result in stucked voting tokens

## Summary
If the airdrop amount is more than `type(uint120).max` the transaction will not revert as expected but will result in voting tokens being stucked.

## Vulnerability Detail
The following check is made on the local variable `totalAmount` that is already downcasted to `uint120` instead of the `uint256 _totalAmount` argument. Hence, the test will always pass.
```solidity
File claim/Distributor.sol

L47:	function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount); // @audit - dangerous downcasting

    // Checks
    //@audit - this check will always pass because the check is on local variable instead of argument
    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }
```

## Impact
An airdrop of more than `type(uint120).max` tokens will not revert as expected. Instead, the `uint120(_totalAmount)` tokens will be distributed and the total amount will be mint as voting tokens in `AdvancedDistributor.sol > _initializeDistributionRecord()`.

```solidity
File claim/abstract/AdvencedDistributor.sol

L77:	function _initializeDistributionRecord(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override {
    super._initializeDistributionRecord(beneficiary, totalAmount);

    // add voting power through ERC20Votes extension
    //@audit - tokensToVotes(totalAmount) is minted
    _mint(beneficiary, tokensToVotes(totalAmount));
  }
```

Then, once claiming the tokens only `uint120(_totalAmount)` voting tokens will be burned is `AdvancedDistributor.sol > _executeClaim()`

```solidity
File claim/abstract/AdvencedDistributor.sol

L87:	function _executeClaim(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount);

    // reduce voting power through ERC20Votes extension
    //@audit - only tokensToVotes(_claimed) amount is burned, i.e. uint120(totalAmount)
    _burn(beneficiary, tokensToVotes(_claimed));
  }
```

This results in `totalAmount - uint120(totalAmount)` voting tokens being stucked and a wrong tracking of voting power.

## Tool used

Manual Review

## Recommendation

Check the argument instead of the local variable:
```diff
File claim/Distributor.sol

L47:	function _initializeDistributionRecord(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual {
    uint120 totalAmount = uint120(_totalAmount); // @audit - dangerous downcasting

    // Checks
-    require(totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');
+    require(_totalAmount <= type(uint120).max, 'Distributor: totalAmount > type(uint120).max');

    // Effects - note that the existing claimed quantity is re-used during re-initialization
    records[beneficiary] = DistributionRecord(true, totalAmount, records[beneficiary].claimed);
    emit InitializeDistributionRecord(beneficiary, totalAmount);
  }
```