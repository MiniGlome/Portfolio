![](https://code4rena.com/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fcdn-c4-uploads-v0%2Fuploads%2FKZrBoCNmNBd.0&w=256&q=75)

# [RabbitHole](https://code4rena.com/contests/2023-01-rabbithole-quest-protocol-contest#top)

| Protocol | Website | Twitter | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| RabbitHole | [Website](https://rabbithole.gg/) | [Twitter](https://twitter.com/rabbithole_gg) | $36,500 USDC | 752 | 5 days | Jan 25, 2023 | Jan 30, 2023 |

## What is RabbitHole?

**Quests Protocol is a protocol to distribute token rewards for completing on-chain tasks.**

We're releasing five new contracts that are in scope for the audit (more inherited contracts included below):

- [Quest Factory](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol): An upgradable factory that creates a new ERC-20 or ERC-1155 Quest. This follows the factory method & factory creation patterns. This also houses logic on aggregating Quests & minting receipts
- [ERC-20 Quest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol): A Quest that rewards an on-chain action with an ERC-20 token. You must own an unclaimed receipt to be able to claim the reward.
- [ERC-1155 Quest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol): A Quest that rewards an on-chain action with an ERC-1155 token. You must own an unclaimed receipt to be able to claim the reward.
- [Receipt](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol): An ERC-721 contract that is able to be claimed by a user once they have completed an on-chain task. This must be held (and unclaimed) to claim a reward.
- [Ticket](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol): An ERC-1155 contract that is used by the RabbitHole team for ERC-1155 Quests. AKA this is an ERC-1155 Reward.

## Audit scope

- [QuestFactory](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol)
- [RabbitHoleReceipt](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol)
- [Quest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol) 
- [RabbitHoleTickets](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol)
- [Erc20Quest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol) 
- [Erc1155Quest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol)  
- [ReceiptRenderer](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/ReceiptRenderer.sol)
- [TicketRenderer](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/TicketRenderer.sol)  
- [IQuest](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/interfaces/IQuest.sol) 
- [IQuestFactory](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/interfaces/IQuestFactory.sol) 
  
## Issues found by MiniGlome

| Severity | Title | Count |
|:--|:--|:--:|
| Medium | Protocol fee can be withdrawn multiple times | [M-01] |

# [M-01] Protocol fee can be withdrawn multiple times
## Impact
Once the Quest `endTime` is passed, anyone can call the `withdrawFee()` function multiple times. This means that the protocol fee can be sent to the `protocolFeeRecipient` several times, thus draining the Quest `rewardToken`. So the `protocolFeeRecipient` can collect all the remaining funds and tokens that have not already be claimed. Therefore, a malicious actor can prevent any player to claim their `rewardTokens` after the end of the Quest.


## Proof of Concept
Erc20Quest.sol:
```javascript
L102:	function withdrawFee() public onlyAdminWithdrawAfterEnd {
        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
    }
```
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102

```javascript
// Withdraw nearly all the remaining tokens
while(IERC20(rewardToken).balanceOf(address(Erc20Quest)) > Erc20Quest.protocolFee()) {
	Erc20Quest.withdrawFee();
}
```

## Recommended Mitigation Steps
`withdrawFee()` should be callable only once.
```javascript
bool feeWithdrawn;
// [...]
function withdrawFee() public onlyAdminWithdrawAfterEnd {
    require(!feeWithdrawn, "Fees already withdrawn");
    IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
    feeWithdrawn = true;
}
```