Searching for `UserBitSet.sol` and reviewing its implementation.


[2 tools called]

Reading the full `UserBitSet.sol` file:


[3 tools called]

## **UserBitSet.sol - Explanation**

### **Purpose and importance**

`UserBitSet` is a library for storing sparse bitmaps per user, optimized for BaseDEX portfolio valuation. It tracks which tokens have debt (`AREA_DEBT_BITSET`) and which don't (`AREA_NO_DEBT_BITSET`) to avoid scanning all tokens.

### **Design**

#### **1. Storage scheme**

```5:8:BaseDEX/contracts/src/main/sol/UserBitSet.sol
// can store up to 2^20*192 bits for the given user
// bit 0 is expected to be zero always
// stores max flipped bit in highest 32 bits of slot0
// the lower 192 bits of all slots are the bits
```

- Capacity: up to 2^20 × 192 = ~201 million bits per user
- Slot 0: highest 32 bits store max bit position; lower 192 bits store first 192 bits
- Subsequent slots: 192 bits each (not the full 256)
- Key format: `(storePos << 44) | userId` (20-bit storePos, 44-bit userId)

**Storage layout:**
- Slot 0 (`key = userId`): `[maxBitPosition (32 bits)] | [bits 0-191 (192 bits)]`
- Slot N (`key = (N << 44) | userId`): `[bits 192*N to 192*N+191 (192 bits)] | [unused (64 bits)]`

### **Core functions**

#### **1. `storeBit` - Setting a bit**

```17:35:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function storeBit(uint64 area, uint64 userId, uint32 bitPos) internal {
        uint64 key = userId;
        uint slot0 = StorageUtils64.readStorage(area, key);
        uint64 storePos = bitPos/192;
        require(storePos < (1 << 20)); // enforce max
        if (bitPos  > uint32(slot0 >> 224)) {
            slot0 = ((slot0 << 32) >> 32) | (uint(bitPos) << 224);
            if (storePos > 0) {
                StorageUtils64.writeStorage(area, key, slot0);
            }
        }
        uint prev = slot0;
        if (storePos > 0) {
            key = (storePos << 44) | userId;
            prev = StorageUtils64.readStorage(area, key);
        }
        prev = prev | (uint(1) << (bitPos % 192));
        StorageUtils64.writeStorage(area, key, prev);
    }
```

Steps:
1. Compute `storePos = bitPos / 192`
2. Read slot 0 to get current max bit
3. Update max bit if needed
4. Compute the target slot and bit position
5. Set the bit

Example:
- `bitPos = 384`
- `storePos = 384 / 192 = 2`
- Bit within slot: `384 % 192 = 0`
- Key: `(2 << 44) | userId`
- Set bit 0 in that slot

#### **2. `clearBit` - Clearing a bit**

```37:44:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function clearBit(uint64 area, uint64 userId, uint32 bitPos) internal {
        uint64 storePos = bitPos/192;
        require(storePos < (1 << 20));
        uint64 key = (storePos << 44) | userId;
        uint prev = StorageUtils64.readStorage(area, key);
        prev = prev & ~(uint(1) << (bitPos % 192));
        StorageUtils64.writeStorage(area, key, prev);
    }
```

- Computes slot and bit position
- Clears the bit with AND + NOT
- Note: doesn't update max bit (safe to leave stale)

#### **3. `isSet` - Checking if a bit is set**

```46:51:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function isSet(uint64 area, uint64 userId, uint32 bitPos) internal view returns (bool) {
        uint64 storePos = bitPos/192;
        uint64 key = (storePos << 44) | userId;
        uint prev = StorageUtils64.readStorage(area, key);
        return (prev & (uint(1) << (bitPos % 192))) != 0;
    }
```

- Read-only check using bitwise AND

#### **4. `firstBits` - Bulk read for first 2 slots**

```10:15:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function firstBits(uint64 area, uint64 userId) internal view returns (uint256) {
        uint v0 = StorageUtils64.readStorage(area, userId);
        uint v1 = StorageUtils64.readStorage(area, uint64(1 << 44) | userId);
        uint max = v0 >> 224; // move the max into the lower bits
        return max | (v0 << 32) | (v1 << 224);
    }
```

- Reads slots 0 and 1
- Returns: `[maxBit (32)] | [slot0 bits 0-191 (192)] | [slot1 bits 192-383 (192)]`
- Used for efficient bulk reads

### **Iterator functionality**

#### **1. Iterator creation**

```53:64:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function iterate(uint64 area, uint64 user) internal view returns (UserBitSetIterator) {
        return iterateStarting(area, user, 1);
    }

    function iterateStarting(uint64 area, uint64 user, uint32 pos) internal view returns (UserBitSetIterator) {
        uint slot = StorageUtils64.readStorage(area, user);
        uint32 inclusiveEnd = uint32(slot >> 224);
        if (inclusiveEnd == 0) {
            return UserBitSetIteratorLib.empty();
        }
        return UserBitSetIteratorLib.searchBits(area, user, 0, pos, 0, inclusiveEnd, slot);
    }
```

- Creates an iterator starting from position `pos`
- Uses `maxBit` from slot 0 as the upper bound

#### **2. Iterator structure**

```67:68:BaseDEX/contracts/src/main/sol/UserBitSet.sol
// further iterator: bits array (192) | inclusiveEnd (28) | offset (28) | pos (8)
type UserBitSetIterator is uint256;
```

Bit layout:
- `pos` (8 bits): bit position within current slot (0-191)
- `offset` (28 bits): global offset (slot number × 192)
- `inclusiveEnd` (28 bits): maximum bit position
- `bits` (192 bits): current slot's bit data

#### **3. Iterator search algorithm**

```71:91:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function searchBits(uint64 area, uint64 user, uint64 storePos, uint32 _pos, uint32 _offset,
        uint32 _inclusiveEnd, uint slot) internal view returns (UserBitSetIterator) {
        unchecked{
            while (_pos + _offset <= _inclusiveEnd) {
                while (_pos < 192 && _pos + _offset <= _inclusiveEnd && slot & (1 << _pos) == 0) {
                    _pos += 1;
                }
                if (_pos + _offset > _inclusiveEnd) {
                    return UserBitSetIteratorLib.empty();
                }
                if (_pos < 192) {
                    return UserBitSetIteratorLib.newIterator(uint8(_pos), _offset, _inclusiveEnd, slot);
                }
                _offset += _pos;
                storePos += 1;
                _pos = 0;
                slot = StorageUtils64.readStorage(area, (storePos << 44) | user);
            }
            return UserBitSetIteratorLib.empty();
        }
    }
```

Algorithm:
1. Scan within current slot for set bits
2. If found before slot end, return iterator
3. Otherwise, move to next slot
4. Continue until max bit or a set bit is found

#### **4. Iterator navigation**

```129:136:BaseDEX/contracts/src/main/sol/UserBitSet.sol
    function next(UserBitSetIterator _it, uint64 area, uint64 user) internal view returns(UserBitSetIterator, uint32) {
        uint32 _pos = UserBitSetIteratorLib.pos(_it);
        uint32 _offset = UserBitSetIteratorLib.offset(_it);
        uint32 _end = UserBitSetIteratorLib.inclusiveEnd(_it);
        unchecked{
            return (searchBits(area, user, _offset / 192, _pos + 1, _offset, _end, UserBitSetIterator.unwrap(_it) >> 64), _pos + _offset);
        }
    }
```

- Advances by one position
- Returns updated iterator and the global bit position

### **Primary usage: Portfolio valuation optimization**

#### **1. Debt and no-debt tracking**

```245:255:BaseDEX/contracts/src/main/sol/CompositeExchangeStorage.sol
    function setNoDebtBit(uint64 userId, uint32 tokenId) internal {
        UserBitSet.storeBit(AREA_NO_DEBT_BITSET, userId, tokenId);
    }

    function isNoDebtBitSet(uint64 userId, uint32 tokenId) internal view returns (bool) {
        return UserBitSet.isSet(AREA_NO_DEBT_BITSET, userId, tokenId);
    }

    function setDebtBit(uint64 userId, uint32 tokenId) internal {
        UserBitSet.storeBit(AREA_DEBT_BITSET, userId, tokenId);
    }
```

- `AREA_DEBT_BITSET`: tokens with debt
- `AREA_NO_DEBT_BITSET`: tokens with non-zero balances but no debt

#### **2. Efficient portfolio valuation**

```625:649:BaseDEX/contracts/src/main/sol/CompositeExchangeStorage.sol
    function computePortfolioWithRisk(uint64 userId, bool shortCircuitNoDebt) internal view returns (int64) {
        UserBitSetIterator debtIt = UserBitSet.iterateStarting(AREA_DEBT_BITSET, userId, BASE_TOKEN_ID + 1);
        int128 baseBal = computeBaseValuation(userId);
        if (shortCircuitNoDebt && !debtIt.hasNext() && baseBal >= 0) {
            return 0;
        }
        while(debtIt.hasNext()) {
            uint32 tokenId;
            (debtIt, tokenId) = debtIt.next(AREA_DEBT_BITSET, userId);
            baseBal = computeTokenValuation(userId, tokenId, baseBal);
        }
        UserBitSetIterator noDebtIt = UserBitSet.iterateStarting(AREA_NO_DEBT_BITSET, userId, BASE_TOKEN_ID + 1);
        while(noDebtIt.hasNext()) {
            uint32 tokenId;
            (noDebtIt, tokenId) = noDebtIt.next(AREA_NO_DEBT_BITSET, userId);
            if (!UserBitSet.isSet(AREA_DEBT_BITSET, userId, tokenId)) {
                baseBal = computeTokenValuationNoDebt(userId, tokenId, baseBal);
            }
            if (shortCircuitNoDebt && baseBal >= 0) {
                return 0; // all debts are accounted for, no point to go further
            }
        }
        require(!shortCircuitNoDebt || baseBal >= 0, ExchangeErrors.TradeOrWithdrawCausesLiquidation());
        return int64(baseBal);
    }
```

Benefits:
1. Iterates only tokens with debt or non-zero balances
2. Skips tokens with zero balances
3. Early exit when portfolio is positive (`shortCircuitNoDebt`)
4. O(S) where S = number of set bits, not total tokens

### **Comparison with BitSet.sol**

| Feature | `BitSet.sol` | `UserBitSet.sol` |
|---------|--------------|------------------|
| **Scope** | Global (no user context) | Per-user |
| **Storage** | `StorageUtils` (32-bit areas) | `StorageUtils64` (64-bit areas) |
| **Capacity** | 65,536 bits (uint16) | ~201 million bits (uint32) |
| **Slot Size** | 256 bits (full slot) | 192 bits per slot |
| **Max Bit Tracking** | No | Yes (in slot 0) |
| **Use Case** | Price levels in orderbooks | User token tracking |

### **Gas optimization techniques**

1. 192-bit slots: leaves space for metadata while using most of the slot
2. Max bit tracking: avoids scanning beyond the highest bit
3. Sparse iteration: only processes set bits
4. Bulk reads: `firstBits` reads two slots in one call
5. Compact keys: `(storePos << 44) | userId` encodes slot and user

### **Limitations**

1. Bit 0 is reserved and expected to be zero
2. Max bit not updated on `clearBit` (safe but potentially stale)
3. 192-bit slots (not 256) for efficiency trade-offs
4. Requires `StorageUtils64` (64-bit areas)

### **Example usage**

```solidity
// Set debt bit for token ID 5
UserBitSet.storeBit(AREA_DEBT_BITSET, userId, 5);

// Check if token has debt
bool hasDebt = UserBitSet.isSet(AREA_DEBT_BITSET, userId, 5);

// Iterate all tokens with debt
UserBitSetIterator debtIt = UserBitSet.iterate(AREA_DEBT_BITSET, userId);
while (debtIt.hasNext()) {
    uint32 tokenId;
    (debtIt, tokenId) = debtIt.next(AREA_DEBT_BITSET, userId);
    // Process token with debt
}
```

### **Summary**

`UserBitSet` provides sparse per-user bitmaps optimized for portfolio valuation. It stores up to ~201 million bits per user, tracks the maximum bit position, and supports efficient iteration. Used to track tokens with debt and tokens with non-zero balances to avoid scanning all tokens, improving gas efficiency for portfolio valuation.