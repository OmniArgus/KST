Examining `LiquidationHelper.sol` to understand its purpose and implementation.


[4 tools called]

Checking how liquidation bots use this contract and examining related constants:


[4 tools called]

Checking how `userBits` works to understand the filtering logic:


[2 tools called]

## LiquidationHelper.sol - Explanation

`LiquidationHelper.sol` is an off-chain helper contract for liquidation bots. It provides efficient bulk reads for scanning users and retrieving portfolio data to identify liquidation candidates. It's designed for off-chain use (view functions) and optimizes data retrieval.

---

## Purpose and Architecture

```48:48:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
contract LiquidationHelper {
```

**Key Characteristics:**
- **View-only contract**: All functions are `external view`; no state changes
- **Off-chain use**: Designed for liquidation bots (Java)
- **Gas-efficient data packing**: Compact structs to minimize return data size
- **Debt filtering**: Only includes users with debt (`debtBits != 0`)
- **Batch operations**: Supports sequential scanning and targeted batch reads

---

## Compact Data Structures

### 1. UserTokenCountBalance
```7:9:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
/// tokenCount is the number of tokens that have non-zero balances/positions, including the base token\
/// BitFormat: baseBal (128) | tokenCount (32) | userId (44)
type UserTokenCountBalance is uint256;
```

**Bit Layout:**
- Bits 0-43: `userId` (44 bits)
- Bits 44-75: `tokenCount` (32 bits)
- Bits 76-203: `baseBal` (128 bits)

**Purpose**: First slot per user with user ID, token count, and base token balance.

### 2. TokenIdAndBalance
```11:12:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
/// BitFormat: tokenBal (128) | tokenId (32) | zeros (44)
type TokenIdAndBalance is uint256;
```

**Bit Layout:**
- Bits 0-43: `zeros` (44 bits, always 0 to differentiate from user entries)
- Bits 44-75: `tokenId` (32 bits)
- Bits 76-203: `tokenBal` (128 bits)

**Purpose**: Identifies token balances (LSB 44 bits are 0, unlike user entries).

### 3. FiveUsers
```17:18:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
/// BitFormat: user5 (44) | user4 (44) | ... | user1 (44)
type FiveUsers is uint256;
```

**Purpose**: Packs 5 user IDs into one `uint256` (44 bits each) for batch reads.

### 4. TwoMarks
```30:31:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
/// BitFormat: token2 (32) | mark2 (64) | token1 (32) | mark1 (64)
type TwoMarks is uint256;
```

**Purpose**: Packs 2 mark prices into one `uint256` for efficient price retrieval.

---

## Main Functions

### 1. Sequential Portfolio Scanning

```58:103:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
    function readUserPortfolios(address exchange, uint64 firstUser, uint32 probeCount, uint32 maxUsers) external view returns (uint64, uint256[] memory) {
        ICompositeExchangePub exch = ICompositeExchangePub(exchange);
        uint32 maxToken = exch.getHighestTokenId();
        uint64 endUser = exch.bulkReadMaxUserId_5445644137();
        if (endUser > firstUser + probeCount) {
            endUser = firstUser + probeCount;
        }
        uint32 tokenCount = maxToken - 2; // token ids 2 and 3 are not used
        uint256[] memory result = new uint256[](maxUsers * (2 + 4 * tokenCount));
        uint256 count = 0;
        uint256 pos = 0;
        uint64 user = firstUser > 10 ? firstUser : 11;
        while (count < maxUsers && user <= endUser) {
            (uint256 debtBits, uint256 noDebtBits) = exch.userBits(user);
            if (debtBits != 0) {
                ++count;
                uint256 allTokens = debtBits | noDebtBits;
                uint32 tokens = countTokens(allTokens >> 32);
                (uint128 baseBal,, ) = exch.getBalance(user, BASE_TOKEN_ID);
                result[pos] = uint(user) | (uint(tokens) << 44) | (uint(baseBal) << 76);
                ++pos;
                result[pos] = exch.bulkReadLendingAgg_9676672908(user, BASE_TOKEN_ID).raw();
                ++pos;
                for (uint32 tokenId=4; tokenId <= maxToken; tokenId++) {
                    if (allTokens & (uint(1) << (tokenId + 32)) != 0) {
                        (uint128 bal,, ) = exch.getBalance(user, tokenId);
                        result[pos] = (uint(tokenId) << 44) | (uint(bal) << 76);
                        ++pos;
                        result[pos] = exch.bulkReadLendingAgg_9676672908(user, tokenId).raw();
                        ++pos;
                        (PerpAggPosition pap, int256 b) = exch.readPerpAggPosition(user, tokenId);
                        result[pos] = pap.raw();
                        ++pos;
                        result[pos] = uint256(b);
                        ++pos;
                    }
                }
            }
            ++user;
        }

        assembly ("memory-safe") {
            mstore(result, pos)
        }
        return (user, result);
    }
```

**Purpose**: Sequentially scans users and returns portfolio data for users with debt.

**Parameters:**
- `exchange`: CompositeExchange address
- `firstUser`: Starting user ID
- `probeCount`: Max users to scan
- `maxUsers`: Max users to return (only those with debt)

**Return Value:**
- `uint64`: Last probed user ID (for pagination)
- `uint256[]`: Portfolio data array

**Data Structure per User:**
1. Slot 0: `UserTokenCountBalance` (userId, tokenCount, baseBalance)
2. Slot 1: `BulkLendAggPosition` for base token
3. For each non-zero token (tokenId >= 4):
   - Slot N: `TokenIdAndBalance` (tokenId, balance)
   - Slot N+1: `BulkLendAggPosition` for token
   - Slot N+2: `PerpAggPosition` (raw)
   - Slot N+3: `PerpAggOwedBase` (signed int256)

**Filtering Logic:**
- Skips users with `debtBits == 0` (no debt)
- Starts at user 11 (users 1-10 reserved)
- Uses `userBits()` to identify tokens with balances/positions

**Usage Example** (from Java code):
```java
// Sequential scanning with pagination
Tuple2<BigInteger, List<BigInteger>> tuple = helper.readUserPortfolios(
    exchangeAddress, BigInteger.ONE, BigInteger.valueOf(10000), BigInteger.valueOf(1000)
);
// Continue until done
while (tuple.component1().compareTo(maxUserId) < 0) {
    tuple = helper.readUserPortfolios(exchangeAddress,
        tuple.component1(), BigInteger.valueOf(10000), BigInteger.valueOf(1000)
    );
}
```

### 2. Batch Portfolio Reading

```120:162:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
    function batchReadPortfolios(address exchange, FiveUsers[] calldata users) external view returns (uint256[] memory) {
        ICompositeExchangePub exch = ICompositeExchangePub(exchange);
        uint32 maxToken = exch.getHighestTokenId();
        uint64 maxUsers = uint64(users.length * 5);
        uint32 tokenCount = maxToken - 2; // token ids 2 and 3 are not used
        uint256[] memory result = new uint256[](maxUsers * (2 + 4 * tokenCount));
        uint256 pos = 0;
        for(uint i=0;i<users.length;i++) {
            for (uint j=0;j<5;j++) {
                uint64 user = uint64((users[i].raw() >> (44 * j)) & CLIENT_ID_MASK);
                if (user != 0) {
                    (uint256 debtBits, uint256 noDebtBits) = exch.userBits(user);
                    uint256 allTokens = debtBits | noDebtBits;
                    uint32 tokens = countTokens(allTokens >> 32);
                    (uint128 baseBal,, ) = exch.getBalance(user, BASE_TOKEN_ID);
                    result[pos] = uint(user) | (uint(tokens) << 44) | (uint(baseBal) << 76);
                    ++pos;
                    result[pos] = exch.bulkReadLendingAgg_9676672908(user, BASE_TOKEN_ID).raw();
                    ++pos;
                    for (uint32 tokenId=4; tokenId <= maxToken; tokenId++) {
                        if (allTokens & (uint(1) << (tokenId + 32)) != 0) {
                            (uint128 bal,, ) = exch.getBalance(user, tokenId);
                            result[pos] = (uint(tokenId) << 44) | (uint(bal) << 76);
                            ++pos;
                            result[pos] = exch.bulkReadLendingAgg_9676672908(user, tokenId).raw();
                            ++pos;
                            (PerpAggPosition pap, int256 b) = exch.readPerpAggPosition(user, tokenId);
                            result[pos] = pap.raw();
                            ++pos;
                            result[pos] = uint256(b);
                            ++pos;
                        }
                    }

                }
            }
        }

        assembly ("memory-safe") {
            mstore(result, pos)
        }
        return result;
    }
```

**Purpose**: Reads portfolios for specific users (e.g., after updates).

**Parameters:**
- `exchange`: CompositeExchange address
- `users`: Array of `FiveUsers` (5 user IDs per element)

**Differences from `readUserPortfolios`:**
- No pagination (specific users)
- No `debtBits` filtering (reads all provided users)
- Used for targeted updates after events

**Usage Example** (from Java code):
```java
// Pack users into FiveUsers format
List<BigInteger> fiveUserList = new ArrayList<>();
BigInteger v = BigInteger.ZERO;
for(Long u: users) {
    count++;
    v = v.shiftLeft(44).or(BigInteger.valueOf(u));
    if (count == 5) {
        fiveUserList.add(v);
        v = BigInteger.ZERO;
        count = 0;
    }
}
// Batch read
List<BigInteger> portfolios = helper.batchReadPortfolios(exchangeAddress, fiveUserList).send();
```

### 3. Mark Price Reading

```164:180:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
    function readAllMarks(address exchange) external view returns (uint256[] memory) {
        ICompositeExchangePub exch = ICompositeExchangePub(exchange);
        uint32 maxToken = exch.getHighestTokenId();
        uint32 tokenCount = maxToken - 3; // token ids 2 and 3 are not used, base has no mark
        uint256[] memory result = new uint256[](tokenCount/2 + (tokenCount & 1));
        for (uint32 tokenId=4; tokenId <= maxToken; tokenId++) {
            Price59EN5 p = exch.getMarkPrice(tokenId);
            uint r = w64(p.raw()) | (w32(tokenId) << 64);
            uint index = (tokenId - 4)/2;
            if (tokenId & 1 == 0) {
                result[index] = r;
            } else {
                result[index] = TwoMarks.wrap(result[index]).second(r).raw();
            }
        }
        return result;
    }
```

**Purpose**: Retrieves all mark prices efficiently.

**Optimization:**
- Packs 2 prices per `uint256` (even tokenIds in low 96 bits, odd in high 96 bits)
- Reduces array size by ~50%

**Bit Layout per `TwoMarks`:**
- Bits 0-31: `tokenId1` (32 bits)
- Bits 32-95: `mark1` (64 bits, Price59EN5)
- Bits 96-127: `tokenId2` (32 bits)
- Bits 128-191: `mark2` (64 bits, Price59EN5)

---

## Helper Functions

### Token Counting

```105:110:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
    function countTokens(uint256 v) internal pure returns (uint32) {
        return uint32(count64BitsSet(v & SIXTY_FOUR_BIT_MASK) +
            count64BitsSet((v >> 64) & SIXTY_FOUR_BIT_MASK) +
            count64BitsSet((v >> 128) & SIXTY_FOUR_BIT_MASK) +
            count64BitsSet((v >> 192) & SIXTY_FOUR_BIT_MASK));
    }
```

**Purpose**: Counts set bits in four 64-bit chunks (popcount).

### Bit Counting Algorithm

```112:118:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
    function count64BitsSet(uint256 i) internal pure returns (uint256) {
        unchecked {
            i = i - ((i >> 1) & 0x5555555555555555);
            i = (i & 0x3333333333333333) + ((i >> 2) & 0x3333333333333333);
            return uint64(((i + (i >> 4)) & 0xF0F0F0F0F0F0F0F) * 0x101010101010101) >> 56;
        }
    }
```

**Purpose**: Efficient popcount using bit manipulation (parallel bit counting).

---

## Usage in Liquidation Bots

### Initial Scanning Flow

1. **Sequential Scan**: `readUserPortfolios()` scans users sequentially
2. **Debt Filtering**: Only users with `debtBits != 0` are included
3. **Portfolio Evaluation**: Off-chain bots evaluate portfolios for liquidation eligibility
4. **Mark Price Fetching**: `readAllMarks()` gets current prices

### Update Flow

1. **Event Monitoring**: Bots monitor for user activity events
2. **Targeted Updates**: `batchReadPortfolios()` reads specific users
3. **Re-evaluation**: Portfolios are re-evaluated for liquidation eligibility

### Integration Example

From Java liquidation bot code:
```java
// Sequential scanning
Tuple2<BigInteger, List<BigInteger>> tuple = helper.readUserPortfolios(
    exchangeAddress, BigInteger.ONE, BigInteger.valueOf(10000), BigInteger.valueOf(1000)
);
List<Portfolio> portfolios = Portfolio.decodeList(state, tokenConfigs, list);

// Batch reading for updates
List<BigInteger> fiveUserList = packUsersIntoFiveUsers(usersToUpdate);
List<BigInteger> portfolios = helper.batchReadPortfolios(exchangeAddress, fiveUserList).send();
```

---

## Gas Optimization Techniques

### 1. Memory Array Resizing
```99:101:BaseDEX/contracts/src/main/sol/LiquidationHelper.sol
        assembly ("memory-safe") {
            mstore(result, pos)
        }
```

**Purpose**: Resizes the array to actual length using assembly to reduce return data size.

### 2. Bit Packing
- 5 user IDs per `uint256` (FiveUsers)
- 2 mark prices per `uint256` (TwoMarks)
- Portfolio data packed into compact structs

### 3. Debt Filtering
- Skips users with no debt early
- Reduces unnecessary reads

### 4. Efficient Token Detection
- Uses `userBits()` bitmaps to identify active tokens
- Iterates only over tokens with balances/positions

---

## Summary

`LiquidationHelper.sol` is an off-chain helper contract for liquidation bots that:
- Provides efficient bulk portfolio scanning and batch reads
- Filters users by debt presence
- Packs data into compact structs
- Supports sequential scanning with pagination
- Enables targeted batch reads for updates
- Retrieves mark prices efficiently

It enables off-chain bots to efficiently scan users, identify liquidation candidates, and monitor portfolio changes while minimizing on-chain data retrieval costs.