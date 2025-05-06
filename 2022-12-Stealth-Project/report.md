## Findings

https://code4rena.com/reports/2022-12-stealth-project

## FINDINGS

NB: Some functions have been truncated where neccessary to just show affected parts of the code
Throught the report some places might be denoted with audit tags to show the actual place affected.

### Pack structs by putting data types that can fit together next to each other

As the solidity EVM works with 32 bytes, variables less than 32 bytes should be packed inside a struct so that they can be stored in the same slot, this saves gas when writing to storage ~20000 gas

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L363-L375

### We can save 2 SLOTS here by packing from 9 SLOTS to 7 SLOTS(Save 2 Storage SLOTS)

```solidity
File: /maverick-v1/contracts/models/Pool.sol

//@audit: move int32 shiftAmount and uint128 firstBinId next to int32 tickLimit;
363:    struct MoveData {
364:        int32 tickLimit;
365:        uint8[2] kinds;
366:        int32 shiftAmount;
367:        BinMap.Active[] activeList;
368:        uint128 firstBinId;
369:        uint256 mergeBinBalance;
370:        uint128 totalReserveA;
371:        uint128 totalReserveB;
372:        uint256 binCounter;
373:        uint256 mergeBinCounter;
374:        uint128[] mergeBins;
375:    }
```

```diff
diff --git a/maverick-v1/contracts/models/Pool.sol b/maverick-v1/contracts/models/Pool.sol
index 9cc0680..d456be8 100644
--- a/maverick-v1/contracts/models/Pool.sol
+++ b/maverick-v1/contracts/models/Pool.sol
@@ -362,10 +362,10 @@ contract Pool is IPool, IPoolAdmin {

     struct MoveData {
         int32 tickLimit;
-        uint8[2] kinds;
         int32 shiftAmount;
-        BinMap.Active[] activeList;
         uint128 firstBinId;
+        uint8[2] kinds;
+        BinMap.Active[] activeList;
         uint256 mergeBinBalance;
         uint128 totalReserveA;
         uint128 totalReserveB;
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/Delta.sol#L5-L20

### Pack from 10 SLOTS to 9 SLOTS Saving 1 SLOT

```solidity
File: /maverick-v1/contracts/libraries/Delta.sol
//@audit: Move all bools together
5:    struct Instance {
6:        uint256 deltaInBinInternal;
7:        uint256 deltaInErc;
8:        uint256 deltaOutErc;
9:        uint256 excess;
10:        bool tokenAIn;
11:        uint256 endSqrtPrice;
12:        bool exactOutput;
13:        bool swappedToMaxPrice;
14:        bool skipCombine;
15:        bool decrementTick;
16:        uint256 sqrtPriceLimit;
17:        uint256 sqrtLowerTickPrice;
18:        uint256 sqrtUpperTickPrice;
19:        uint256 sqrtPrice;
20:    }
```

```diff
diff --git a/maverick-v1/contracts/libraries/Delta.sol b/maverick-v1/contracts/libraries/Delta.sol
index 3e280a8..a4625b7 100644
--- a/maverick-v1/contracts/libraries/Delta.sol
+++ b/maverick-v1/contracts/libraries/Delta.sol
@@ -7,9 +7,9 @@ library Delta {
         uint256 deltaInErc;
         uint256 deltaOutErc;
         uint256 excess;
-        bool tokenAIn;
         uint256 endSqrtPrice;
         bool exactOutput;
+        bool tokenAIn;
         bool swappedToMaxPrice;
         bool skipCombine;
         bool decrementTick;
```

### Cache storage values in memory to minimize SLOADs

The code can be optimized by minimizing the number of SLOADs.

SLOADs are expensive (100 gas after the 1st one) compared to MLOADs/MSTOREs (3 gas each). Storage values read multiple times should instead be cached in memory the first time (costing 1 SLOAD) and then read from this cache to avoid multiple SLOADs.

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/PositionMetadata.sol#L21-L23

### PositionMetadata.sol.tokenURI(): baseURI should be cached

```solidity
File: /contracts/models/PositionMetadata.sol
21:    function tokenURI(uint256 tokenId) external view override returns (string memory) {
22:        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
23:    }
```

```diff
diff --git a/maverick-v1/contracts/models/PositionMetadata.sol b/maverick-v1/contracts/models/PositionMetadata.sol
index 975c585..7d53a0a 100644
--- a/maverick-v1/contracts/models/PositionMetadata.sol
+++ b/maverick-v1/contracts/models/PositionMetadata.sol
@@ -19,6 +19,7 @@ contract PositionMetadata is IPositionMetadata, Ownable {
     }

     function tokenURI(uint256 tokenId) external view override returns (string memory) {
-        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
+        string _baseURI = baseURI;
+        return bytes(_baseURI).length > 0 ? string(abi.encodePacked(_baseURI, tokenId.toString())) : "";
     }
 }
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L549-L551

### Pool.sol.\_amountToBin(): state.protocolFeeRatio should be cached(Saves 1 SLOAD - happy path)

```solidity
File: /maverick-v1/contracts/models/Pool.sol
549:    function _amountToBin(uint256 deltaInErc, uint256 feeBasis) internal view returns (uint256 amount) {
550:        amount = state.protocolFeeRatio != 0 ? Math.clip(deltaInErc, feeBasis.mul(uint256(state.protocolFeeRatio) * PROTOCOL_FEE_SCALE) + 1) : deltaInErc;
551:    }
```

```diff
diff --git a/maverick-v1/contracts/models/Pool.sol b/maverick-v1/contracts/models/Pool.sol
index 9cc0680..fa0731a 100644
--- a/maverick-v1/contracts/models/Pool.sol
+++ b/maverick-v1/contracts/models/Pool.sol
@@ -547,7 +547,8 @@ contract Pool is IPool, IPoolAdmin {
     }

     function _amountToBin(uint256 deltaInErc, uint256 feeBasis) internal view returns (uint256 amount) {
-        amount = state.protocolFeeRatio != 0 ? Math.clip(deltaInErc, feeBasis.mul(uint256(state.protocolFeeRatio) * PROTOCOL_FEE_SCALE) + 1) : deltaInErc;
+        uint16 _stateProtocolFeeRatio = state.protocolFeeRatio;
+        amount = _stateProtocolFeeRatio != 0 ? Math.clip(deltaInErc, feeBasis.mul(uint256(_stateProtocolFeeRatio) * PROTOCOL_FEE_SCALE) + 1) : deltaInErc;
     }
```

### Internal/Private functions only called once can be inlined to save gas

Not inlining costs 20 to 40 gas because of two extra JUMP instructions and additional stack operations needed for function calls.

Affected code:

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L320

```solidity
File: /maverick-v1/contracts/models/Pool.sol
320:    function claimProtocolFees(address recipient, bool isTokenA) internal {

332:    function setProtocolFeeRatio(uint16 _protocolFeeRatio) internal {

448:    function _moveBins(int32 activeTick, int32 startingTickInitial, int32 lastTwap) internal {

486:    function _getOrCreateBin(State memory currentState, uint8 kind, int32 pos, bool isDelta) internal returns (uint128 binId, Bin.Instance storage bin) {

516:    function _currentTickLiquidity(int32 tick) internal view returns (uint256 sqrtLowerTickPrice, uint256 sqrtUpperTickPrice, uint256 sqrtPrice, uint128[] memory binIds, SwapData memory output) {

537:    function _firstKindAtTickLiquidity(int32 activeTick) internal view returns (uint256 currentReserveA, uint256 currentReserveB) {

553:    function _computeSwapExactIn(
554:        uint256 sqrtEdgePrice,
555:        uint256 sqrtPrice,
556:        uint256 liquidity,
557:        uint256 reserveA,
558:        uint256 reserveB,
559:        uint256 amountIn,
560:        bool limitInBin,
561:        bool tokenAIn
562:    ) internal view returns (Delta.Instance memory delta) {

597:    function _computeSwapExactOut(uint256 sqrtPrice, uint256 liquidity, uint256 reserveA, uint256 reserveB, uint256 amountOut, bool tokenAIn) internal view returns (Delta.Instance memory delta) {

614:    function _swapTick(State memory currentState, Delta.Instance memory delta) internal returns (Delta.Instance memory newDelta) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/PoolInspector.sol#L77

```solidity
File: /maverick-v1/contracts/models/PoolInspector.sol

77:    function _getBinsAtTick(IPool pool, int32 tick) internal view returns (IPool.BinState[] memory bins) {

95:    function _currentTickLiquidity(IPool pool) internal view returns (uint256 sqrtPrice, uint256 liquidity, uint256 reserveA, uint256 reserveB) {
```

### Emitting storage values instead of the memory one.

Here, the values emitted shouldn’t be read from storage. The existing memory values should be used instead:

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Position.sol#L23-L26

```solidity
File: /maverick-v1/contracts/models/Position.sol
23:    function setMetadata(IPositionMetadata _metadata) external onlyOwner {
24:        metadata = _metadata;
25:        emit SetMetadata(metadata);
26:    }
```

```diff
diff --git a/maverick-v1/contracts/models/Position.sol b/maverick-v1/contracts/models/Position.sol
index c10d01a..97942be 100644
--- a/maverick-v1/contracts/models/Position.sol
+++ b/maverick-v1/contracts/models/Position.sol
@@ -22,7 +22,7 @@ contract Position is ERC721, ERC721Enumerable, Ownable, IPosition {

     function setMetadata(IPositionMetadata _metadata) external onlyOwner {
         metadata = _metadata;
-        emit SetMetadata(metadata);
+        emit SetMetadata(_metadata);
     }
```

### Using storage instead of memory for structs/arrays saves gas

When fetching data from a storage location, assigning the data to a memory variable causes all fields of the struct/array to be read from storage, which incurs a Gcoldsload (2100 gas) for each field of the struct/array. If the fields are read from the new memory variable, they incur an additional MLOAD rather than a cheap stack read. Instead of declearing the variable with the memory keyword, declaring the variable with the storage keyword and caching any fields that need to be re-read in stack variables, will be much cheaper, only incuring the Gcoldsload for the fields actually read. The only time it makes sense to read the whole struct/array into a memory variable, is if the full struct/array is being returned by the function, is being passed to a function that requires memory, or if the array/struct is being read from another memory array/struct

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L128

```solidity
File: /maverick-v1/contracts/models/Pool.sol
128:            AddLiquidityParams memory param = params[i];

194:            RemoveLiquidityParams memory param = params[i];

225:            RemoveLiquidityParams memory param = params[i];

390:            BinMap.Active memory active = moveData.activeList[i];
```

### Avoid contract existence checks by using solidity version 0.8.10 or later

Prior to 0.8.10 the compiler inserted extra code, including EXTCODESIZE (100 gas), to check for contract existence for external calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value.

**28 Instances identified**

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L87

```solidity
File: /maverick-v1/contracts/models/Pool.sol
87:        require(msg.sender == position.ownerOf(tokenId) || msg.sender == position.getApproved(tokenId), "P");

133:            (temp.deltaA, temp.deltaB, temp.deltaLpBalance) = bin.addLiquidity(
134:                tokenId,
135:                Math.fromScale(param.deltaA, tokenAScale),
136:                Math.fromScale(param.deltaB, tokenBScale),
137:                currentState.activeTick,
138:                temp.reserveA,
139:                temp.reserveB
140:            );

228:            (deltaA, deltaB, deltaLpBalance) = bin.removeLiquidity(bins, tokenId, param.amount);

267:        int32 lastTwa = twa.floor();

342:        require(msg.sender == factory.owner());

426:                (, , moveData.mergeBinBalance) = firstBin.lpTokensFromDeltaReserve(bin.state.reserveA, bin.state.reserveB, activeTick, 0, 0);

449:        int32 floorTwap = twa.getTwaFloor();
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/PoolInspector.sol#L24

```solidity
File: /maverick-v1/contracts/models/PoolInspector.sol
24:        uint128 binCounter = pool.getState().binCounter;

31:        IPool.BinState memory bin = pool.getBin(i + 1);

46:        IPool.BinState memory bin = pool.getBin(binId);

51:        bin = pool.getBin(bin.mergeId);

81:        uint128 binId = pool.binPositions(tick, i);

83:        IPool.BinState memory bin = pool.getBin(binId);

96:        int32 activeTick = pool.getState().activeTick;
97:        uint256 tickSpacing = pool.tickSpacing();
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Position.sol#L29

```solidity
File: /maverick-v1/contracts/models/Position.sol
29:        tokenId = _tokenIdCounter.current();
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/router-v1/contracts/Router.sol#L60

```solidity
File: /router-v1/contracts/Router.sol
60:        uint256 balanceWETH9 = WETH9.balanceOf(address(this));

71:        uint256 balanceToken = token.balanceOf(address(this));

128:        (, amountOut) = pool.swap(recipient, amountIn, tokenAIn, false, sqrtPriceLimitD18, abi.encode(data));

176:        (amountIn, amountOutReceived) = pool.swap(recipient, amountOut, tokenAIn, true, 0, abi.encode(data));

224:                tokenId = IPosition(position).tokenOfOwnerByIndex(msg.sender, 0);

226:                tokenId = IPosition(position).mint(msg.sender);

231:        AddLiquidityCallbackData memory data = AddLiquidityCallbackData({tokenA: pool.tokenA(), tokenB: pool.tokenB(), pool: pool, payer: msg.sender});
232:        (tokenAAmount, tokenBAmount, binDeltas) = pool.addLiquidity(tokenId, params, abi.encode(data));

260:        int32 activeTick = pool.getState().activeTick;

272:            pool = IFactory(factory).create(poolParams.fee, poolParams.tickSpacing, poolParams.lookback, poolParams.activeTick, poolParams.tokenA, poolParams.tokenB);

304:        require(msg.sender == position.ownerOf(tokenId), "P");

307:        (tokenAAmount, tokenBAmount, binDeltas) = pool.removeLiquidity(recipient, tokenId, params);
```

### x += y costs more gas than x = x + y for state variables

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L161-L162

```solidity
File:/maverick-v1/contracts/models/Pool.sol
161:        binBalanceA += tokenAAmount.toUint128();
162:        binBalanceB += tokenBAmount.toUint128();
```

```diff
diff --git a/maverick-v1/contracts/models/Pool.sol b/maverick-v1/contracts/models/Pool.sol
index 9cc0680..360797e 100644
--- a/maverick-v1/contracts/models/Pool.sol
+++ b/maverick-v1/contracts/models/Pool.sol
@@ -158,8 +158,8 @@ contract Pool is IPool, IPoolAdmin {
         uint256 previousABalance = _tokenABalance();
         uint256 previousBBalance = _tokenBBalance();

-        binBalanceA += tokenAAmount.toUint128();
-        binBalanceB += tokenBAmount.toUint128();
+        binBalanceA = binBalanceA + tokenAAmount.toUint128();
+        binBalanceB = binBalanceB + tokenBAmount.toUint128();

```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L288

```solidity
File: /maverick-v1/contracts/models/Pool.sol
288:                binBalanceA += delta.deltaInBinInternal.toUint128();

291:                binBalanceB += delta.deltaInBinInternal.toUint128();
```

### Usage of uints/ints smaller than 32 bytes (256 bits) incurs overhead

When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size.

https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html Use a larger size then downcast where needed

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/router-v1/contracts/Router.sol#L250-L259

```solidity
File: /router-v1/contracts/Router.sol

//@audit:int32 minActiveTick,int32 maxActiveTick
250:    function addLiquidityWTickLimits(
251:        IPool pool,
252:        uint256 tokenId,
253:        IPool.AddLiquidityParams[] calldata params,
254:        uint256 minTokenAAmount,
255:        uint256 minTokenBAmount,
256:        int32 minActiveTick,
257:        int32 maxActiveTick,
258:        uint256 deadline
259:    ) external payable checkDeadline(deadline) returns (uint256 receivingTokenId, uint256 tokenAAmount, uint256 tokenBAmount, IPool.BinDelta[] memory binDeltas) {

//@audit: uint32 maxRecursion
290:    function migrateBinsUpStack(IPool pool, uint128[] calldata binIds, uint32 maxRecursion, uint256 deadline) external checkDeadline(deadline) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L33

```solidity
File: /maverick-v1/contracts/models/Factory.sol

//@audit: uint16 _protocolFeeRatio
33:    function setProtocolFeeRatio(uint16 _protocolFeeRatio) external {

//@audit:uint16 _lookback
46:    function lookup(uint256 _fee, uint256 _tickSpacing, uint16 _lookback, IERC20 _tokenA, IERC20 _tokenB) external view override returns (IPool pool) {

//@audit:uint16 _lookback,int32 _activeTick
58:    function create(uint256 _fee, uint256 _tickSpacing, uint16 _lookback, int32 _activeTick, IERC20 _tokenA, IERC20 _tokenB) external override returns (IPool pool) {

```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L332

```solidity
File: /maverick-v1/contracts/models/Pool.sol

//@audit: uint16 _protocolFeeRatio
332:    function setProtocolFeeRatio(uint16 _protocolFeeRatio) internal {

//@audit: uint16 val
339:    function adminAction(uint256 action, uint16 val, address recipient) external {

//@audit: int32 tick, uint8 kind
358:    function _removeBin(int32 tick, uint8 kind) internal {

//@audit: int32 activeTick
377:    function _moveDirection(int32 activeTick, MoveData memory moveData) internal {

//@audit: int32 activeTick, int32 startingTickInitial, int32 lastTwap
448:    function _moveBins(int32 activeTick, int32 startingTickInitial, int32 lastTwap) internal {

//@audit: uint8 kind, int32 pos
486:    function _getOrCreateBin(State memory currentState, uint8 kind, int32 pos, bool isDelta) internal returns (uint128 binId, Bin.Instance storage bin) {

//@audit: int32 tick
516:    function _currentTickLiquidity(int32 tick) internal view returns (uint256 sqrtLowerTickPrice, uint256 sqrtUpperTickPrice, uint256 sqrtPrice, uint128[] memory binIds, SwapData memory output) {

//@audit: int32 activeTick
537:    function _firstKindAtTickLiquidity(int32 activeTick) internal view returns (uint256 currentReserveA, uint256 currentReserveB) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/PoolInspector.sol#L23

```solidity
File: /maverick-v1/contracts/models/PoolInspector.sol

//@audit: uint128 startBinIndex, uint128 endBinIndex
23:    function getActiveBins(IPool pool, uint128 startBinIndex, uint128 endBinIndex) external view returns (BinInfo[] memory bins) {

//@audit: uint128 binId
45:    function getBinDepth(IPool pool, uint128 binId) public view returns (uint256 depth) {

//@audit: uint128 amount
64:    function calculateSwap(IPool pool, uint128 amount, bool tokenAIn, bool exactOutput) external returns (uint256 amountOut) {

//@audit: int32 tick
77:    function _getBinsAtTick(IPool pool, int32 tick) internal view returns (IPool.BinState[] memory bins) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/Deployer.sol#L12-L23

```solidity
File: /maverick-v1/contracts/libraries/Deployer.sol

//@audit: int32 _activeTick,uint16 _lookback,uint16 _protocolFeeRatio,
12:    function deploy(
13:        uint256 _fee,
14:        uint256 _tickSpacing,
15:        int32 _activeTick,
16:        uint16 _lookback,
17:        uint16 _protocolFeeRatio,
18:        IERC20 _tokenA,
19:        IERC20 _tokenB,
20:        uint256 _tokenAScale,
21:        uint256 _tokenBScale,
22:        IPosition _position
23:    ) external returns (IPool pool) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/BinMap.sol#L20

```solidity
File: /maverick-v1/contracts/libraries/BinMap.sol

//@audit: int32 tick_kinds
20:    function getMapPointer(int32 tick_kinds) internal pure returns (uint8 offset, int32 mapIndex) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/libraries/Bin.sol#L22-L29

```solidity
File: /maverick-v1/contracts/libraries/Bin.sol

//@audit: int32 _activeLowerTick
22:    function lpTokensFromDeltaReserve(
23:        Bin.Instance storage self,
24:        uint256 _deltaA,
25:        uint256 _deltaB,
26:        int32 _activeLowerTick,
27:        uint256 _existingReserveA,
28:        uint256 _existingReserveB
29:    ) internal view returns (uint256 deltaAOptimal, uint256 deltaBOptimal, uint256 proRataLiquidity) {


//@audit: int32 _activeLowerTick
75:    function addLiquidity(
76:        Bin.Instance storage self,
77:        uint256 tokenId,
78:        uint256 _deltaA,
79:        uint256 _deltaB,
80:        int32 _activeLowerTick,
81:        uint256 _existingReserveA,
82:        uint256 _existingReserveB
83:    ) internal returns (uint256 deltaA, uint256 deltaB, uint256 deltaLpToken) {

//@audit: uint32 maxRecursion
94:    function migrateBinsUpStack(Instance storage self, mapping(uint128 => Instance) storage bins, uint32 maxRecursion) internal {

//@audit: uint128 tokenAmount, uint128 reserve, uint128 totalSupply
121:    function adjustAmounts(uint128 tokenAmount, uint128 reserve, uint128 totalSupply) internal pure returns (uint128 delta, uint128 reserveOut) {
```

### Splitting require() statements that use && saves gas - (saves 8 gas per &&)

Instead of using the && operator in a single require statement to check multiple conditions,using multiple require statements with 1 condition per require statement will save 8 GAS per &&
The gas difference would only be realized if the revert condition is realized(met).

**Proof**
**The following tests were carried out in remix with both optimization turned on and off**

```solidity
function multiple (uint a) public pure returns (uint){
	require ( a > 1 && a < 5, "Initialized");
	return  a + 2;
}
```

**Execution cost**
21617 with optimization and using &&
21976 without optimization and using &&

After splitting the require statement

```solidity
function multiple(uint a) public pure returns (uint){
	require (a > 1 ,"Initialized");
	require (a < 5 , "Initialized");
	return a + 2;
}
```

**Execution cost**
21609 with optimization and split require
21968 without optimization and using split require

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/router-v1/contracts/Router.sol#L100

```solidity
File: /router-v1/contracts/Router.sol
100:        require(amountToPay > 0 && amountOut > 0, "In or Out Amount is Zero");

234:        require(tokenAAmount >= minTokenAAmount && tokenBAmount >= minTokenBAmount, "Too little added");

262:        require(activeTick >= minActiveTick && activeTick <= maxActiveTick, "activeTick not in range");

309:        require(tokenAAmount >= minTokenAAmount && tokenBAmount >= minTokenBAmount, "Too little removed");
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/router-v1/contracts/libraries/TransferHelper.sol#L15

```solidity
File: /router-v1/contracts/libraries/TransferHelper.sol
15:        require(success && (data.length == 0 || abi.decode(data, (bool))), "STF");

25:        require(success && (data.length == 0 || abi.decode(data, (bool))), "ST");

35:        require(success && (data.length == 0 || abi.decode(data, (bool))), "SA");
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L41

```solidity
File: /maverick-v1/contracts/models/Factory.sol
41:        require(msg.sender == owner && _owner != address(0), "Factory:NOT_ALLOWED");

60:        require(_fee > 0 && _fee < 1e18, "Factory:FEE_OUT_OF_BOUNDS");

62:        require(_lookback >= 3600 && _lookback <= uint16(type(int16).max), "Factory:LOOKBACK_OUT_OF_BOUNDS");
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L93

```solidity
File: /maverick-v1/contracts/models/Pool.sol
93:        require((currentState.status & LOCKED == 0) && (allowInEmergency || (currentState.status & EMERGENCY == 0)), "E");

168:        require(previousABalance + tokenAAmount <= _tokenABalance() && previousBBalance + tokenBAmount <= _tokenBBalance(), "A");
```

### Using unchecked blocks to save gas - Increments in for loop can be unchecked ( save 30-40 gas per loop iteration)

The majority of Solidity for loops increment a uint256 variable that starts at 0. These increment operations never need to be checked for over/underflow because the variable will never reach the max number of uint256 (will run out of gas long before that happens). The default over/underflow check wastes gas in every iteration of virtually every for loop . eg.

e.g Let's work with a sample loop below.

```solidity
for(uint256 i; i < 10; i++){
//doSomething
}

```

can be written as shown below.

```solidity
for(uint256 i; i < 10;) {
  // loop logic
  unchecked { i++; }
}
```

We can also write it as an inlined function like below.

```solidity
function inc(i) internal pure returns (uint256) {
  unchecked { return i + 1; }
}
for(uint256 i; i < 10; i = inc(i)) {
  // doSomething
}
```

**Affected code**
https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/router-v1/contracts/libraries/Multicall.sol#L13-L26

```solidity
File: /contracts/libraries/Multicall.sol
13:        for (uint256 i = 0; i < data.length; i++) {

26:        }
```

```diff
diff --git a/router-v1/contracts/libraries/Multicall.sol b/router-v1/contracts/libraries/Multicall.sol
index 187c0ef..88ccd52 100644
--- a/router-v1/contracts/libraries/Multicall.sol
+++ b/router-v1/contracts/libraries/Multicall.sol
@@ -10,7 +10,7 @@ abstract contract Multicall is IMulticall {
     /// @inheritdoc IMulticall
     function multicall(bytes[] calldata data) public payable override returns (bytes[] memory results) {
         results = new bytes[](data.length);
-        for (uint256 i = 0; i < data.length; i++) {
+        for (uint256 i = 0; i < data.length;) {
             (bool success, bytes memory result) = address(this).delegatecall(data[i]);

             if (!success) {
@@ -23,6 +23,9 @@ abstract contract Multicall is IMulticall {
             }

             results[i] = result;
+            unchecked {
+                ++i;
+            }
         }
     }
 }
```

**Other Instances to modify**
https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L127

```solidity
File: /maverick-v1/contracts/models/Pool.sol
127:        for (uint256 i; i < params.length; i++) {

180:        for (uint256 i; i < binIds.length; i++) {

193:        for (uint256 i; i < params.length; i++) {

224:        for (uint256 i; i < params.length; i++) {

380:        for (uint256 j; j < 2; j++) {

389:        for (uint256 i; i <= moveData.binCounter; i++) {

414:        for (uint256 i; i < moveData.mergeBinCounter; i++) {

523:        for (uint256 i; i < NUMBER_OF_KINDS; i++) {

540:        for (uint256 i; i < NUMBER_OF_KINDS; i++) {

653:        for (uint256 i; i < swapData.counter; i++) {
```

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/PoolInspector.sol#L101

```solidity
File: /contracts/models/PoolInspector.sol
101:        for (uint256 i; i < bins.length; i++) {
```

[see resource](https://github.com/ethereum/solidity/issues/10695)

### Use shorter revert strings(less than 32 bytes)

Every reason string takes at least 32 bytes so make sure your string fits in 32 bytes or it will become more expensive.Each extra chunk of byetes past the original 32 incurs an MSTORE which costs 3 gas

Shortening revert strings to fit in 32 bytes will decrease deployment time gas and will decrease runtime gas when the revert condition is met.
Revert strings that are longer than 32 bytes require at least one additional mstore, along with additional overhead for computing memory offset, etc.

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Factory.sol#L27

```solidity
File: /maverick-v1/contracts/models/Factory.sol
27:        require(_protocolFeeRatio <= ONE_3_DECIMAL_SCALE, "Factory:PROTOCOL_FEE_CANNOT_EXCEED_ONE");

35:        require(_protocolFeeRatio <= ONE_3_DECIMAL_SCALE, "Factory:PROTOCOL_FEE_CANNOT_EXCEED_ONE");

59:        require(_tokenA < _tokenB, "Factory:TOKENS_MUST_BE_SORTED_BY_ADDRESS");

61:        require(_tickSpacing > 0, "Factory:TICK_SPACING_OUT_OF_BOUNDS");


```
