<img src="https://pbs.twimg.com/profile_images/1313563688931069952/MtjyDDDh_400x400.jpg" width=100 height=100>

# [PoolTogether](https://code4rena.com/contests/2023-07-pooltogether#top)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| PoolTogether | [Website](https://pooltogether.com/) | [Twitter](https://twitter.com/PoolTogether_) | $121,650 USDC | 3335 | 7 days | Jul 07, 2023 | Jul 14, 2023 |

## What is PoolTogether?

PoolTogether is a prize savings protocol. Yield on deposits is awarded periodically as random prizes. PoolTogether is a gamification layer that can be added to any yield-bearing asset. We like to call them prize-wrapped assets.

## Audit scope

- [claimer/src/libraries/LinearVRGDALib.sol](https://github.com/GenerationSoftware/pt-v5-claimer/blob/57a381aef690a27c9198f4340747155a71cae753/src/libraries/LinearVRGDALib.sol)
-  [claimer/src/Claimer.sol](https://github.com/GenerationSoftware/pt-v5-claimer/blob/57a381aef690a27c9198f4340747155a71cae753/src/Claimer.sol) 
-  [prize-pool/src/abstract/TieredLiquidityDistributor.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/abstract/TieredLiquidityDistributor.sol) 
- [prize-pool/src/libraries/DrawAccumulatorLib.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/libraries/DrawAccumulatorLib.sol)
- [prize-pool/src/libraries/TierCalculationLib.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/libraries/TierCalculationLib.sol)
- [prize-pool/src/libraries/UD34x4.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/libraries/UD34x4.sol) 
- [prize-pool/src/PrizePool.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/PrizePool.sol) 
- [twab-controller/src/libraries/ObservationLib.sol](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/libraries/ObservationLib.sol) 
- [twab-controller/src/libraries/OverflowSafeComparatorLib.sol](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/libraries/OverflowSafeComparatorLib.sol) 
- [twab-controller/src/libraries/TwabLib.sol](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/libraries/TwabLib.sol) 
- [twab-controller/src/TwabController.sol](https://github.com/GenerationSoftware/pt-v5-twab-controller/blob/0145eeac23301ee5338c659422dd6d69234f5d50/src/TwabController.sol) 
- [vault/src/interfaces/IVaultHooks.sol](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/interfaces/IVaultHooks.sol) 
- [vault/src/Vault.sol](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/Vault.sol) 
- [vault/src/VaultFactory.sol](https://github.com/GenerationSoftware/pt-v5-vault/blob/b1deb5d494c25f885c34c83f014c8a855c5e2749/src/VaultFactory.sol)
  
## Issues found by Mlome

| Severity | Title | Count |
|:--|:--|:--:|
| Medium | `claimPrizes()` can be frontrun to get the claiming fees | [M-01] |

# [M-01] - `claimPrizes()` can be frontrun to get the claiming fees

## Impact
A malicious user is able to claim the prizes without having to compete in the bot race

## Proof of concept
When `claimPrizes()` is called to claim the prizes on behalf of the users, it is possible to frontrun this call and get the claiming reward by replacing the `_feeRecipient` address.

```solidity
File: Claimer.Sol

60:   function claimPrizes( // @audit - possible frontrun
    Vault vault,
    uint8 tier,
    address[] calldata winners,
    uint32[][] calldata prizeIndices,
    address _feeRecipient
  ) external returns (uint256 totalFees) {
    uint256 claimCount;
    for (uint i = 0; i < winners.length; i++) {
      claimCount += prizeIndices[i].length;
    }

    uint96 feePerClaim = uint96(
      _computeFeePerClaim(
        _computeMaxFee(tier, prizePool.numberOfTiers()),
        claimCount,
        prizePool.claimCount()
      )
    );

    vault.claimPrizes(tier, winners, prizeIndices, feePerClaim, _feeRecipient);

    return feePerClaim * claimCount;
  }
```
https://github.com/GenerationSoftware/pt-v5-claimer/blob/57a381aef690a27c9198f4340747155a71cae753/src/Claimer.sol#L60


## Recommended Mitigation Steps
The protocol should implement an anti-frontrunning strategy such as the LibSubmarine (https://libsubmarine.org/)