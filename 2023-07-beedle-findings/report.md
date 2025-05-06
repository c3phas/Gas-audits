## Findings

https://github.com/Cyfrin/2023-07-beedle/issues/1699

## Auditor's Disclaimer

While we try our best to maintain readability in the provided code snippets, some functions have been truncated to highlight the affected portions.

It's important to note that during the implementation of these suggested changes, developers must exercise caution to avoid introducing vulnerabilities. Although the optimizations have been tested prior to these recommendations, it is the responsibility of the developers to conduct thorough testing again.

Code reviews and additional testing are strongly advised to minimize any potential risks associated with the refactoring process.

## Codebase impressions

The developers have done an excellent job of identifying and implementing some of the most evident optimizations like use of Immutables. The code is also well commented out which makes it easy for one to follow along.
However, we have identified additional areas where optimization is possible, some of which may not be immediately apparent.

## Note on Gas estimates.

There seems to be an issue with how the command `forge test --gas-report --optimize` gives results. It seems to be giving varying results for the same tests cases. eg For some functions one run might give an average gas of `2000` while running it for the second time gives `1500` . For this reason giving the exact gas savings from the tests included wasn't feasible.
However for the slot packings the gas can be calculated based on how many slots are being saved.

## Tighly pack storage variables/optimize the order of variable declaration(Save 2 SLOTS: 4200 Gas)

The EVM works with 32 byte words. Variables less than 32 bytes can be declared next to each other in storage and this will pack the values together into a single 32 byte storage slot (if the values combined are <= 32 bytes). If the variables packed together are retrieved together in functions we will effectively save ~2000 gas with every subsequent SLOAD for that storage slot. This is due to us incurring a `Gwarmaccess (100 gas)` versus a `Gcoldsload (2100 gas)`.

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L63

```solidity
File: /src/Lender.sol
63:    uint256 public lenderFee = 1000;

65:    uint256 public borrowerFee = 50;

67:    address public feeReceiver;
```

Reducing the size of `lenderFee` should be safe given how values are assigned to it. see below
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L84-L87

```solidity
File: /src/Lender.sol
84:    function setLenderFee(uint256 _fee) external onlyOwner {
85:        if (_fee > 5000) revert FeeTooHigh();
86:        lenderFee = _fee;
87:    }
```

When setting `lenderFee` we have a conditional check that ensures the value assigned to our state variable is always less than 5000. This means any size from `uint16` should be big enough to hold our value

Similar thing with `borrowerFee` where constraint the value to less than `500`

```solidity
        if (_fee > 500) revert FeeTooHigh();
        borrowerFee = _fee;
```

As the three state variables are being accessed together, packing would save us a lot of gas

```diff
diff --git a/src/Lender.sol b/src/Lender.sol
index d04d14f..a5473c6 100644
--- a/src/Lender.sol
+++ b/src/Lender.sol
@@ -60,9 +60,9 @@ contract Lender is Ownable {
     /// @notice the maximum auction length is 3 days
     uint256 public constant MAX_AUCTION_LENGTH = 3 days;
     /// @notice the fee taken by the protocol in BIPs
-    uint256 public lenderFee = 1000;
+    uint64 public lenderFee = 1000;
     /// @notice the fee taken by the protocol in BIPs
-    uint256 public borrowerFee = 50;
+    uint64 public borrowerFee = 50;
     /// @notice the address of the fee receiver
     address public feeReceiver;

```

## Pack structs by putting data types that can fit together next to each other

As the solidity EVM works with 32 bytes, variables less than 32 bytes should be packed inside a struct so that they can be stored in the same slot, this saves gas when writing to storage ~20000 gas

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/utils/Structs.sol#L4-L23

### auctionLength and interestRate can be packed with an address(We can save 2 SLOTS: 4000 Gas)

```solidity
File: /src/utils/Structs.sol
4: struct Pool {
5:    /// @notice address of the lender
6:    address lender;
7:    /// @notice address of the loan token
8:    address loanToken;
9:    /// @notice address of the collateral token
10:    address collateralToken;
11:    /// @notice the minimum size of the loan (to prevent griefing)
12:    uint256 minLoanSize;
13:    /// @notice the maximum size of the loan (also equal to the balance of the lender)
14:    uint256 poolBalance;
15:    /// @notice the max ratio of loanToken/collateralToken (multiplied by 10**18)
16:    uint256 maxLoanRatio;
17:    /// @notice the length of a refinance auction
18:    uint256 auctionLength;
19:    /// @notice the interest rate per year in BIPs
20:    uint256 interestRate;
21:    /// @notice the outstanding loans this pool has
22:    uint256 outstandingLoans;
23: }
```

When setting up the pool, the values of `auctionLength` and `interestRate` are being constrained as shown below

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L132-L139

```solidity
File: /src/Lender.sol
132:        if (
133:            p.lender != msg.sender ||
134:            p.minLoanSize == 0 ||
135:            p.maxLoanRatio == 0 ||
136:            p.auctionLength == 0 ||
137:            p.auctionLength > MAX_AUCTION_LENGTH ||
138:            p.interestRate > MAX_INTEREST_RATE
139:        ) revert PoolConfig();
```

Note, `p.auctionLength and p.interestRate ` cannot be greater than `MAX_AUCTION_LENGTH` and `MAX_INTEREST_RATE` respectively as this would trigger a revert

For `MAX_AUCTION_LENGTH` , it's a constant with a value of `3 days` while `MAX_INTEREST_RATE` has a value of `100000` . See https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L59C49-L61

The biggest value here is `100000` , `uint64` should be big enough

```diff
diff --git a/src/utils/Structs.sol b/src/utils/Structs.sol
index 42d6fb6..1df1d60 100644
--- a/src/utils/Structs.sol
+++ b/src/utils/Structs.sol
@@ -2,6 +2,10 @@
 pragma solidity ^0.8.19;

 struct Pool {
+    /// @notice the length of a refinance auction
+    uint64 auctionLength;
+    /// @notice the interest rate per year in BIPs
+    uint64 interestRate;
     /// @notice address of the lender
     address lender;
     /// @notice address of the loan token
@@ -14,10 +18,6 @@ struct Pool {
     uint256 poolBalance;
     /// @notice the max ratio of loanToken/collateralToken (multiplied by 10**18)
     uint256 maxLoanRatio;
-    /// @notice the length of a refinance auction
-    uint256 auctionLength;
-    /// @notice the interest rate per year in BIPs
-    uint256 interestRate;
     /// @notice the outstanding loans this pool has
     uint256 outstandingLoans;
 }
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/utils/Structs.sol#L34-L55

### Due to how values are being constrained when assigning, we can save 2 SLOTS here

`1 SLOT = 2k gas`
**`Total Gas Saved: 4K Gas`**
**We pack `interestRate,startTimestamp and auctionLength` in one slot** We can actually pack more though. See explanation below.

```solidity
File: /src/utils/Structs.sol
34: struct Loan {
35:    /// @notice address of the lender
36:    address lender;
37:    /// @notice address of the borrower
38:    address borrower;
39:    /// @notice address of the loan token
40:    address loanToken;
41:    /// @notice address of the collateral token
42:    address collateralToken;
43:    /// @notice the amount borrowed
44:    uint256 debt;
45:    /// @notice the amount of collateral locked in the loan
46:    uint256 collateral;
47:    /// @notice the interest rate of the loan per second (in debt tokens)
48:    uint256 interestRate;
49:    /// @notice the timestamp of the loan start
50:    uint256 startTimestamp;
51:    /// @notice the timestamp of a refinance auction start
52:    uint256 auctionStartTimestamp;
53:    /// @notice the refinance auction length
54:    uint256 auctionLength;
55: }
```

Before we assign our struct values , we have some validations being done for the values that can be assigned
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L373-L375

```solidity
File: /src/Lender.sol
373:            if (pool.interestRate > loan.interestRate) revert RateTooHigh();
374:            // auction length cannot be shorter than old auction length
375:            if (pool.auctionLength < loan.auctionLength) revert AuctionTooShort();
```

The above means two things, `loan.interestRate and loan.auctionLength` would always be less or equal to `pool.interestRate and pool.auctionLength` respectively.
**This constraint is being enforced whenever we write new values to this struct.**

The struct above is being set when we give out a loan https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L355-L432
Of interest to us is the following piece of code

```solidity
File: /src/Lender.sol
415:            // update the loan with the new info
416:            loans[loanId].lender = pool.lender;
417:            loans[loanId].interestRate = pool.interestRate;
418:            loans[loanId].startTimestamp = block.timestamp;
419:            loans[loanId].auctionStartTimestamp = type(uint256).max;
420:            loans[loanId].debt = totalDebt;
```

Note the value being assigned to `loans[loanId].interestRate` is the value` pool.interestRate` which comes from a previous struct `POOL`

The values `pool.interestRate and pool.auctionLength` have some max enforced. See previous `struct packing finding for more explanation`
'Note, `p.auctionLength and p.interestRate ` cannot be greater than `MAX_AUCTION_LENGTH` and `MAX_INTEREST_RATE` respectively as this would trigger a revert'
This means the variables from the `struct Loan` need not to be bigger than those in `struct Pool`.
We can reduce the size to `uint64` and still be safe.

**For Timestamps `uint64` should also be big enough:**
**For `auctionStartTimestamp` the value seems to be hard-coded as `type(uint256).max` so I avoided changing it. If not really required we can switch to uint64 and save 1 more SLOT**

```diff
 struct Loan {
+    /// @notice the timestamp of the loan start
+    uint64 startTimestamp;
+    /// @notice the interest rate of the loan per second (in debt tokens)
+    uint64 interestRate;
+    /// @notice the refinance auction length
+    uint64 auctionLength;
     /// @notice address of the lender
     address lender;
     /// @notice address of the borrower
@@ -42,16 +48,11 @@ struct Loan {
     address collateralToken;
     /// @notice the amount borrowed
     uint256 debt;
+        /// @notice the timestamp of a refinance auction start
+    uint256 auctionStartTimestamp;
     /// @notice the amount of collateral locked in the loan
     uint256 collateral;
-    /// @notice the interest rate of the loan per second (in debt tokens)
-    uint256 interestRate;
-    /// @notice the timestamp of the loan start
-    uint256 startTimestamp;
-    /// @notice the timestamp of a refinance auction start
-    uint256 auctionStartTimestamp;
-    /// @notice the refinance auction length
-    uint256 auctionLength;
+
 }
```

Refactoring the above means we need to do some other changes on the other parts of code that were referencing this variables.

## Expensive operations inside a for loop

### Reading state variables in a loop might be too expensive if the array length is big

\*\*We also emit an event on every loop
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L233-L286

```solidity
File: /src/Lender.sol
        for (uint256 i = 0; i < borrows.length; i++) {

            // calculate the fees
            uint256 fees = (debt * borrowerFee) / 10000;
            // transfer fees
            IERC20(loan.loanToken).transfer(feeReceiver, fees);
            // transfer the loan tokens from the pool to the borrower
            IERC20(loan.loanToken).transfer(msg.sender, debt - fees);
            // transfer the collateral tokens from the borrower to the contract
            IERC20(loan.collateralToken).transferFrom(
                msg.sender,
                address(this),
                collateral
            );
            loans.push(loan);
            emit Borrowed(
                msg.sender,
                pool.lender,
                loans.length - 1,
                debt,
                collateral,
                pool.interestRate,
                block.timestamp
            );
        }
```

The above loop reads `borrowerFee` and `feeReceiver` repeatedly which is too expensive.

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L325

### Cache `feeReceiver` outside the loop

```solidity
File: /src/Lender.sol
325:                feeReceiver,

```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L403

### Cache `feeReceiver` outside the loop

```solidity
File: /src/Lender.sol
403:            IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);
```

## Multiple address mappings can be combined into a single mapping of an address to a struct, where appropriate

Saves a storage slot for the mapping. Depending on the circumstances and sizes of types, can avoid a Gsset (20000 gas) per mapping combined. Reads and subsequent writes can also be cheaper when a function requires both values and they both fit in the same storage slot. Finally, if both fields are accessed in the same function, can save ~42 gas per access due to not having to recalculate the key's keccak256 hash (Gkeccak256 - 30 gas) and that calculation's associated stack operations.

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L18-L24

```solidity
File: /src/Staking.sol
18:    /// @notice mapping of user indexes
19:    mapping(address => uint256) public supplyIndex;

21:    /// @notice mapping of user balances
22:    mapping(address => uint256) public balances;
23:    /// @notice mapping of user claimable rewards
24:    mapping(address => uint256) public claimable;
```

```diff
-
-    /// @notice mapping of user indexes
-    mapping(address => uint256) public supplyIndex;

-    /// @notice mapping of user balances
-    mapping(address => uint256) public balances;
-    /// @notice mapping of user claimable rewards
-    mapping(address => uint256) public claimable;
+    struct User{
+        uint256 supplyIndex;
+        uint256 balances;
+        uint256 claimable;
+    }
+    mapping (address => User) usersInfo;

```

```diff
-        balances[msg.sender] += _amount;
+        usersInfo[msg.sender].balances += _amount;
```

To help the optimizer further, we declare a storage type variable and use it instead of repeatedly fetching the reference in a map or an array.

```diff
     function updateFor(address recipient) public {
         update();
-        uint256 _supplied = balances[recipient];
+        User storage _usersInfo = usersInfo[recipient];
+        uint256 _supplied = _usersInfo.balances;
         if (_supplied > 0) {
-            uint256 _supplyIndex = supplyIndex[recipient];
-            supplyIndex[recipient] = index;
+            uint256 _supplyIndex = _usersInfo.supplyIndex;
+            _usersInfo.supplyIndex = index;
             uint256 _delta = index - _supplyIndex;
             if (_delta > 0) {
               uint256 _share = _supplied * _delta / 1e18;
-              claimable[recipient] += _share;
+              _usersInfo.claimable += _share;
             }
         } else {
-            supplyIndex[recipient] = index;
+            _usersInfo.supplyIndex = index;
         }
     }
```

## Using storage instead of memory for structs/arrays saves gas

When fetching data from a storage location, assigning the data to a memory variable causes all fields of the struct/array to be read from storage, which incurs a Gcoldsload (2100 gas) for each field of the struct/array. If the fields are read from the new memory variable, they incur an additional MLOAD rather than a cheap stack read. Instead of declearing the variable with the memory keyword, declaring the variable with the storage keyword and caching any fields that need to be re-read in stack variables, will be much cheaper, only incuring the Gcoldsload for the fields actually read. The only time it makes sense to read the whole struct/array into a memory variable, is if the full struct/array is being returned by the function, is being passed to a function that requires memory, or if the array/struct is being read from another memory array/struct

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L116-L121

```solidity
File: /src/Lender.sol
116:    function getLoanDebt(uint256 loanId) external view returns (uint256 debt) {
117:        Loan memory loan = loans[loanId];
118:        // calculate the accrued interest
119:        (uint256 interest, uint256 fees) = _calculateInterest(loan);
120:        debt = loan.debt + interest + fees;
121:    }
```

```diff
     function getLoanDebt(uint256 loanId) external view returns (uint256 debt) {
-        Loan memory loan = loans[loanId];
+        Loan storage loan = loans[loanId];
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L467

```solidity
File: /src/Lender.sol
467:        Loan memory loan = loans[loanId];
```

## Multiple accesses of a mapping/array should use a local variable cache

Caching a mapping's value in a local storage or calldata variable when the value is accessed multiple times saves ~42 gas per access due to not having to perform the same offset calculation every time.
**Help the Optimizer by saving a storage variable's reference instead of repeatedly fetching it**

To help the optimizer,declare a storage type variable and use it instead of repeatedly fetching the reference in a map or an array.
As an example, instead of repeatedly calling `someMap[someIndex]`, save its reference like this: `SomeStruct storage someStruct = someMap[someIndex]` and use it.

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L130-L176

### Lender.sol.setPool(): Cache `pools[poolId]` in local storage

**The gas benchmarks cannot be relied on as the included tests seems to give different values on every run**

|        | Min    | Average | Median | Max    |
| ------ | ------ | ------- | ------ | ------ |
| Before | 201208 | 219563  | 225608 | 225608 |
| After  | 9324   | 212134  | 225478 | 225478 |

```solidity
File: /src/Lender.sol
130:    function setPool(Pool calldata p) public returns (bytes32 poolId) {

144:        // you can't change the outstanding loans
145:        if (p.outstandingLoans != pools[poolId].outstandingLoans)
146:            revert PoolConfig();

148:        uint256 currentBalance = pools[poolId].poolBalance;


167:        if (pools[poolId].lender == address(0)) {

175:        pools[poolId] = p;
176:    }
```

```diff
diff --git a/src/Lender.sol b/src/Lender.sol
index d04d14f..d384b39 100644
--- a/src/Lender.sol
+++ b/src/Lender.sol
@@ -142,10 +142,11 @@ contract Lender is Ownable {
         poolId = getPoolId(p.lender, p.loanToken, p.collateralToken);

         // you can't change the outstanding loans
-        if (p.outstandingLoans != pools[poolId].outstandingLoans)
+        Pool storage _pool = pools[poolId];
+        if (p.outstandingLoans != _pool.outstandingLoans)
             revert PoolConfig();

-        uint256 currentBalance = pools[poolId].poolBalance;
+        uint256 currentBalance = _pool.poolBalance;

         if (p.poolBalance > currentBalance) {
             // if new balance > current balance then transfer the difference from the lender
@@ -164,7 +165,7 @@ contract Lender is Ownable {

         emit PoolBalanceUpdated(poolId, p.poolBalance);

-        if (pools[poolId].lender == address(0)) {
+        if (_pool.lender == address(0)) {
             // if the pool doesn't exist then create it
             emit PoolCreated(poolId, p);
         } else {
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182-L192

```solidity
File: /src/Lender.sol
182:    function addToPool(bytes32 poolId, uint256 amount) external {
183:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
184:        if (amount == 0) revert PoolConfig();
185:        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
186:        // transfer the loan tokens from the lender to the contract
187:        IERC20(pools[poolId].loanToken).transferFrom(
188:            msg.sender,
189:            address(this),
190:            amount
191:        );
192:    }
```

```diff
     function addToPool(bytes32 poolId, uint256 amount) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
+        Pool storage _pools = pools[poolId];
+        if (_pools.lender != msg.sender) revert Unauthorized();
         if (amount == 0) revert PoolConfig();
-        _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
+        _updatePoolBalance(poolId, _pools.poolBalance + amount);
         // transfer the loan tokens from the lender to the contract
-        IERC20(pools[poolId].loanToken).transferFrom(
+        IERC20(_pools.loanToken).transferFrom(
             msg.sender,
             address(this),
             amount
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L198-L204

```solidity
File: /src/Lender.sol
198:    function removeFromPool(bytes32 poolId, uint256 amount) external {
199:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
200:        if (amount == 0) revert PoolConfig();
201:        _updatePoolBalance(poolId, pools[poolId].poolBalance - amount);
202:        // transfer the loan tokens from the contract to the lender
203:        IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);
204:    }
```

```diff
     function removeFromPool(bytes32 poolId, uint256 amount) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
+        Pool storage _pools = pools[poolId];
+        if (_pools.lender != msg.sender) revert Unauthorized();
         if (amount == 0) revert PoolConfig();
-        _updatePoolBalance(poolId, pools[poolId].poolBalance - amount);
+        _updatePoolBalance(poolId, _pools.poolBalance - amount);
         // transfer the loan tokens from the contract to the lender
-        IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);
+        IERC20(_pools.loanToken).transfer(msg.sender, amount);
     }
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L210-L215

```solidity
File: /src/Lender.sol
210:    function updateMaxLoanRatio(bytes32 poolId, uint256 maxLoanRatio) external {
211:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
212:        if (maxLoanRatio == 0) revert PoolConfig();
213:        pools[poolId].maxLoanRatio = maxLoanRatio;
214:        emit PoolMaxLoanRatioUpdated(poolId, maxLoanRatio);
215:    }
```

```diff
     function updateMaxLoanRatio(bytes32 poolId, uint256 maxLoanRatio) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
+        Pool storage _pools = pools[poolId];
+        if (_pools.lender != msg.sender) revert Unauthorized();
         if (maxLoanRatio == 0) revert PoolConfig();
-        pools[poolId].maxLoanRatio = maxLoanRatio;
+        _pools.maxLoanRatio = maxLoanRatio;
         emit PoolMaxLoanRatioUpdated(poolId, maxLoanRatio);
     }
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221-L226

```solidity
File: /src/Lender.sol
221:    function updateInterestRate(bytes32 poolId, uint256 interestRate) external {
222:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
223:        if (interestRate > MAX_INTEREST_RATE) revert PoolConfig();
224:        pools[poolId].interestRate = interestRate;
225:        emit PoolInterestRateUpdated(poolId, interestRate);
226:    }
```

```diff
     function updateInterestRate(bytes32 poolId, uint256 interestRate) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
+        Pool storage _pools = pools[poolId];
+        if (_pools.lender != msg.sender) revert Unauthorized();
         if (interestRate > MAX_INTEREST_RATE) revert PoolConfig();
-        pools[poolId].interestRate = interestRate;
+        _pools.interestRate = interestRate;
         emit PoolInterestRateUpdated(poolId, interestRate);
     }
```

## Lender.sol.buyLoan(): We can optimize this function by caching and using storage for some variables

**To avoid stack too deep error, we introduce an internal function**
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L465-L534
**The gas benchmarks cannot be relied on as the included tests seems to give different values on every run**

|        | Min  | Average | Median | Max   |
| ------ | ---- | ------- | ------ | ----- |
| Before | 2107 | 10665   | 2582   | 35392 |
| After  | 997  | 9665    | 1481   | 34712 |

```solidity
File: /src/Lender.sol
    function buyLoan(uint256 loanId, bytes32 poolId) public {
        // get the loan info
        Loan memory loan = loans[loanId];
        // validate the loan
        if (loan.auctionStartTimestamp == type(uint256).max)
            revert AuctionNotStarted();
        if (block.timestamp > loan.auctionStartTimestamp + loan.auctionLength)
            revert AuctionEnded();
        // calculate the current interest rate
        uint256 timeElapsed = block.timestamp - loan.auctionStartTimestamp;
        uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
            loan.auctionLength;
        // validate the rate
        if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh();
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
            loan
        );


        // reject if the pool is not big enough
        uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
        if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();


        // if they do have a big enough pool then transfer from their pool
        _updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
        pools[poolId].outstandingLoans += totalDebt;


        // now update the pool balance of the old lender
        bytes32 oldPoolId = getPoolId(
            loan.lender,
            loan.loanToken,
            loan.collateralToken
        );
        _updatePoolBalance(
            oldPoolId,
            pools[oldPoolId].poolBalance + loan.debt + lenderInterest
        );
        pools[oldPoolId].outstandingLoans -= loan.debt;


        // transfer the protocol fee to the governance
        IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);


        emit Repaid(
            loan.borrower,
            loan.lender,
            loanId,
            loan.debt + lenderInterest + protocolInterest,
            loan.collateral,
            loan.interestRate,
            loan.startTimestamp
        );


        // update the loan with the new info
        loans[loanId].lender = msg.sender;
        loans[loanId].interestRate = pools[poolId].interestRate;
        loans[loanId].startTimestamp = block.timestamp;
        loans[loanId].auctionStartTimestamp = type(uint256).max;
        loans[loanId].debt = totalDebt;


        emit Borrowed(
            loan.borrower,
            msg.sender,
            loanId,
            loans[loanId].debt,
            loans[loanId].collateral,
            pools[poolId].interestRate,
            block.timestamp
        );
        emit LoanBought(loanId);
    }
```

```diff
diff --git a/src/Lender.sol b/src/Lender.sol
index d04d14f..28ee496 100644
--- a/src/Lender.sol
+++ b/src/Lender.sol
@@ -458,36 +458,49 @@ contract Lender is Ownable {
         }
     }

-    /// @notice buy a loan in a refinance auction
+  /*  /// @notice buy a loan in a refinance auction
     /// can be called by anyone but you must have a pool with tokens
     /// @param loanId the id of the loan to refinance
-    /// @param poolId the pool to accept
-    function buyLoan(uint256 loanId, bytes32 poolId) public {
+    /// @param poolId the pool to accept*/
+    function buyLoanHelper(uint loanId) internal returns(uint currentAuctionRate,Loan storage loan){
         // get the loan info
-        Loan memory loan = loans[loanId];
+        loan = loans[loanId];
         // validate the loan
+        uint256 _auctionStartTimestamp = loan.auctionStartTimestamp;
+        if (_auctionStartTimestamp == type(uint256).max)
             revert AuctionNotStarted();
+        uint256 _auctionLength = loan.auctionLength;
+        if (block.timestamp > _auctionStartTimestamp + _auctionLength)
             revert AuctionEnded();
         // calculate the current interest rate
-        uint256 timeElapsed = block.timestamp - loan.auctionStartTimestamp;
-        uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
-            loan.auctionLength;
+        uint256 timeElapsed = block.timestamp - _auctionStartTimestamp;
+        currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
+            _auctionLength;
         // validate the rate
-        if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh();
+        //
+      //  return (currentAuctionRate,loan);
+
+    }
+    function buyLoan(uint256 loanId, bytes32 poolId) public {
+        (uint currentAuctionRate, Loan storage loan) = buyLoanHelper(loanId);
+
+        Pool storage _pools = pools[poolId];
+
+        if (_pools.interestRate > currentAuctionRate) revert RateTooHigh();
         // calculate the interest
         (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
             loan
         );

         // reject if the pool is not big enough
-        uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
-        if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();
+        uint256 _loanDebt = loan.debt;
+        uint256 totalDebt = _loanDebt + lenderInterest + protocolInterest;
+        uint _poolsBalance = _pools.poolBalance;
+        if (_poolsBalance < totalDebt) revert PoolTooSmall();

         // if they do have a big enough pool then transfer from their pool
-        _updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
-        pools[poolId].outstandingLoans += totalDebt;
+        _updatePoolBalance(poolId, _poolsBalance - totalDebt);
+        _pools.outstandingLoans += totalDebt;

         // now update the pool balance of the old lender
         bytes32 oldPoolId = getPoolId(
@@ -495,11 +508,12 @@ contract Lender is Ownable {
             loan.loanToken,
             loan.collateralToken
         );
+        Pool storage _oldPool = pools[oldPoolId];
         _updatePoolBalance(
             oldPoolId,
-            pools[oldPoolId].poolBalance + loan.debt + lenderInterest
+            _oldPool.poolBalance + _loanDebt + lenderInterest
         );
-        pools[oldPoolId].outstandingLoans -= loan.debt;
+        _oldPool.outstandingLoans = _oldPool.outstandingLoans - _loanDebt;

         // transfer the protocol fee to the governance
         IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);
@@ -508,26 +522,27 @@ contract Lender is Ownable {
             loan.borrower,
             loan.lender,
             loanId,
-            loan.debt + lenderInterest + protocolInterest,
+            _loanDebt + lenderInterest + protocolInterest,
             loan.collateral,
             loan.interestRate,
             loan.startTimestamp
         );

         // update the loan with the new info
-        loans[loanId].lender = msg.sender;
-        loans[loanId].interestRate = pools[poolId].interestRate;
-        loans[loanId].startTimestamp = block.timestamp;
-        loans[loanId].auctionStartTimestamp = type(uint256).max;
-        loans[loanId].debt = totalDebt;
+        uint256 _poolsInterestRate = _pools.interestRate;
+        loan.lender = msg.sender;
+        loan.interestRate = _poolsInterestRate;
+        loan.startTimestamp = block.timestamp;
+        loan.auctionStartTimestamp = type(uint256).max;
+        loan.debt = totalDebt;

         emit Borrowed(
             loan.borrower,
             msg.sender,
             loanId,
-            loans[loanId].debt,
-            loans[loanId].collateral,
-            pools[poolId].interestRate,
+            totalDebt,
+            loan.collateral,
+            _poolsInterestRate,
             block.timestamp
         );
         emit LoanBought(loanId);
```

## x += y costs more gas than x = x + y for state variable

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L41

```solidity
File: /src/Staking.sol
41:        balances[msg.sender] += _amount;
```

```diff
     function deposit(uint _amount) external {
         TKN.transferFrom(msg.sender, address(this), _amount);
         updateFor(msg.sender);
-        balances[msg.sender] += _amount;
+        balances[msg.sender] = balances[msg.sender] + _amount;
     }

```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L48

```solidity
File: /src/Staking.sol
48:        balances[msg.sender] -= _amount;
```

```diff
     function withdraw(uint _amount) external {
         updateFor(msg.sender);
-        balances[msg.sender] -= _amount;
+        balances[msg.sender] = balances[msg.sender] - _amount;
         TKN.transfer(msg.sender, _amount);
     }
```

## No need to initialize variables with their default values

If a variable is not set/initialized, it is assumed to have the default value (0, false, 0x0 etc depending on the data type). If you explicitly initialize it with its default value, you increase deployment cost and size

Initializing variables with their default values requires additional bytecode instructions to set those values explicitly during deployment. By excluding these instructions, the overall size of the compiled contract's bytecode can be reduced.

The gas cost difference (remix and foundry) is 2206 gas per variable

We get 3 extra opcodes when we initialize it.

```
PUSH1 00 --> 3 gas
DUP1     --> 3 gas
SSTORE   --> 2200 gas
```

The above explains the `2206 gas` difference.
since we have two variables we would save ~4412 gas in the following

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L14-L16

```solidity
File: /src/Staking.sol
14:    uint256 public balance = 0;
15:    /// @notice the index of the last update
16:    uint256 public index = 0;
```

```diff
 contract Staking is Ownable {

     /// @notice the balance of reward tokens
-    uint256 public balance = 0;
+    uint256 public balance;
     /// @notice the index of the last update
-    uint256 public index = 0;
+    uint256 public index;

```

## Using unchecked blocks to save gas

Solidity version 0.8+ comes with implicit overflow and underflow checks on unsigned integers. When an overflow or an underflow isn’t possible (as an example, when a comparison is made before the arithmetic operation), some gas can be saved by using an unchecked block
[see resource](https://github.com/ethereum/solidity/issues/10695)

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L66

```solidity
File: /src/Staking.sol
66:                uint256 _diff = _balance - balance;
```

The operation `_balance - balance` cannot underflow due to the check on [Line 65](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Staking.sol#L65) that ensures that `_balance` is greater than `balance` for this operation to be performed

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L155

```solidity
File: /src/Lender.sol
155:                p.poolBalance - currentBalance
```

The above operation cannot underflow as it would only be performed if `p.poolBalance` is greater than `currentBalance` due to the condition check ` if (p.poolBalance > currentBalance) {` on [Line 150](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L150)
It should be safe to wrap the whole external call

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L161

```solidity
File: /src/Lender.sol
161:                currentBalance - p.poolBalance
```

This is the else block of the previous check, The code would only be evaluated if `p.poolBalance` is less than `currentBalance` due to the check on [Line 157](https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L157)

## Optimizing check order for cost efficient function execution

Checks that involve constants should come before checks that involve state variables, function calls, and calculations. By doing these checks first, the function is able to revert before wasting a Gcoldsload (2100 gas) in a function that may ultimately revert in the unhappy case.

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L182-L192

### Validate function parameters before making any state reads

```solidity
File: /src/Lender.sol
182:    function addToPool(bytes32 poolId, uint256 amount) external {
183:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
184:        if (amount == 0) revert PoolConfig();
```

The first check in the above, reads a state variable `pools[poolId]` and compares it to `msg.sender` The second check only reads a function parameter comparing it against 0.
If the first check passes but we revert on the second case, the gas spent doing the state read on the first check would be wasted. As it's cheaper to read function parameter, the check should be done first.

```diff
diff --git a/src/Lender.sol b/src/Lender.sol
index d04d14f..0c6388d 100644
--- a/src/Lender.sol
+++ b/src/Lender.sol
@@ -180,8 +180,8 @@ contract Lender is Ownable {
     /// @param poolId the id of the pool to add to
     /// @param amount the amount to add
     function addToPool(bytes32 poolId, uint256 amount) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         if (amount == 0) revert PoolConfig();
+        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         _updatePoolBalance(poolId, pools[poolId].poolBalance + amount);
         // transfer the loan tokens from the lender to the contract
         IERC20(pools[poolId].loanToken).transferFrom(
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L198-L200

### No need to validate state variables if we might end up reverting on a cheaper function parameter

```solidity
File: /src/Lender.sol
198:    function removeFromPool(bytes32 poolId, uint256 amount) external {
199:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
200:        if (amount == 0) revert PoolConfig();
```

Similar explanation to the previous case, we reorder the checks here to validate function parameters first

```diff
     function removeFromPool(bytes32 poolId, uint256 amount) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         if (amount == 0) revert PoolConfig();
+        if (pools[poolId].lender != msg.sender) revert Unauthorized();
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L210-L212

### Reorder the checks to validate cheaper variables first

```solidity
File: /src/Lender.sol
210:    function updateMaxLoanRatio(bytes32 poolId, uint256 maxLoanRatio) external {
211:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
212:        if (maxLoanRatio == 0) revert PoolConfig();
```

`maxLoanRatio` is a function parameter therefore cheaper to read compared to the state read `pools[poolId].lender`. In case of a revert on the cheaper check, we want to minimize the gas spent, thus should validate the function parameter first

```diff
     function updateMaxLoanRatio(bytes32 poolId, uint256 maxLoanRatio) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         if (maxLoanRatio == 0) revert PoolConfig();
+        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         pools[poolId].maxLoanRatio = maxLoanRatio;
         emit PoolMaxLoanRatioUpdated(poolId, maxLoanRatio);
     }
```

https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221-L223

### Validate function parameters before reading from state

```solidity
File: /src/Lender.sol
221:    function updateInterestRate(bytes32 poolId, uint256 interestRate) external {
222:        if (pools[poolId].lender != msg.sender) revert Unauthorized();
223:        if (interestRate > MAX_INTEREST_RATE) revert PoolConfig();
```

As the first check reads from storage,`pools[poolId].lender` , The second check is way cheaper as it only involves reading a function parameter and a constant value. If we end up reverting on the second check, the gas spent making the state read in the first check would be wasted. Reorder the checks to have the cheaper check first

```diff
     function updateInterestRate(bytes32 poolId, uint256 interestRate) external {
-        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         if (interestRate > MAX_INTEREST_RATE) revert PoolConfig();
+        if (pools[poolId].lender != msg.sender) revert Unauthorized();
         pools[poolId].interestRate = interestRate;
         emit PoolInterestRateUpdated(poolId, interestRate);
     }
```

## Conclusion

It is important to emphasize that the provided recommendations aim to enhance the efficiency of the code without compromising its readability. We understand the value of maintainable and easily understandable code to both developers and auditors.

As you proceed with implementing the suggested optimizations, please exercise caution and be diligent in conducting thorough testing. It is crucial to ensure that the changes are not introducing any new vulnerabilities and that the desired performance improvements are achieved. Review code changes, and perform thorough testing to validate the effectiveness and security of the refactored code.

Should you have any questions or need further assistance, please don't hesitate to reach out.
