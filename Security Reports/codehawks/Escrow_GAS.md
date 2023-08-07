
Relevant GitHub Links
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L41

https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/EscrowFactory.sol#L56

### Gas Optimizations
| |Issue|Instances|
|-|:-|:-:|
| [GAS-01] | Remove useless check | 1 | 
| [GAS-02] | Function used only once can be inlined | 1 | 
| [GAS-03] | Setting the constructor to payable | 1 |

### [GAS-01] Remove useless check
In the `Escrow.sol` contract's constructor the buyer can not be `address(0)` because it is passed as `msg.sender` in `EscrowFactory::newEscrow`.

```solidity
File: EscrowFactory.sol

40:    Escrow escrow = new Escrow{salt: salt}(
            price,
            tokenContract,
            msg.sender, 
            seller,
            arbiter,
            arbiterFee
        );
```
        
Therefore the following check can be removed:


```solidity
File: Escrow.sol

41:    if (buyer == address(0)) revert Escrow__BuyerZeroAddress();
```

### [GAS-02] Function used only once can be inlined
The function `EscrowFactory::computeEscrowAddress` is used only once, so it can be inlined to save the gas of a function call. The function `computeEscrowAddress()` can be removed and `newEscrow()` modified as such:

```solidity
File: EscrowFactory.sol

20:    function newEscrow(
        uint256 price,
        IERC20 tokenContract,
        address seller,
        address arbiter,
        uint256 arbiterFee,
        bytes32 salt
    ) external returns (IEscrow) {
+        /// @dev See https://docs.soliditylang.org/en/latest/control-structures.html#salted-contract-creations-create2
+        address computedAddress = address(
+            uint160(
+                uint256(
+                    keccak256(
+                        abi.encodePacked(
+                            bytes1(0xff),
+                            deployer,
+                            salt,
+                            keccak256(
+                                abi.encodePacked(
+                                    byteCode, abi.encode(price, tokenContract, buyer, seller, arbiter, arbiterFee)
+                                )
+                            )
+                        )
+                    )
+                )
+            )
+        );
        tokenContract.safeTransferFrom(msg.sender, computedAddress, price);
        Escrow escrow = new Escrow{salt: salt}(
            price,
            tokenContract,
            msg.sender, 
            seller,
            arbiter,
            arbiterFee
        );
        if (address(escrow) != computedAddress) {
            revert EscrowFactory__AddressesDiffer();
        }
        emit EscrowCreated(address(escrow), msg.sender, seller, arbiter);
        return escrow;
    }
```

### [GAS-03] Setting the constructor to `payable`
Saves ~13 gas per instance

_Instance (1):_
```solidity
File: Escrow.sol
32:    constructor(
        uint256 price,
        IERC20 tokenContract,
        address buyer,
        address seller,
        address arbiter,
        uint256 arbiterFee
    ) {
```