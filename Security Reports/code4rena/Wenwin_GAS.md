## Gas Optimizations
| |Issue|Instances|
|-|:-|:-:| 
| [GAS-01] | Setting the `constructor` to `payable` | 10 | 
| [GAS-02] | Usage of uint/int smaller than 32 bytes | 39 | 
| [GAS-03] | Use `<`/`>` instead of `>=`/`>=` | 1 | 
| [GAS-04] | Using fixed bytes is cheaper than using `string` | 3 | 
| [GAS-05] | `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables | 8 | 
| [GAS-06] | Unused named return variables | 10 | 

### [GAS-01] Setting the `constructor` to `payable`
Saves ~13 gas per instance

*Instances (10)*:
```solidity
File: Wenwin\Lottery.sol
84:    constructor(
        LotterySetupParams memory lotterySetupParams,
        uint256 playerRewardFirstDraw,
        uint256 playerRewardDecreasePerDraw,
        uint256[] memory rewardsToReferrersPerDraw,
        uint256 maxRNFailedAttempts,
        uint256 maxRNRequestDelay
    )
        Ticket()
        LotterySetup(lotterySetupParams)
        ReferralSystem(playerRewardFirstDraw, playerRewardDecreasePerDraw, rewardsToReferrersPerDraw)
        RNSourceController(maxRNFailedAttempts, maxRNRequestDelay)
    {

```

```solidity
File: Wenwin\LotterySetup.sol
41:    constructor(LotterySetupParams memory lotterySetupParams) {

```

```solidity
File: Wenwin\LotteryToken.sol
17:    constructor() ERC20("Wenwin Lottery", "LOT") {

```

```solidity
File: Wenwin\ReferralSystem.sol
27:    constructor(
        uint256 _playerRewardFirstDraw,
        uint256 _playerRewardDecreasePerDraw,
        uint256[] memory _rewardsToReferrersPerDraw
    ) {

```

```solidity
File: Wenwin\RNSourceBase.sol
11:    constructor(address _authorizedConsumer) {

```

```solidity
File: Wenwin\RNSourceController.sol
26:    constructor(uint256 _maxFailedAttempts, uint256 _maxRequestDelay) {

```

```solidity
File: Wenwin\Ticket.sol
17:    constructor() ERC721("Wenwin Lottery Ticket", "WLT") { }

```

```solidity
File: Wenwin\VRFv2RNSource.sol
13:    constructor(
        address _authorizedConsumer,
        address _linkAddress,
        address _wrapperAddress,
        uint16 _requestConfirmations,
        uint32 _callbackGasLimit
    )
        RNSourceBase(_authorizedConsumer)
        VRFV2WrapperConsumerBase(_linkAddress, _wrapperAddress)
    {

```

```solidity
File: Wenwin\staking\StakedTokenLock.sol
16:    constructor(address _stakedToken, uint256 _depositDeadline, uint256 _lockDuration) {

```

```solidity
File: Wenwin\staking\Staking.sol
22:    constructor(
        ILottery _lottery,
        IERC20 _rewardsToken,
        IERC20 _stakingToken,
        string memory name,
        string memory symbol
    )
        ERC20(name, symbol)
    {

```

### [GAS-02] Usage of uint/int smaller than 32 bytes
When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size. Each operation involving a uint8 costs an extra 22-28 gas (depending on whether the other operand is also a variable of type uint8) as compared to ones involving uint256, due to the compiler having to clear the higher bits of the memory word before operating on the uint8, as well as the associated stack operations of doing so. https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html<br>Use a larger size then downcast where needed.

*Instances (39)*:
```solidity
File: Wenwin\Lottery.sol
37:    mapping(uint128 => mapping(uint8 => uint256)) public override winAmount;

159:    function claimable(uint256 ticketId) external view override returns (uint256 claimableAmount, uint8 winTier) {

211:            for (uint8 winTier = 1; winTier < selectionSize; ++winTier) {

234:    function currentRewardSize(uint8 winTier) public view override returns (uint256 rewardSize) {

238:    function drawRewardSize(uint128 drawId, uint8 winTier) private view returns (uint256 rewardSize) {

```

```solidity
File: Wenwin\LotterySetup.sol
30:    uint8 public immutable override selectionSize;

31:    uint8 public immutable override selectionMax;

120:    function fixedReward(uint8 winTier) public view override returns (uint256 amount) {

126:            uint256 mask = uint256(type(uint16).max) << (winTier * 16);

169:        for (uint8 winTier = 1; winTier < selectionSize; ++winTier) {

170:            uint16 reward = uint16(rewards[winTier] / divisor);

```

```solidity
File: Wenwin\TicketUtils.sol
19:        uint8 selectionSize,

20:        uint8 selectionMax

28:            for (uint8 i = 0; i < selectionMax; ++i) {

45:        uint8 selectionSize,

46:        uint8 selectionMax

54:        uint8[] memory numbers = new uint8[](selectionSize);

58:            numbers[i] = uint8(randomNumber % currentSelectionCount);

66:            uint8 currentNumber = numbers[i];

86:        uint8 selectionSize,

87:        uint8 selectionMax

91:        returns (uint8 winTier)

95:            for (uint8 i = 0; i < selectionMax; ++i) {

96:                winTier += uint8(intersection & uint120(1));

```

```solidity
File: Wenwin\VRFv2RNSource.sol
10:    uint16 public immutable override requestConfirmations;

17:        uint16 _requestConfirmations,

```

```solidity
File: Wenwin\interfaces\ILottery.sol
104:    function winAmount(uint128 drawId, uint8 winTier) external view returns (uint256 amount);

109:    function currentRewardSize(uint8 winTier) external view returns (uint256 rewardSize);

165:    function claimable(uint256 ticketId) external view returns (uint256 claimableAmount, uint8 winTier);

```

```solidity
File: Wenwin\interfaces\ILotterySetup.sol
71:    uint8 selectionSize;

73:    uint8 selectionMax;

93:        uint8 indexed selectionSize,

94:        uint8 indexed selectionMax,

125:    function fixedReward(uint8 winTier) external view returns (uint256 amount);

132:    function selectionSize() external view returns (uint8 size);

137:    function selectionMax() external view returns (uint8 max);

```

```solidity
File: Wenwin\interfaces\IVRFv2RNSource.sol
17:    function requestConfirmations() external returns (uint16 minConfirmations);

```

### [GAS-03] Use `<`/`>` instead of `>=`/`>=`
In Solidity, there is no single op-code for <= or >= expressions. What happens under the hood is that the Solidity compiler executes the LT/GT (less than/greater than) op-code and afterwards it executes an ISZERO op-code to check if the result of the previous comparison (LT/ GT) is zero and validate it or not. Example:
```solidity
// Gas cost: 21394
function check() exernal pure returns (bool) {
		return 3 >= 3;
}
```
```solidity
// Gas cost: 21391
function check() exernal pure returns (bool) {
		return 3 > 2;
}
```
The gas cost between these contract differs by 3 which is the cost executing the ISZERO op-code,**making the use of < and > cheaper than <= and >=.**

*Instance (1)*:
```solidity
File: Wenwin\LotterySetup.sol
51:        if (lotterySetupParams.selectionMax >= 120) {

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

*Instances (3)*:
```solidity
File: Wenwin\RNSourceController.sol
114:        } catch Error(string memory reason) {

```

```solidity
File: Wenwin\staking\Staking.sol
26:        string memory name,

27:        string memory symbol

```

### [GAS-05] `<x> += <y>` Costs More Gas Than `<x> = <x> + <y>` For State Variables
Using the addition operator instead of plus-equals saves **[113 gas](https://gist.github.com/MiniGlome/f462d69a30f68c89175b0ce24ce37cae)**
Same for `-=`, `*=` and `/=`.

*Instances (8)*:
```solidity
File: Wenwin\Lottery.sol
129:        frontendDueTicketSales[frontend] += tickets.length;

275:            currentNetProfit += int256(unclaimedJackpotTickets * winAmount[drawId][selectionSize]);

```

```solidity
File: Wenwin\ReferralSystem.sol
64:                    totalTicketsForReferrersPerDraw[currentDraw] +=

67:                totalTicketsForReferrersPerDraw[currentDraw] += numberOfTickets;

69:            unclaimedTickets[currentDraw][referrer].referrerTicketCount += uint128(numberOfTickets);

71:        unclaimedTickets[currentDraw][player].playerTicketCount += uint128(numberOfTickets);

```

```solidity
File: Wenwin\staking\StakedTokenLock.sol
30:        depositedBalance += amount;

43:        depositedBalance -= amount;

```

### [GAS-06] Unused named return variables
Not using the named return variables when function returns, wastes deployment gas

*Instances (10)*:
```solidity
File: LotterySetup.sol

L122: return _baseJackpot(initialPot);

L124: return 0;

L128: return extracted * (10 ** (IERC20Metadata(address(rewardToken)).decimals() - 1));
```

```solidity
File: Lottery.sol

L234: function currentRewardSize(uint8 winTier) public view override returns (uint256 rewardSize) {
        return drawRewardSize(currentDraw, winTier);
    }

L239: function drawRewardSize(uint128 drawId, uint8 winTier) private view returns (uint256 rewardSize) {
        return LotteryMath.calculateReward(
```

```solidity
File: ReferralSystem.sol

L158: return playerRewardFirstDraw > decrease ? (playerRewardFirstDraw - decrease) : 0;

L162: return rewardsToReferrersPerDraw[Math.min(rewardsToReferrersPerDraw.length - 1, drawId)];
```

```solidity
File: Staking.sol

L51: return rewardPerTokenStored;

L58: return rewardPerTokenStored + (unclaimedRewards * 1e18 / _totalSupply);

L62: return balanceOf(account) * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18 + rewards[account];
```



