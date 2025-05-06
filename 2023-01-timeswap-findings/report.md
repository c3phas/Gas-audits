## Findings

https://code4rena.com/reports/2023-01-timeswap

### Table of contents

- [FINDINGS](#findings)
- [The result of a function call should be cached rather than re-calling the function](#the-result-of-a-function-call-should-be-cached-rather-than-re-calling-the-function)
  - [TimeswapV2Token.sol.burn(): The results of timeswapV2TokenPosition.toKey() should be cached rather than call it three times](#timeswapv2tokensolburn--the-results-of-timeswapv2tokenpositiontokey-should-be-cached-rather-than-call-it-three-times)
- [Multiple accesses of a mapping/array should use a local variable cache](#multiple-accesses-of-a-mappingarray-should-use-a-local-variable-cache)
  - [TimeswapV2Pool.sol.addPoolEnumerationIfNecessary(): reentrancyGuards\[strike\]\[maturity\] should be cached in local storage](#timeswapv2poolsoladdpoolenumerationifnecessary-reentrancyguardsstrikematurity-should-be-cached-in-local-storage)
  - [ERC1155Enumerable.sol.\_addTokenEnumeration(): \_idTotalSupply\[id\] should be cached in local storage](#erc1155enumerablesol_addtokenenumeration-_idtotalsupplyid-should-be-cached-in-local-storage)
  - [ERC1155Enumerable.sol.\_removeTokenEnumeration(): \_idTotalSupply\[id\] should be cached in local storage](#erc1155enumerablesol_removetokenenumeration-_idtotalsupplyid-should-be-cached-in-local-storage)
  - [ERC1155Enumerable.sol.\_addTokenToOwnerEnumeration(): \_currentIndex\[to\] should be cached in local storage](#erc1155enumerablesol_addtokentoownerenumeration-_currentindexto-should-be-cached-in-local-storage)
  - [TimeswapV2LiquidityToken.sol.collect(): \_feesPositions\[id\]\[msg.sender\] should be cached in local storage](#timeswapv2liquiditytokensolcollect-_feespositionsidmsgsender-should-be-cached-in-local-storage)
- [Emitting storage values instead of the memory one.](#emitting-storage-values-instead-of-the-memory-one)
- [Multiple address mappings can be combined into a single mapping of an address to a struct, where appropriate](#multiple-address-mappings-can-be-combined-into-a-single-mapping-of-an-address-to-a-struct-where-appropriate)
- [Using storage instead of memory for structs/arrays saves gas](#using-storage-instead-of-memory-for-structsarrays-saves-gas)
- [Using unchecked blocks to save gas](#using-unchecked-blocks-to-save-gas)
- [require() or revert() statements that check input arguments should be at the top of the function](#require-or-revert-statements-that-check-input-arguments-should-be-at-the-top-of-the-function)
  - [Reorder the if's to fail cheaply incase of a revert(Saves ~80 to 100 gas)](#reorder-the-ifs-to-fail-cheaply-incase-of-a-revertsaves-80-to-100-gas)
- [Avoid contract existence checks by using solidity version 0.8.10 or later](#avoid-contract-existence-checks-by-using-solidity-version-0810-or-later)
- [x += y costs more gas than x = x + y for state variables](#x--y-costs-more-gas-than-x--x--y-for-state-variables)
- [Usage of uints/ints smaller than 32 bytes (256 bits) incurs overhead](#usage-of-uintsints-smaller-than-32-bytes-256-bits-incurs-overhead)
- [Use a more recent version of solidity](#use-a-more-recent-version-of-solidity)

## FINDINGS

NB: Some functions have been truncated where necessary to just show affected parts of the code
Throughout the report some places might be denoted with audit tags to show the actual place affected.

## The result of a function call should be cached rather than re-calling the function

External calls are expensive. Consider caching the following:

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2Token.sol#L198-L269

### TimeswapV2Token.sol.burn(): The results of timeswapV2TokenPosition.toKey() should be cached rather than call it three times

```solidity
File: /packages/v2-token/src/TimeswapV2Token.sol
    function burn(TimeswapV2TokenBurnParam calldata param) external override returns (bytes memory data) {

        if (param.long0Amount != 0) {

            _burn(msg.sender, _timeswapV2TokenPositionIds[timeswapV2TokenPosition.toKey()], param.long0Amount);
        } //@audit:timeswapV2TokenPosition.toKey() called here

        if (param.long1Amount != 0) {

            _burn(msg.sender, _timeswapV2TokenPositionIds[timeswapV2TokenPosition.toKey()], param.long1Amount);
        }//@audit:timeswapV2TokenPosition.toKey() called here

        if (param.shortAmount != 0) {

            _burn(msg.sender, _timeswapV2TokenPositionIds[timeswapV2TokenPosition.toKey()], param.shortAmount);
        }//@audit:timeswapV2TokenPosition.toKey() called here
```

## Multiple accesses of a mapping/array should use a local variable cache

Caching a mapping's value in a local storage or calldata variable when the value is accessed multiple times saves ~42 gas per access due to not having to perform the same offset calculation every time.
**Help the Optimizer by saving a storage variable's reference instead of repeatedly fetching it**

To help the optimizer,declare a storage type variable and use it instead of repeatedly fetching the reference in a map or an array.
As an example, instead of repeatedly calling `someMap[someIndex]`, save its reference like this: `SomeStruct storage someStruct = someMap[someIndex]` and use it.

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L57-L62

### TimeswapV2Pool.sol.addPoolEnumerationIfNecessary(): reentrancyGuards\[strike]\[maturity] should be cached in local storage

```solidity
File: /packages/v2-pool/src/TimeswapV2Pool.sol
57:    function addPoolEnumerationIfNecessary(uint256 strike, uint256 maturity) private {
58:        if (reentrancyGuards[strike][maturity] == ReentrancyGuard.NOT_INTERACTED) {
59:            reentrancyGuards[strike][maturity] = ReentrancyGuard.NOT_ENTERED;
60:            listOfPools.push(StrikeAndMaturity({strike: strike, maturity: maturity}));
61:        }
62:    }
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L58-L67

### ERC1155Enumerable.sol.\_addTokenEnumeration(): \_idTotalSupply\[id] should be cached in local storage

```solidity
File: /packages/v2-token/src/base/ERC1155Enumerable.sol
58:    function _addTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
59:        if (from == address(0)) {
60:            if (_idTotalSupply[id] == 0 && _additionalConditionAddTokenToAllTokensEnumeration(id)) _addTokenToAllTokensEnumeration(id);
61:            _idTotalSupply[id] += amount;
62:        }
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L92-L101

### ERC1155Enumerable.sol.\_removeTokenEnumeration(): \_idTotalSupply[id] should be cached in local storage

```solidity
File: /packages/v2-token/src/base/ERC1155Enumerable.sol
92:    function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
93:        if (to == address(0)) {
94:            if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
95:            _idTotalSupply[id] -= amount;
96:        }
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L116-L121

### ERC1155Enumerable.sol.\_addTokenToOwnerEnumeration(): \_currentIndex\[to] should be cached in local storage

```solidity
File: /packages/v2-token/src/base/ERC1155Enumerable.sol
116:    function _addTokenToOwnerEnumeration(address to, uint256 tokenId) private {
117:        _currentIndex[to] += 1; //@audit: Initial access
118:        uint256 length = _currentIndex[to]; //@audit: 2nd access
119:        _ownedTokens[to][length] = tokenId;
120:        _ownedTokensIndex[tokenId] = length;
121:    }
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2LiquidityToken.sol#L182-L222

### TimeswapV2LiquidityToken.sol.collect(): \_feesPositions\[id][msg.sender] should be cached in local storage

```solidity
File: /packages/v2-token/src/TimeswapV2LiquidityToken.sol
182:    function collect(TimeswapV2LiquidityTokenCollectParam calldata param) external returns (uint256 long0Fees, uint256 long1Fees, uint256 shortFees, bytes memory data) {

199:        (long0Fees, long1Fees, shortFees) = _feesPositions[id][msg.sender].getFees(param.long0FeesDesired, param.long1FeesDesired, param.shortFeesDesired);

216:        _feesPositions[id][msg.sender].burn(long0Fees, long1Fees, shortFees);
```

## Emitting storage values instead of the memory one.

Here, the values emitted shouldn’t be read from storage. The existing memory values should be used instead:
https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/base/OwnableTwoSteps.sol#L23-L32

```solidity
File: /packages/v2-pool/src/base/OwnableTwoSteps.sol
23:    function setPendingOwner(address chosenPendingOwner) external override {
24:        Ownership.checkIfOwner(owner);

29:        pendingOwner = chosenPendingOwner;

31:        emit SetOwner(pendingOwner);
32:    }
```

```diff
diff --git a/packages/v2-pool/src/base/OwnableTwoSteps.sol b/packages/v2-pool/src/base/OwnableTwoSteps.sol
index 6edc3cf..cda191e 100644
--- a/packages/v2-pool/src/base/OwnableTwoSteps.sol
+++ b/packages/v2-pool/src/base/OwnableTwoSteps.sol
@@ -28,7 +28,7 @@ contract OwnableTwoSteps is IOwnableTwoSteps {

         pendingOwner = chosenPendingOwner;

-        emit SetOwner(pendingOwner);
+        emit SetOwner(chosenPendingOwner);
     }

```

## Multiple address mappings can be combined into a single mapping of an address to a struct, where appropriate

Saves a storage slot for the mapping. Depending on the circumstances and sizes of types, can avoid a Gsset (20000 gas) per mapping combined. Reads and subsequent writes can also be cheaper when a function requires both values and they both fit in the same storage slot. Finally, if both fields are accessed in the same function, can save ~42 gas per access due to not having to recalculate the key's keccak256 hash (Gkeccak256 - 30 gas) and that calculation's associated stack operations.

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L15-L17

```solidity
File: /packages/v2-token/src/base/ERC1155Enumerable.sol
15:    mapping(address => mapping(uint256 => uint256)) private _ownedTokens; // An index of all tokens
16:
17:    mapping(address => uint256) private _currentIndex; // the current index for an address
```

## Using storage instead of memory for structs/arrays saves gas

When fetching data from a storage location, assigning the data to a memory variable causes all fields of the struct/array to be read from storage, which incurs a Gcoldsload (2100 gas) for each field of the struct/array. If the fields are read from the new memory variable, they incur an additional MLOAD rather than a cheap stack read. Instead of declearing the variable with the memory keyword, declaring the variable with the storage keyword and caching any fields that need to be re-read in stack variables, will be much cheaper, only incuring the Gcoldsload for the fields actually read. The only time it makes sense to read the whole struct/array into a memory variable, is if the full struct/array is being returned by the function, is being passed to a function that requires memory, or if the array/struct is being read from another memory array/struct

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2LiquidityToken.sol#L238

```solidity
File: /packages/v2-token/src/TimeswapV2LiquidityToken.sol
238:            TimeswapV2LiquidityTokenPosition memory timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];
```

```diff
-            TimeswapV2LiquidityTokenPosition memory timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];
+            TimeswapV2LiquidityTokenPosition storage timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];

```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2LiquidityToken.sol#L264

```solidity
File: /packages/v2-token/src/TimeswapV2LiquidityToken.sol
264:        TimeswapV2LiquidityTokenPosition memory timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];
```

```diff
diff --git a/packages/v2-token/src/TimeswapV2LiquidityToken.sol b/packages/v2-token/src/TimeswapV2LiquidityToken.sol
index 2f71a25..b8fd910 100644
--- a/packages/v2-token/src/TimeswapV2LiquidityToken.sol
+++ b/packages/v2-token/src/TimeswapV2LiquidityToken.sol
@@ -261,7 +261,7 @@ contract TimeswapV2LiquidityToken is ITimeswapV2LiquidityToken, ERC1155Enumerabl
     }

     function _additionalConditionForOwnerTokenEnumeration(address owner, uint256 id) private view returns (bool) {
-        TimeswapV2LiquidityTokenPosition memory timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];
+        TimeswapV2LiquidityTokenPosition storage timeswapV2LiquidityTokenPosition = _timeswapV2LiquidityTokenPositions[id];
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2LiquidityToken.sol#L264

```solidity
File: /packages/v2-token/src/TimeswapV2LiquidityToken.sol
275:        FeesPosition memory feesPosition = _feesPositions[id][owner];
```

```diff
diff --git a/packages/v2-token/src/TimeswapV2LiquidityToken.sol b/packages/v2-token/src/TimeswapV2LiquidityToken.sol
index 2f71a25..2bf53d0 100644
--- a/packages/v2-token/src/TimeswapV2LiquidityToken.sol
+++ b/packages/v2-token/src/TimeswapV2LiquidityToken.sol
@@ -272,7 +272,7 @@ contract TimeswapV2LiquidityToken is ITimeswapV2LiquidityToken, ERC1155Enumerabl
             (long0FeeGrowth, long1FeeGrowth, shortFeeGrowth) = ITimeswapV2Pool(poolPair).feeGrowth(timeswapV2LiquidityTokenPosition.strike, timeswapV2LiquidityTokenPosition.maturity);
         }

-        FeesPosition memory feesPosition = _feesPositions[id][owner];
+        FeesPosition storage feesPosition = _feesPositions[id][owner];

```

## Using unchecked blocks to save gas

Solidity version 0.8+ comes with implicit overflow and underflow checks on unsigned integers. When an overflow or an underflow isn’t possible (as an example, when a comparison is made before the arithmetic operation), some gas can be saved by using an unchecked block
[see resource](https://github.com/ethereum/solidity/issues/10695)

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/libraries/ConstantProduct.sol#L319

```solidity
File: /packages/v2-pool/src/libraries/ConstantProduct.sol
319:        else if (rate > deltaRate) newRate = rate - deltaRate;
```

The operation `rate - deltaRate` cannot underflow due to the check `rate > deltaRate`

## require() or revert() statements that check input arguments should be at the top of the function

Checks that involve constants should come before checks that involve state variables, function calls, and calculations. By doing these checks first, the function is able to revert before wasting a Gcoldsload (2100 gas) in a function that may ultimately revert in the unhappy case.

**I've made some notes before the diffs explaining the how and why**

### Reorder the if's to fail cheaply incase of a revert(Saves ~80 to 100 gas)

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L152-L162

```solidity
File: /packages/v2-pool/src/TimeswapV2Pool.sol
152:    function transferLiquidity(uint256 strike, uint256 maturity, address to, uint160 liquidityAmount) external override {
153:        hasLiquidity(strike, maturity);

155:        if (blockTimestamp(0) > maturity) Error.alreadyMatured(maturity, blockTimestamp(0));
156:        if (to == address(0)) Error.zeroAddress();
157:        if (liquidityAmount == 0) Error.zeroInput();
```

It's cheaper to evaluate `liquidityAmount == 0` and `to == address(0)` as compared to `blockTimestamp(0) > maturity` thus we should restructure the if's to take advantage of the fail cheaply concept ie We don't have to waste gas computing the check `blockTimestamp(0) > maturity` if we might end up reverting on `liquidityAmount == 0`

```diff
diff --git a/packages/v2-pool/src/TimeswapV2Pool.sol b/packages/v2-pool/src/TimeswapV2Pool.sol
index dfedce9..e130434 100644
--- a/packages/v2-pool/src/TimeswapV2Pool.sol
+++ b/packages/v2-pool/src/TimeswapV2Pool.sol
@@ -151,10 +151,9 @@ contract TimeswapV2Pool is ITimeswapV2Pool, NoDelegateCall {
     /// @inheritdoc ITimeswapV2Pool
     function transferLiquidity(uint256 strike, uint256 maturity, address to, uint160 liquidityAmount) external override {
         hasLiquidity(strike, maturity);
-
-        if (blockTimestamp(0) > maturity) Error.alreadyMatured(maturity, blockTimestamp(0));
         if (to == address(0)) Error.zeroAddress();
         if (liquidityAmount == 0) Error.zeroInput();
+        if (blockTimestamp(0) > maturity) Error.alreadyMatured(maturity, blockTimestamp(0));

         pools[strike][maturity].transferLiquidity(to, liquidityAmount, blockTimestamp(0));
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L175-L181

```solidity
File: /packages/v2-pool/src/TimeswapV2Pool.sol
175:    function initialize(uint256 strike, uint256 maturity, uint160 rate) external override noDelegateCall {
176:        if (maturity < blockTimestamp(0)) Error.alreadyMatured(maturity, blockTimestamp(0));
177:        if (rate == 0) Error.cannotBeZero();
```

Cheaper to evaluate `rate == 0` compared to `maturity < blockTimestamp(0)` thus we should start with the cheaper option to minimize the gas spent in case of an early revert

```diff
diff --git a/packages/v2-pool/src/TimeswapV2Pool.sol b/packages/v2-pool/src/TimeswapV2Pool.sol
index dfedce9..83fa7c8 100644
--- a/packages/v2-pool/src/TimeswapV2Pool.sol
+++ b/packages/v2-pool/src/TimeswapV2Pool.sol
@@ -173,8 +173,8 @@ contract TimeswapV2Pool is ITimeswapV2Pool, NoDelegateCall {

     /// @inheritdoc ITimeswapV2Pool
     function initialize(uint256 strike, uint256 maturity, uint160 rate) external override noDelegateCall {
-        if (maturity < blockTimestamp(0)) Error.alreadyMatured(maturity, blockTimestamp(0));
         if (rate == 0) Error.cannotBeZero();
+        if (maturity < blockTimestamp(0)) Error.alreadyMatured(maturity, blockTimestamp(0));
         addPoolEnumerationIfNecessary(strike, maturity);

         pools[strike][maturity].initialize(rate);
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-option/src/TimeswapV2Option.sol#L97-L106

```solidity
File: /packages/v2-option/src/TimeswapV2Option.sol
97:    function transferPosition(uint256 strike, uint256 maturity, address to, TimeswapV2OptionPosition position, uint256 amount) external override {
98:        if (!hasInteracted[strike][maturity]) Error.inactiveOptionChoice(strike, maturity);
99:        if (to == address(0)) Error.zeroAddress();
100:        if (amount == 0) Error.zeroInput();
101:        PositionLibrary.check(position);
```

Cheaper here to evaluate `amount == 0` and `to == address(0)` compared to `!hasInteracted[strike][maturity]`.

```diff
diff --git a/packages/v2-option/src/TimeswapV2Option.sol b/packages/v2-option/src/TimeswapV2Option.sol
index c25fc61..f03a05a 100644
--- a/packages/v2-option/src/TimeswapV2Option.sol
+++ b/packages/v2-option/src/TimeswapV2Option.sol
@@ -95,9 +95,10 @@ contract TimeswapV2Option is ITimeswapV2Option, NoDelegateCall {

     /// @inheritdoc ITimeswapV2Option
     function transferPosition(uint256 strike, uint256 maturity, address to, TimeswapV2OptionPosition position, uint256 amount) external override {
-        if (!hasInteracted[strike][maturity]) Error.inactiveOptionChoice(strike, maturity);
-        if (to == address(0)) Error.zeroAddress();
         if (amount == 0) Error.zeroInput();
+        if (to == address(0)) Error.zeroAddress();
+        if (!hasInteracted[strike][maturity]) Error.inactiveOptionChoice(strike, maturity);
+
         PositionLibrary.check(position);

         options[strike][maturity].transferPosition(to, position, amount);
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-library/src/FullMath.sol#L62-L69

```solidity
File: /packages/v2-library/src/FullMath.sol
62:    function sub512(uint256 minuend0, uint256 minuend1, uint256 subtrahend0, uint256 subtrahend1) internal pure returns (uint256 difference0, uint256 difference1) {
63:        assembly {
64:            difference0 := sub(minuend0, subtrahend0)
65:            difference1 := sub(sub(minuend1, subtrahend1), lt(minuend0, subtrahend0))
66:        }

68:        if (subtrahend1 > minuend1 || (subtrahend1 == minuend1 && subtrahend0 > minuend0)) revert SubUnderflow(minuend0, minuend1, subtrahend0, subtrahend1);
69:    }
```

In the above as we have a check that doesn't depend on the results of the operations in assembly ie `difference0 and difference1 ` we can move the check above the assembly code. This way , if we end u reverting due to one of our conditions failing, we wouln't waste any gas doing the assembly operation

```diff
diff --git a/packages/v2-library/src/FullMath.sol b/packages/v2-library/src/FullMath.sol
index a0fba43..c1f8776 100644
--- a/packages/v2-library/src/FullMath.sol
+++ b/packages/v2-library/src/FullMath.sol
@@ -60,12 +60,13 @@ library FullMath {
     /// @return difference0 The least significant part of difference.
     /// @return difference1 The most significant part of difference.
     function sub512(uint256 minuend0, uint256 minuend1, uint256 subtrahend0, uint256 subtrahend1) internal pure returns (uint256 difference0, uint256 difference1) {
+        if (subtrahend1 > minuend1 || (subtrahend1 == minuend1 && subtrahend0 > minuend0)) revert SubUnderflow(minuend0, minuend1, subtrahend0, subtrahend1);
         assembly {
             difference0 := sub(minuend0, subtrahend0)
             difference1 := sub(sub(minuend1, subtrahend1), lt(minuend0, subtrahend0))
         }

-        if (subtrahend1 > minuend1 || (subtrahend1 == minuend1 && subtrahend0 > minuend0)) revert SubUnderflow(minuend0, minuend1, subtrahend0, subtrahend1);
+
     }

     /// @dev Calculate the product of two uint256 numbers that may result to uint512 product.
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-library/src/FullMath.sol#L181-L258

```solidity
File: /packages/v2-library/src/FullMath.sol
181:    function mulDiv(uint256 multiplicand, uint256 multiplier, uint256 divisor, bool roundUp) internal pure returns (uint256 result) {
182:        (uint256 product0, uint256 product1) = mul512(multiplicand, multiplier);

184:        // Handle non-overflow cases, 256 by 256 division
185:        if (product1 == 0) return result = product0.div(divisor, roundUp);

187:        // Make sure the result is less than 2**256.
188:        // Also prevents divisor == 0
189:        if (divisor <= product1) revert MulDivOverflow(multiplicand, multiplier, divisor);
```

In the above, as we would revert incase `divisor <= product1` there is no need to calculate `result` if we end up reverting as it would just be a waste of some gas. revert early and cheaply as shown below.

```diff
diff --git a/packages/v2-library/src/FullMath.sol b/packages/v2-library/src/FullMath.sol
index a0fba43..02e4883 100644
--- a/packages/v2-library/src/FullMath.sol
+++ b/packages/v2-library/src/FullMath.sol
@@ -181,12 +181,14 @@ library FullMath {
     function mulDiv(uint256 multiplicand, uint256 multiplier, uint256 divisor, bool roundUp) internal pure returns (uint256 result) {
         (uint256 product0, uint256 product1) = mul512(multiplicand, multiplier);

-        // Handle non-overflow cases, 256 by 256 division
-        if (product1 == 0) return result = product0.div(divisor, roundUp);
-
         // Make sure the result is less than 2**256.
         // Also prevents divisor == 0
         if (divisor <= product1) revert MulDivOverflow(multiplicand, multiplier, divisor);
+
+        // Handle non-overflow cases, 256 by 256 division
+        if (product1 == 0) return result = product0.div(divisor, roundUp);
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-library/src/CatchError.sol#L12-L20

```solidity
File: /packages/v2-library/src/CatchError.sol
12:    function catchError(bytes memory reason, bytes4 selector) internal pure returns (bytes memory) {
13:        uint256 length = reason.length;

15:        if ((length - 4) % 32 == 0 && bytes4(reason) == selector) return BytesLib.slice(reason, 4, length - 4);
```

Use the rules of short circuiting to your advantage. In (x && y) , if x is true, y won't be evaluated thus we should always start with the least consuming gas operation

```diff
diff --git a/packages/v2-library/src/CatchError.sol b/packages/v2-library/src/CatchError.sol
index 35d86fc..1b3e123 100644
--- a/packages/v2-library/src/CatchError.sol
+++ b/packages/v2-library/src/CatchError.sol
@@ -12,7 +12,7 @@ library CatchError {
     function catchError(bytes memory reason, bytes4 selector) internal pure returns (bytes memory) {
         uint256 length = reason.length;

-        if ((length - 4) % 32 == 0 && bytes4(reason) == selector) return BytesLib.slice(reason, 4, length - 4);
+        if (bytes4(reason) == selector && (length - 4) % 32 == 0) return BytesLib.slice(reason, 4, length - 4);

         assembly {
             revert(reason, length)
```

## Avoid contract existence checks by using solidity version 0.8.10 or later

Prior to 0.8.10 the compiler inserted extra code, including EXTCODESIZE (100 gas), to check for contract existence for external calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/libraries/PoolFactory.sol#L32

```solidity
File: /packages/v2-pool/src/libraries/PoolFactory.sol

//@audit: get(optionPair)
32:        poolPair = ITimeswapV2PoolFactory(poolFactory).get(optionPair);

//@audit: get(optionPair)
46:        poolPair = ITimeswapV2PoolFactory(poolFactory).get(optionPair);
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/TimeswapV2LiquidityToken.sol#L66

```solidity
File: /packages/v2-token/src/TimeswapV2LiquidityToken.sol

//@audit: toKey()
66:        amount = balanceOf(owner, _timeswapV2LiquidityTokenPositionIds[timeswapV2LiquidityTokenPosition.toKey()]);

//@audit: toKey()
71:        safeTransferFrom(from, to, _timeswapV2LiquidityTokenPositionIds[timeswapV2LiquidityTokenPosition.toKey()], liquidityAmount, bytes(""));

//@audit: toKey()
81:        uint256 id = _timeswapV2LiquidityTokenPositionIds[position.toKey()];

//@audit: toKey()
109:        bytes32 key = timeswapV2LiquidityTokenPosition.toKey();

//@audit: liquidityOf(
125:        uint160 liquidityBalanceTarget = ITimeswapV2Pool(poolPair).liquidityOf(param.strike, param.maturity, address(this)) + param.liquidityAmount;

//@audit: timeswapV2LiquidityTokenMintCallback(
131:        data = ITimeswapV2LiquidityTokenMintCallback(msg.sender).timeswapV2LiquidityTokenMintCallback(

//@audit: toKey()
153:         bytes32 key = TimeswapV2LiquidityTokenPosition({token0: param.token0, token1: param.token1, strike: param.strike, maturity: param.maturity}).toKey();

//@audit: timeswapV2LiquidityTokenBurnCallback(
163:            data = ITimeswapV2LiquidityTokenBurnCallback(msg.sender).timeswapV2LiquidityTokenBurnCallback(

//@audit: toKey()
185:        bytes32 key = TimeswapV2LiquidityTokenPosition({token0: param.token0, token1: param.token1, strike: param.strike, maturity: param.maturity}).toKey();

//@audit: timeswapV2LiquidityTokenCollectCallback(
202:            data = ITimeswapV2LiquidityTokenCollectCallback(msg.sender).timeswapV2LiquidityTokenCollectCallback(

//@audit: feeGrowth(
246:    (long0FeeGrowth, long1FeeGrowth, shortFeeGrowth) = ITimeswapV2Pool(poolPair).feeGrowth(timeswapV2LiquidityTokenPosition.strike, timeswapV2LiquidityTokenPosition.maturity);

//@audit: feeGrowth(
272:             (long0FeeGrowth, long1FeeGrowth, shortFeeGrowth) = ITimeswapV2Pool(poolPair).feeGrowth(timeswapV2LiquidityTokenPosition.strike, timeswapV2LiquidityTokenPosition.maturity);

//@audit: feesEarnedOf(
277:         (uint256 long0Fees, uint256 long1Fees, uint256 shortFees) = feesPosition.feesEarnedOf(uint160(balanceOf(owner, id)), long0FeeGrowth, long1FeeGrowth, shortFeeGrowth);
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L281

```solidity
File: /packages/v2-pool/src/structs/Pool.sol

//@audit: collectTransactionFees(long0Requested, long1Requested, shortRequested)
281:        (long0Amount, long1Amount, shortAmount) = liquidityPosition.collectTransactionFees(long0Requested, long1Requested, shortRequested);

//@audit: timeswapV2PoolMintChoiceCallback(
349:        (long0Amount, long1Amount, data) = ITimeswapV2PoolMintCallback(msg.sender).timeswapV2PoolMintChoiceCallback(

//@audit: timeswapV2PoolBurnChoiceCallback(
438:        (long0Amount, long1Amount, data) = ITimeswapV2PoolBurnCallback(msg.sender).timeswapV2PoolBurnChoiceCallback(

//@audit: timeswapV2PoolDeleverageChoiceCallback(
532:        (long0Amount, long1Amount, data) = ITimeswapV2PoolDeleverageCallback(msg.sender).timeswapV2PoolDeleverageChoiceCallback(

//@audit: timeswapV2PoolLeverageChoiceCallback(
615:            (long0Amount, long1Amount, data) = ITimeswapV2PoolLeverageCallback(msg.sender).timeswapV2PoolLeverageChoiceCallback(
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L78

```solidity
File: /packages/v2-pool/src/TimeswapV2Pool.sol

//@audit: parameter()
78:        (poolFactory, optionPair, transactionFee, protocolFee) = ITimeswapV2PoolDeployer(msg.sender).parameter();

267:        if (long0Amount != 0) long0BalanceTarget = ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Long0) + long0Amount;
//@audit: positionOf()
271:        if (long1Amount != 0) long1BalanceTarget = ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Long1) + long1Amount;

//@audit: positionOf()
274:        uint256 shortBalanceTarget = ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Short) + shortAmount;

//@audit: timeswapV2PoolMintCallback(
277:        data = ITimeswapV2PoolMintCallback(msg.sender).timeswapV2PoolMintCallback(

//@audit: //@audit: positionOf()
293:        if (long0Amount != 0) Error.checkEnough(ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Long0), long0BalanceTarget);

//@audit: //@audit: positionOf()
295:        if (long1Amount != 0) Error.checkEnough(ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Long1), long1BalanceTarget);

//@audit: //@audit: positionOf()
297:     Error.checkEnough(ITimeswapV2Option(optionPair).positionOf(param.strike, param.maturity, address(this), TimeswapV2OptionPosition.Short), shortBalanceTarget);
```

## x += y costs more gas than x = x + y for state variables

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L117

```solidity
File: /packages/v2-token/src/base/ERC1155Enumerable.sol
117:        _currentIndex[to] += 1;
```

```diff
diff --git a/packages/v2-token/src/base/ERC1155Enumerable.sol b/packages/v2-token/src/base/ERC1155Enumerable.sol
index 4ec23ff..d31ce05 100644
--- a/packages/v2-token/src/base/ERC1155Enumerable.sol
+++ b/packages/v2-token/src/base/ERC1155Enumerable.sol
@@ -114,7 +114,7 @@ abstract contract ERC1155Enumerable is IERC1155Enumerable, ERC1155 {
     /// @param to address representing the new owner of the given token ID
     /// @param tokenId uint256 ID of the token to be added to the tokens list of the given address
     function _addTokenToOwnerEnumeration(address to, uint256 tokenId) private {
-        _currentIndex[to] += 1;
+        _currentIndex[to] = _currentIndex[to] + 1;
         uint256 length = _currentIndex[to];
         _ownedTokens[to][length] = tokenId;
         _ownedTokensIndex[tokenId] = length;
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/LiquidityPosition.sol#L55-L57

```solidity
File: /packages/v2-pool/src/structs/LiquidityPosition.sol
55:            liquidityPosition.long0Fees += FeeCalculation.getFees(liquidity, liquidityPosition.long0FeeGrowth, long0FeeGrowth);
56:            liquidityPosition.long1Fees += FeeCalculation.getFees(liquidity, liquidityPosition.long1FeeGrowth, long1FeeGrowth);
57:            liquidityPosition.shortFees += FeeCalculation.getFees(liquidity, liquidityPosition.shortFeeGrowth, shortFeeGrowth);


66:        liquidityPosition.liquidity += liquidityAmount;

70:        liquidityPosition.long0Fees += long0Fees;
71:        liquidityPosition.long1Fees += long1Fees;
72:        liquidityPosition.shortFees += shortFees;

76:        liquidityPosition.liquidity -= liquidityAmount;

80:        liquidityPosition.long0Fees -= long0Fees;
81:        liquidityPosition.long1Fees -= long1Fees;
82:        liquidityPosition.shortFees -= shortFees;
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L361-L362

```solidity
File: /packages/v2-pool/src/structs/Pool.sol
361:        if (long0Amount != 0) pool.long0Balance += long0Amount;
362:        if (long1Amount != 0) pool.long1Balance += long1Amount;

365:        pool.liquidity += liquidityAmount;

452:        if (long0Amount != 0) pool.long0Balance -= long0Amount;
453:        if (long1Amount != 0) pool.long1Balance -= long1Amount;

455:        pool.liquidity -= liquidityAmount;

537:        if (long0Amount != 0) pool.long0Balance += long0Amount;
538:        if (long1Amount != 0) pool.long1Balance += long1Amount;

636:        pool.long0Balance -= (long0Amount + long0Fees);

650:        pool.long1Balance -= (long1Amount + long1Fees);

677:        pool.long1Balance -= (long1Amount + longFees);

689:        pool.long1Balance -= (long1Amount + longFees);

695:        pool.long0Balance += long0Amount;

710:        pool.long0Balance -= (long0Amount + longFees);

719:        pool.long0Balance -= (long0Amount + longFees);

722:        pool.long1Balance += long1Amount;
```

## Usage of uints/ints smaller than 32 bytes (256 bits) incurs overhead

When using elements that are smaller than 32 bytes, your contract’s gas usage may be higher. This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller than that, the EVM must use more operations in order to reduce the size of the element from 32 bytes to the desired size.

https://docs.soliditylang.org/en/v0.8.11/internals/layout_in_storage.html Use a larger size then downcast where needed

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L82

```solidity
File: /packages/v2-pool/src/TimeswapV2Pool.sol

//@audit: uint96 durationForward
82:    function blockTimestamp(uint96 durationForward) internal view virtual returns (uint96) {

//@audit: uint160 liquidityAmount
152:    function transferLiquidity(uint256 strike, uint256 maturity, address to, uint160 liquidityAmount) external override {

//@audit: uint96 durationForward
252:    function mint(
253:        TimeswapV2PoolMintParam calldata param,
254:        bool isQuote,
255:        uint96 durationForward
256:    ) private noDelegateCall returns (uint160 liquidityAmount, uint256 long0Amount, uint256 long1Amount, uint256 shortAmount, bytes memory data) {

//@audit: uint96 durationForward
320:        uint96 durationForward

//@audit: uint96 durationForward
380:        uint96 durationForward

```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/LiquidityPosition.sol#L65

```solidity
File: /packages/v2-pool/src/structs/LiquidityPosition.sol

//@audit: uint160 liquidityAmount
65:    function mint(LiquidityPosition storage liquidityPosition, uint160 liquidityAmount) internal {

//@audit: uint160 liquidityAmount
75:    function burn(LiquidityPosition storage liquidityPosition, uint160 liquidityAmount) internal {
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Param.sol#L142

```solidity
File: /packages/v2-pool/src/structs/Param.sol

//@audit:uint96 blockTimestamp
142:    function check(TimeswapV2PoolMintParam memory param, uint96 blockTimestamp) internal pure {

//@audit:uint96 blockTimestamp
153:    function check(TimeswapV2PoolBurnParam memory param, uint96 blockTimestamp) internal pure {

//@audit:uint96 blockTimestamp
165:    function check(TimeswapV2PoolDeleverageParam memory param, uint96 blockTimestamp) internal pure {

//@audit:uint96 blockTimestamp
176:    function check(TimeswapV2PoolLeverageParam memory param, uint96 blockTimestamp) internal pure {

//@audit:uint96 blockTimestamp
188:    function check(TimeswapV2PoolRebalanceParam memory param, uint96 blockTimestamp) internal pure {
```

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L101

```solidity
File: /packages/v2-pool/src/structs/Pool.sol

//@audit: uint96 blockTimestamp
101:    function totalPositions(Pool storage pool, uint256 maturity, uint96 blockTimestamp) external view returns (uint256 longAmount, uint256 shortAmount) {

//@audit: uint160 liquidity,uint160 rate, uint96 duration,uint96 blockTimestamp
114:    function updateDurationWeight(
115:        uint160 liquidity,
116:        uint160 rate,
117:        uint256 shortFeeGrowth,
118:        uint96 duration,
119:        uint96 blockTimestamp
120:    ) private pure returns (uint96 newLastTimestamp, uint256 newShortFeeGrowth) {

//@audit:uint96 blockTimestamp
128:    function updateDurationWeightBeforeMaturity(Pool storage pool, uint96 blockTimestamp) private {

//@audit:uint96 blockTimestamp
142:    function updateDurationWeightAfterMaturity(Pool storage pool, uint256 maturity, uint96 blockTimestamp) private {

//@audit: uint160 liquidityAmount, uint96 blockTimestamp
158:    function transferLiquidity(Pool storage pool, address to, uint160 liquidityAmount, uint96 blockTimestamp) external {

//@audit:uint96 blockTimestamp
183:    function transferFees(Pool storage pool, uint256 maturity, address to, uint256 long0Fees, uint256 long1Fees, uint256 shortFees, uint96 blockTimestamp) external {

//@audit:uint160 rate
205:    function initialize(Pool storage pool, uint160 rate) external {

//@audit:  uint96 blockTimestamp
262:    function collectTransactionFees(
263:        Pool storage pool,
264:        uint256 maturity,
265:        uint256 long0Requested,
266:        uint256 long1Requested,
267:        uint256 shortRequested,
268:        uint96 blockTimestamp
269:    ) external returns (uint256 long0Amount, uint256 long1Amount, uint256 shortAmount) {

//@audit: uint96 blockTimestamp
294:    function mint(
295:        Pool storage pool,
296:        TimeswapV2PoolMintParam memory param,
297:        uint96 blockTimestamp
298:    ) external returns (uint160 liquidityAmount, uint256 long0Amount, uint256 long1Amount, uint256 shortAmount, bytes memory data) {

//@audit: uint96 blockTimestamp
380:    function burn(
381:        Pool storage pool,
382:        TimeswapV2PoolBurnParam memory param,
383:        uint96 blockTimestamp
384:    ) external returns (uint160 liquidityAmount, uint256 long0Amount, uint256 long1Amount, uint256 shortAmount, bytes memory data) {

//@audit: uint96 blockTimestamp
469:    function deleverage(
470:        Pool storage pool,
471:        TimeswapV2PoolDeleverageParam memory param,
472:        uint256 transactionFee,
473:        uint256 protocolFee,
474:        uint96 blockTimestamp
475:    ) external returns (uint256 long0Amount, uint256 long1Amount, uint256 shortAmount, bytes memory data) {
```

## Use a more recent version of solidity

Use a solidity version of at least 0.8.2 to get simple compiler automatic inlining Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than revert()/require() strings Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-option/src/structs/Process.sol#L2

```solidity
File: /packages/v2-option/src/structs/Process.sol
2: pragma solidity =0.8.8;
```

Majority of the codebase is using the above. Consider upgrading to the latst
