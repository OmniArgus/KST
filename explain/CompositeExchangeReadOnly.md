I'll examine the `CompositeExchangeReadOnly.sol` file to provide a comprehensive explanation.

[13 tools called]

## CompositeExchangeReadOnly.sol - Explanation

`CompositeExchangeReadOnly.sol` is a **standalone read-only contract** that provides gas-free query functions for retrieving exchange state and user data. Unlike the main `CompositeExchange` contract which handles trading and state-modifying operations, this contract focuses exclusively on **view functions** for external querying.

---

## **Purpose and Architecture**

```1:11:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.10;

import {CompositeExchangeStorage} from "./CompositeExchangeStorage.sol";
import "./pub/PublicStruct.sol";
import "./CompactStruct.sol";
import "./StorageUtils64.sol";
import "./AggregateFundingRateLib.sol";
import "./UserBitSet.sol";

contract CompositeExchangeReadOnly is CompositeExchangeStorage {
```

**Key Characteristics:**
- **Inherits from `CompositeExchangeStorage`**: Shares the same storage layout and areas as `CompositeExchange`
- **Independent Deployment**: Deployed as a separate contract, not part of the proxy pattern
- **All Functions are `external view`**: No state modifications, making all calls gas-free from external calls
- **Used by UIs and Indexers**: Provides efficient read access for wallets, frontends, and off-chain services

---

## **Core Functionality Groups**

### **1. User Identification and Mapping**

```14:20:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function getUserId(address userAddress) external view returns (uint64) {
        return uint64(StorageUtils64.readStorageForAddress(AREA_USER_ADDRESS_TO_ID, userAddress));
    }

    function getUserAddress(uint64 userId) external view returns (address) {
        return address(uint160(StorageUtils64.readStorage(AREA_USER_ID_TO_ADDRESS, userId)));
    }
```

- **Bidirectional Mapping**: Converts between Ethereum addresses and internal user IDs (uint64)
- **Gas Optimization**: The exchange uses compact uint64 user IDs internally instead of 160-bit addresses
- **Returns 0**: If user is not registered

### **2. Vault Ledger Queries**

```22:24:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readLedger(uint64 userId, uint32 tokenId) external view returns(VaultLedger) {
        return internalReadLedger(userId, tokenId);
    }
```

**VaultLedger Structure** (from PublicStruct.sol):
```python
# BitFormat: perpSeqBalance(64) | spotLendSeqBalance(64) | userBalance(128)
```
- **`userBalance`**: Total balance in vault decimals
- **`spotLendSeqBalance`**: Amount locked for spot/lending orders
- **`perpSeqBalance`**: Amount locked for perpetual orders
- **Available Balance**: `userBalance - sequestered amounts`

### **3. Token Configuration Queries**

```30:41:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function getHighestTokenId() external view returns (uint32) {
        return uint32(StorageUtils64.readStorage(AREA_TOKEN_CONFIG, TOKEN_META));
    }

    // returns token configuration compact struct
    function readTokenConfig(uint32 tokenId) external view returns (VaultTokenConfig) {
        return internalReadTokenConfig(tokenId);
    }

    function getDefaultErc20TokenConfig(address erc20) external view returns (VaultTokenConfig) {
        return readDefaultErc20TokenConfig(erc20);
    }
```

**VaultTokenConfig** contains:
- **Decimal Conversions**: Native ERC20 decimals, vault decimals, position decimals
- **Risk Parameters**: Margin requirements, liquidation thresholds, haircuts
- **Token Metadata**: Symbol, ERC20 address, sequestration multiplier

### **4. OrderBook Discovery**

```43:59:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readOrderBookConfig(address book) external view returns(OrderBookConfig) {
        return OrderBookConfig.wrap(StorageUtils64.readStorageForAddress(AREA_ORDERBOOK_TO_CONFIG, book));
    }

    function getLendOrderBook(uint32 tokenId) external view returns (address){
        return internalGetLendBook(tokenId);
    }

    // returns zeros if not found
    function getSpotOrderBook(uint32 token1, uint32 token2) external view returns (address, uint32 buyToken, uint32 payToken) {
        return getGenericOrderBook(AREA_PAIR_TO_SPOTBOOK, token1, token2);
    }

    // returns zeros if not found
    function getPerpOrderBook(uint32 token1, uint32 token2) external view returns (address, uint32 buyToken, uint32 payToken) {
        return getGenericOrderBook(AREA_PAIR_TO_PERPBOOK, token1, token2);
    }
```

- **Reverse Lookup**: Given an address, check if it's a registered orderbook
- **Token Pair Lookup**: Find the orderbook for a specific token pair
- **Bidirectional Search**: Automatically checks both (token1, token2) and (token2, token1) combinations

### **5. Position Queries**

#### **Lending Positions**

```80:102:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readLenderPositions(uint64 accountId, uint32 tokenId, uint32 max, uint32 startPosition) external view returns(uint256[] memory){
        uint64 lenderArea = userIdToLendingLists(accountId);
        uint64 lendListId = tokenIdToLenderListId(tokenId);
        (ListIterator it, uint64 firstItem, uint32 totalLen) = StorageUtils64.loopStartFrom(lenderArea, lendListId, startPosition);
        if (totalLen == 0) {
            return new uint[](1);
        }
        unchecked { // loop variables can't overflow
            uint itemsToRead = totalLen - startPosition;
            if (max > 0 && itemsToRead > max) {
                itemsToRead = max;
            }
            uint[] memory result = new uint[](1+itemsToRead);
            result[0] = uint(totalLen);
            result[1] = LendingPositionPlib.fromLenderMatch(LendMatch.wrap(StorageUtils64.readStorage(AREA_LENDING_POSITIONS, firstItem)), firstItem).raw();
            for(uint i = 2; i <= itemsToRead; i++) {
                uint64 item;
                (it, item) = it.loopNext(lenderArea, lendListId);
                result[i] = LendingPositionPlib.fromLenderMatch(LendMatch.wrap(StorageUtils64.readStorage(AREA_LENDING_POSITIONS, item)), item).raw();
            }
            return result;
        }
    }
```

**Key Features:**
- **Pagination Support**: `max` limits results, `startPosition` for offset
- **Total Count**: First element `result[0]` contains total position count
- **Compact Return**: Returns `uint256[]` where each element is a packed `LendingPosition`
- **LIFO Iteration**: Traverses doubly-linked list of lending positions

#### **Perpetual Positions**

```108:130:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readPerpPositions(uint64 accountId, uint32 tokenId, uint32 max, uint32 startPosition) external view returns(uint256[] memory){
        uint64 perpArea = userIdToPerpList(accountId);
        uint64 perpListId = uint64(tokenId);
        (ListIterator it, uint64 firstItem, uint32 totalLen) = StorageUtils64.loopStartFrom(perpArea, perpListId, startPosition);
        if (totalLen == 0) {
            return new uint[](1);
        }
        unchecked { // loop variables can't overflow
            uint itemsToRead = totalLen - startPosition;
            if (max > 0 && itemsToRead > max) {
                itemsToRead = max;
            }
            uint[] memory result = new uint[](1+itemsToRead);
            result[0] = uint(totalLen);
            result[1] = convertToPerpPosition(accountId, firstItem, tokenId).raw();
            for(uint i = 2; i <= itemsToRead; i++) {
                uint64 item;
                (it, item) = it.loopNext(perpArea, perpListId);
                result[i] = convertToPerpPosition(accountId, item, tokenId).raw();
            }
            return result;
        }
    }
```

**Position Reconstruction:**

```132:142:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function convertToPerpPosition(uint64 user1, uint64 listItem, uint32 tokenId) internal view returns (PerpPosition) {
        uint key = listItem;
        if ((key & (uint(1) << 44)) != 0) {
            //user1 is long
            key = ((uint(user1) << 44) | (key & BLANK_BIT_AT_44));
        } else {
            key = ((key & BLANK_BIT_AT_44) << 44) | uint(user1);
        }
        PerpMatch pm = PerpMatch.wrap(StorageUtils64.readStorage128(tokenIdToPerpMatch(tokenId), uint128(key)));
        return pm.toPerpPosition(key);
    }
```

- **Long/Short Detection**: Bit 44 indicates if user is long or short
- **Key Construction**: Builds storage key from long ID and short ID
- **Two-Sided Storage**: Each perp position is stored once but accessible from both users' lists

### **6. Aggregated Position Data**

```26:28:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readPerpAggPosition(uint64 userId, uint32 tokenId) external view returns(PerpAggPosition, int256) {
        return (internalReadPerpAgg(userId, tokenId), internalReadPerpAggOwedBase(userId, tokenId));
    }
```

**PerpAggPosition** (Aggregated Perpetual):
- **Net Position**: Sum of all long/short positions for a user+token
- **Entry Price**: Weighted average entry price
- **Owed Nominal**: Funding payment due (positive = user receives)
- **Start Time**: Last true-up time for funding calculation

```104:106:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function readLendingAggregation(uint64 userId, uint32 tokenId) external view returns (LendAggPosition) {
        return readLendAgg(userId, tokenId);
    }
```

**LendAggPosition** (Aggregated Lending):
- **Lender Quantity**: Total amount lent out
- **Borrower Quantity**: Total amount borrowed
- **Highest Interest Rate**: Max rate across all positions

### **7. Interest and Fee Calculations**

```172:179:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function calculateInterestAndFeesDue(uint64 positionId) external view returns(uint128) {
        LendMatch lendMatch = readLendMatch(positionId);
        require(lendMatch.isValid(), ExchangeErrors.NoSuchPosition()); // no such position
        uint128 interest;
        uint128 fees;
        (interest, fees, ) = calcInterestAndFeesInternal(lendMatch, uint32(positionId));
        return interest + fees;
    }
```

- **Position-Specific**: Calculates for a specific lending position
- **Time-Based**: Uses current block timestamp vs position start time
- **Vault Decimals**: Returns value in vault decimals (not position decimals)

### **8. Funding Rate Queries**

```189:200:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function summedFundingRate(uint32 tokenId, uint32 from, uint32 to) external view returns (int32) {
        return AggregateFundingRateLib.summedRateFromTo(tokenIdToFundingRateArea(tokenId), from, to);
    }

    function readFundingRateStats(uint32 tokenId) external view returns (FundingRateStats) {
        return readAvgFundingRate(tokenId).asFundingRateStats();
    }

    function readFundingRate(uint64 time, uint32 tokenId) external view returns(int16) {
        uint32 fundingTime = uint32((time - JAN_1_2024)/3600/8);
        return int16(AggregateFundingRateLib.summedRateFromTo(tokenIdToFundingRateArea(tokenId), fundingTime, fundingTime + 1));
    }
```

- **Historical Rates**: Query funding rate at any past timestamp
- **Range Queries**: Sum rates over arbitrary time periods using binary aggregation
- **Stats**: Average rate, volume-weighted metrics

### **9. Risk and Portfolio Valuation**

```181:187:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function riskAdjustedPortfolioValue(uint64 userId) external view returns (int64) {
        return computePortfolioWithRisk(userId, false);
    }

    function userBits(uint64 userId) external view returns (uint256, uint256) {
        return (UserBitSet.firstBits(AREA_DEBT_BITSET, userId), UserBitSet.firstBits(AREA_NO_DEBT_BITSET, userId));
    }
```

**Risk-Adjusted Portfolio Value:**
- **Collateral Haircuts**: Applies discounts to asset values
- **Position Risk**: Evaluates perp exposure and lending risk
- **Negative Value**: Indicates eligible for liquidation
- **Base Token Denominated**: Returns value in base token with position decimals

**User Bits (Optimization):**
- **Debt Bitset**: Tokens where user has debt/borrowing
- **No-Debt Bitset**: Tokens with positive balances
- **Sparse Iteration**: Allows skipping empty token positions in portfolio valuation

### **10. Mark Price Configuration**

```202:204:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function getMarkPriceConfig(uint32 tokenId) external view returns (MarkPriceConfig) {
        return MarkPriceConfig.wrap(StorageUtils64.readStorage(AREA_MARK_PRICE_CONFIG, uint64(tokenId)));
    }
```

- **Oracle Configuration**: Specifies price source (Pyth, Chainlink, internal)
- **Mark-to-Market**: Used for perp settlement and liquidation checks

---

## **Relationship to Main Exchange**

```1:14:BaseDEX/contracts/src/main/sol/pub/ICompositeExchangePub.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.10;

import {ICompositeExchangeReadOnly} from "./ICompositeExchangeReadOnly.sol";
import {ICompositeExchangeTrading} from "./ICompositeExchangeTrading.sol";
import {ICompositeExchangeUser} from "./ICompositeExchangeUser.sol";
import {ICompositeExchangeBulk} from "./ICompositeExchangeBulk.sol";

/// public sdk's should code to this interface
interface ICompositeExchangePub is ICompositeExchangeReadOnly, ICompositeExchangeTrading,
                ICompositeExchangeUser, ICompositeExchangeBulk {

}
```

**Design Pattern:**
- **Separation of Concerns**: Read-only vs state-modifying operations
- **Multiple Implementations**: 
  - `CompositeExchange` + proxies implement full interface
  - `CompositeExchangeReadOnly` implements only view functions
- **Shared Storage**: Both access the same storage areas via `CompositeExchangeStorage`

**Usage in OrderBooks:**

```24:34:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function internalReadExchangeAsRO() internal view returns (ICompositeExchangeReadOnly) {
        return ICompositeExchangeReadOnly(address(uint160(TempStorageUtils64.readStorage(AREA_PARAMS, PARAM_EXCHANGE))));
    }

    function internalReadExchange() internal view returns (ICompExchForBooks) {
        return ICompExchForBooks(address(uint160(TempStorageUtils64.readStorage(AREA_PARAMS, PARAM_EXCHANGE))));
    }

    function internalReadTokenConfig(uint32 tokenId) internal view returns (VaultTokenConfig) {
        return internalReadExchangeAsRO().readTokenConfig(tokenId);
    }
```

OrderBooks cast the exchange address to `ICompositeExchangeReadOnly` when only querying data.

---

## **Gas Efficiency & Design Insights**

### **1. Compact Data Return**
All positions return `uint256[]` arrays where each element is a packed compact struct, minimizing memory allocation and ABI encoding costs.

### **2. Pagination**
Position queries support `max` and `startPosition` to avoid hitting gas limits when users have many positions.

### **3. Bitmap Optimization**
```185:187:BaseDEX/contracts/src/main/sol/CompositeExchangeReadOnly.sol
    function userBits(uint64 userId) external view returns (uint256, uint256) {
        return (UserBitSet.firstBits(AREA_DEBT_BITSET, userId), UserBitSet.firstBits(AREA_NO_DEBT_BITSET, userId));
    }
```
Allows UIs to quickly identify which tokens to query for a user without iterating all tokens.

### **4. Standalone Deployment**
Being separate from the main exchange:
- **Can be called directly**: No proxy overhead for read operations
- **Multiple instances**: Could deploy multiple read contracts for load distribution
- **Interface Stability**: Read interface less likely to change than trading logic

---

## **Key Differences from CompositeExchangeBulk**

| Feature | CompositeExchangeReadOnly | CompositeExchangeBulk |
|---------|---------------------------|------------------------|
| **Access** | Standalone contract | Via proxy (selector `0xFFFFFFxx`) |
| **Purpose** | General read queries | Optimized bulk/aggregated reads |
| **Functions** | Individual position/config queries | Bulk config dumps, aggregations |
| **Usage** | Direct contract calls | Through main exchange proxy |

---

## **Security Considerations**

1. **Pure View Functions**: Cannot modify state, safe for any caller
2. **No Access Control**: All functions are publicly readable
3. **Shared Storage**: Must maintain identical storage layout to `CompositeExchange`
4. **Position Privacy**: All positions are publicly readable on-chain
5. **Stale Data**: Off-chain indexers should handle reorgs when caching this data

---

## **Summary**

`CompositeExchangeReadOnly.sol` is a **gas-free query layer** for the BaseDEX exchange, providing:

- **User & Token Lookups**: Address↔ID mappings, token configs
- **Balance & Ledger Queries**: Vault balances, sequestered amounts
- **Position Retrieval**: Paginated lending/perp position lists
- **Risk Calculations**: Portfolio valuation, liquidation checks
- **Funding & Interest**: Historical rates, accrued interest
- **OrderBook Discovery**: Finding registered orderbooks by token pair

It's designed for **external consumption** by UIs, wallets, indexers, and monitoring tools, with emphasis on efficiency through compact structs, pagination, and bitmap optimizations.