Examining `StorageUtils64.sol` to explain its purpose and implementation.


[3 tools called]

## StorageUtils64.sol - Explanation

`StorageUtils64.sol` is a library for area-based storage partitioning using 64-bit area codes. It extends `StorageUtils` with list operations (LIFO/FIFO iteration) for managing perp positions, lending positions, and other linked structures. Used by the main exchange (`CompositeExchange`, `CompositeExchangeStorage`), which needs a larger namespace and list support.

---

## Purpose and Architecture

```8:8:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
library StorageUtils64 {
```

**Characteristics:**
- **64-bit Areas**: Uses `uint64` area codes (up to ~18.4 quintillion areas)
- **List Support**: Includes LIFO/FIFO list operations
- **Used by Main Exchange**: `CompositeExchange`, `CompositeExchangeStorage`
- **Assembly Optimized**: Uses inline assembly for efficiency

**Key Difference from StorageUtils:**
- **StorageUtils**: 32-bit areas, used by orderbooks
- **StorageUtils64**: 64-bit areas + list operations, used by main exchange

---

## Basic Storage Operations

### Read Storage

```9:16:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function readStorage(uint64 area, uint64 pos) internal view returns (uint) {
        uint result;
        assembly {
            let slot := or(shl(64, pos), area)
            result := sload(slot)
        }
        return result;
    }
```

**Slot Formula**: `slot = (pos << 64) | area`

**Bit Layout:**
- **Bits 0-63**: `area` (64 bits)
- **Bits 64-191**: `pos` (128 bits, but only 64 bits used due to shift)

**Example:**
- `area = 1000`, `pos = 42`
- `slot = (42 << 64) | 1000 = 774763251095801167000`

### Write Storage

```18:23:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function writeStorage(uint64 area, uint64 pos, uint value) internal {
        assembly {
            let slot := or(shl(64, pos), area)
            sstore(slot, value)
        }
    }
```

**Same pattern**: Identical slot calculation with `sstore`

### Address-Based Storage

```41:55:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function readStorageForAddress(uint64 area, address adr) internal view returns (uint) {
        uint result;
        assembly {
            let slot := or(shl(64, adr), area)
            result := sload(slot)
        }
        return result;
    }

    function writeStorageForAddress(uint64 area, address adr, uint value) internal {
        assembly {
            let slot := or(shl(64, adr), area)
            sstore(slot, value)
        }
    }
```

**Purpose**: Maps addresses to storage slots within an area

**Usage**: Orderbook address → configuration mapping

---

## List Storage System

### List Storage Scheme

```57:63:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    /// lists are written with the following scheme:
    /// a list requires two identifiers: area (64bit) and listId (64bit)
    /// the full key for the list members is (uint(listId) << 96) | (uint(chunkNumber) << 64) | uint(area)
    /// therefore, multiple lists may be stored in the same area, but the area should not contain
    /// any other storage
    /// lists store 64 bit values. Chunk 0 contains the list length and 3 values.
    /// all other chunks contain 4 values.
```

**Key Concepts:**
- **Area + ListId**: Identifies a list within an area
- **Chunk-Based Storage**: Lists stored in chunks (`uint256` slots)
- **Chunk 0**: Contains length (64 bits) + 3 values (192 bits)
- **Other Chunks**: Contains 4 values (256 bits = 4 × 64 bits)

**Slot Formula**: `slot = (listId << 96) | (chunkNumber << 64) | area`

**Bit Layout:**
- **Bits 0-63**: `area` (64 bits)
- **Bits 64-95**: `chunkNumber` (32 bits)
- **Bits 96-191**: `listId` (96 bits)

### Read List Chunk

```64:71:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function readListChunk(uint64 area, uint64 listId, uint32 chunkNumber) internal view returns (uint) {
        uint slot = (uint(listId) << 96) | (uint(chunkNumber) << 64) | uint(area);
        uint result;
        assembly {
            result := sload(slot)
        }
        return result;
    }
```

**Purpose**: Reads a specific chunk of a list

**Usage**: Used internally by list operations

### List Length Extraction

```73:75:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function listLength(uint chunk0) internal pure returns (uint64) {
        return uint64(chunk0);
    }
```

**Purpose**: Extracts list length from chunk 0

**Bit Position**: Lower 64 bits of chunk 0 contain the length

---

## List Operations

### Append to List

```86:119:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function appendList(uint64 area, uint64 listId, uint64 value) internal returns (uint32) {
        require(value != 0, ExchangeErrors.CantStoreZero()); //can't store zero
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }

        uint len = listLength(chunk0); //len is really uint32, checked below
        unchecked{
            chunk0 += 1; // increment length
            len += 1;
        }
        require(len < 0xffffffff, ExchangeErrors.ListTooLong()); //List too long
        if (len < 4) {
            chunk0 = chunk0 | (uint(value) << (len * 64));
        } else {
            uint slot = slot0 | ((uint(len >> 2) << 64));
            uint chunk;
            assembly {
                chunk := sload(slot)
            }
            unchecked{
                chunk = chunk | (uint(value) << (((len - 4) & 3) * 64));
            }
            assembly {
                sstore(slot, chunk)
            }
        }
        assembly {
            sstore(slot0, chunk0)
        }
        return uint32(len);
    }
```

**Flow:**
1. **Get Chunk 0**: Read current list state
2. **Increment Length**: Length is stored in lower 64 bits
3. **Check Limit**: Ensures length < 2^32
4. **Store Value**:
   - **If len < 4**: Store in chunk 0 (bits 64-255)
   - **Otherwise**: Store in appropriate chunk (`chunkNumber = len >> 2`)
5. **Position Calculation**: Uses `(len - 4) & 3` to find position within chunk

**Chunk 0 Layout:**
```
Bits: [255:192] Value 3 (64 bits)
      [191:128] Value 2 (64 bits)
      [127:64]  Value 1 (64 bits)
      [63:0]    Length (64 bits)
```

**Other Chunks Layout:**
```
Bits: [255:192] Value 4 (64 bits)
      [191:128] Value 3 (64 bits)
      [127:64]  Value 2 (64 bits)
      [63:0]    Value 1 (64 bits)
```

**Example**: Appending to list with length 5
- Length 5 means positions 0-4 are filled
- Next value goes to chunk 1, position `(5-4) & 3 = 1`
- Stores value at bits 64-127 of chunk 1

### Remove Last (LIFO Pop)

```131:138:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function removeLast(uint64 area, uint64 listId) internal returns (uint64) {
        uint slot0 = _asSlot0(area, listId);
        (uint chunk0, uint64 last, ) = privateRemoveLast(area, listId);
        assembly {
            sstore(slot0, chunk0)
        }
        return last;
    }
```

**Purpose**: Removes and returns the last element (LIFO behavior)

**Usage**: Used for perp position netting (LIFO iteration)

**Private Implementation**:

```199:227:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function privateRemoveLast(uint64 area, uint64 listId) internal returns(uint, uint64, uint64) {
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }
        uint len = listLength(chunk0);
        require(len != 0, ExchangeErrors.ValueNotFound()); //value not found
        unchecked{
            chunk0 -= 1; //decrement length
        }
        uint64 last;
        if (len <= 3) {
            last = uint64(chunk0 >> (len * 64));
            chunk0 = zero64bitSlot(chunk0, len * 64);
        } else {
            uint lastSlot = slot0 | uint(uint((len >> 2)) << 64);
            uint lastChunk;
            assembly {
                lastChunk := sload(lastSlot)
            }
            last = uint64(lastChunk >> (((len - 4) & 3)*64));
            lastChunk = zero64bitSlot(lastChunk, (((len - 4) & 3)*64));
            assembly {
                sstore(lastSlot, lastChunk)
            }
        }
        return (chunk0, last, uint64(len - 1));
    }
```

**Logic:**
1. **Decrement Length**: Reduce length by 1
2. **Find Last Element**:
   - **If len <= 3**: Last element is in chunk 0
   - **Otherwise**: Last element is in chunk `(len-1) >> 2`
3. **Extract Value**: Extract 64-bit value from appropriate position
4. **Zero Out**: Clear the position (set to 0)

**Helper Functions**:

```121:129:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function zero64bitSlot(uint v, uint shiftBits) internal pure returns (uint) {
        uint mask = ~(((1 << 64) - 1) << shiftBits);
        return v & mask;
    }

    function replace64bitValue(uint v, uint shiftBits, uint64 newValue) internal pure returns (uint) {
        uint mask = ~(((1 << 64) - 1) << shiftBits);
        return (v & mask) | (uint(newValue) << shiftBits);
    }
```

**Purpose**: Manipulate 64-bit values within `uint256` chunks

---

## List Value Removal

### Remove List Value (By Value)

```230:252:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function removeListValue(uint64 area, uint64 listId, uint64 value) internal returns (uint64) {
        uint slot0 = _asSlot0(area, listId);
        (uint chunk0, uint64 last, uint64 len) = privateRemoveLast(area, listId);
        bool done = last == value;
        if (!done) {
            for (uint i = 64; i < 256; i+=64) {
                uint64 cur = uint64(chunk0 >> i);
                if (cur == value) {
                    chunk0 = replace64bitValue(chunk0, i, last);
                    done = true;
                    break;
                }
            }
            if (!done && len > 3) {
                done = replaceDeepValue(slot0, len, value, last);
            }
        }
        require(done, ExchangeErrors.ValueNotFound()); //value not found
        assembly {
            sstore(slot0, chunk0)
        }
        return len;
    }
```

**Purpose**: Removes a specific value from the list (not just last element)

**Strategy**:
1. **Remove Last**: Pop last element
2. **Check Match**: If last == value, done
3. **Search Chunk 0**: Search remaining positions in chunk 0
4. **Search Other Chunks**: If not found, search deeper chunks
5. **Replace**: Replace found value with last element (maintains order)

**Usage**: Removing specific positions from perp/lending lists

### Replace List Value

```254:278:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function replaceListValue(uint64 area, uint64 listId, uint64 value, uint64 newValue) internal {
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }
        uint64 len = listLength(chunk0);
        require(len != 0, ExchangeErrors.ValueNotFound()); //value not found
        bool done = false;
        for (uint i = 64; i < 256; i+=64) {
            uint64 cur = uint64(chunk0 >> i);
            if (cur == value) {
                chunk0 = replace64bitValue(chunk0, i, newValue);
                assembly {
                    sstore(slot0, chunk0)
                }
                done = true;
                break;
            }
        }
        if (!done && len > 3) {
            done = replaceDeepValue(slot0, len, value, newValue);
        }
        require(done, ExchangeErrors.ValueNotFound()); //value not found
    }
```

**Purpose**: Replaces a specific value with a new value

**Logic**: Similar to `removeListValue()` but replaces instead of removing

---

## List Iteration

### LIFO Iterator (Last-In-First-Out)

```302:320:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function lifoLoop(uint64 area, uint64 listId) internal view returns (LifoIterator, uint64) {
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }
        uint32 listLen = uint32(listLength(chunk0));
        require(listLen > 0, ExchangeErrors.NoNextElement());
        if (listLen <= 3) {
            return LifoIteratorLib.fromChunk(chunk0 >> 64, listLen - 1, 0);
        }
        uint32 chunkNum = listLen >> 2;
        uint lastSlot = slot0 | (uint(chunkNum) << 64);
        uint lastChunk;
        assembly {
            lastChunk := sload(lastSlot)
        }
        return LifoIteratorLib.fromChunk(lastChunk, (listLen - 4) & 3, chunkNum);
    }
```

**Purpose**: Creates a LIFO iterator starting from the last element

**Usage**: Iterating perp positions in reverse order (LIFO netting)

**LifoIterator Structure**:

```402:442:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
// item2 (64) | item1 (64) | item0 (64) | empty (32) | chunkNum (32)
type LifoIterator is uint256;

// during iteration, next element is always in the upper bits of LifoIterator (aka, item2)
library LifoIteratorLib {
    function fromChunk(uint chunk, uint lastPos, uint32 _chunkNum) internal pure returns (LifoIterator, uint64) {
        uint64 v = uint64(chunk >> (64 * lastPos));
        unchecked {
            chunk = chunk << ((4 - lastPos) * 64);
        }
        return (LifoIterator.wrap(chunk | uint(_chunkNum)), v);
    }

    function chunkNum(LifoIterator _it) internal pure returns (uint32) {
        return uint32(LifoIterator.unwrap(_it));
    }

    function hasNext(LifoIterator _it) internal pure returns (bool) {
        return chunkNum(_it) > 0 || (LifoIterator.unwrap(_it) >> 192) != 0;
    }

    function loopNext(LifoIterator _it, uint64 area, uint64 listId) internal view returns (LifoIterator, uint64) {
        uint64 next = uint64(LifoIterator.unwrap(_it) >> 192);
        if (next != 0) {
            uint v = LifoIterator.unwrap(_it);
            return (LifoIterator.wrap(((v >> 64) << 128) | uint(uint32(v)) ), next);
        }
        uint32 _chunkNum = _it.chunkNum();
        require(_chunkNum > 0, ExchangeErrors.NoNextElement()); // no next element
        unchecked{
            _chunkNum -= 1;
        }
        uint chunk = StorageUtils64.readListChunk(area, listId, _chunkNum);
        uint lastPos = 3;
        if (_chunkNum == 0) {
            lastPos = 2;
            chunk = chunk >> 64;
        }
        return LifoIteratorLib.fromChunk(chunk, lastPos, _chunkNum);
    }
}
```

**LifoIterator Bit Layout:**
- **Bits 0-31**: `chunkNum` (32 bits)
- **Bits 32-63**: Reserved (32 bits)
- **Bits 64-127**: `item0` (64 bits)
- **Bits 128-191**: `item1` (64 bits)
- **Bits 192-255**: `item2` (64 bits, current element)

**Iteration Flow:**
1. **Start**: Begin from last chunk, last position
2. **Next**: Move backward through chunks
3. **Chunk 0**: Special handling (only 3 values, skip length)

### FIFO Iterator (Forward Iteration)

```322:351:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
    function loopStart(uint64 area, uint64 listId) internal view returns (ListIterator) {
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }
        return ListIteratorLib.fromChunk0(chunk0);
    }

    function loopStartFrom(uint64 area, uint64 listId, uint32 start) internal view returns (ListIterator, uint64, uint32) {
        uint slot0 = _asSlot0(area, listId);
        uint chunk0;
        assembly {
            chunk0 := sload(slot0)
        }
        uint32 listLen = uint32(listLength(chunk0));
        if (listLen == 0) {
            return (ListIterator.wrap(0), 0, 0);
        }
        require(start < listLen, ExchangeErrors.StartTooLarge()); //start too large
        if (start < 3) {
            uint64 first = uint64(chunk0 >> 64*(start + 1));
            chunk0 = ((chunk0 >> (64*(start+2))) << 64) | (uint(listLen));
            return (ListIteratorLib.fromChunk0(chunk0), first, listLen);
        }
        uint32 chunkNum = (start + 1) >> 2;
        uint chunk = (readListChunk(area, listId, chunkNum)) >> (((start - 3) & 3)*64);
        (ListIterator it, uint64 firstItem) = ListIteratorLib.fromChunk(chunk, (listLen >> 2) - chunkNum, chunkNum);
        return (it, firstItem, listLen);
    }
```

**Purpose**: Creates forward iterators for FIFO traversal

**ListIterator Structure**:

```354:392:BaseDEX/contracts/src/main/sol/StorageUtils64.sol
// item2 (64) | item1 (64) | item0 (64) | remainChunks (32) | chunkNum (32)
type ListIterator is uint256;

library ListIteratorLib {
    function fromChunk0(uint chunk0) internal pure returns (ListIterator) {
        // the LSB 32 bits, chunkNum, is zero
        return ListIterator.wrap(((chunk0 >> 64) << 64) | (uint(StorageUtils64.listLength(chunk0) >> 2) << 32));
    }

    function fromChunk(uint chunk, uint32 _remainChunks, uint32 _chunkNum) internal pure returns (ListIterator, uint64) {
        return (ListIterator.wrap(((chunk >> 64) << 64) | (uint(_remainChunks) << 32) | uint(_chunkNum)), uint64(chunk));
    }

    function chunkNum(ListIterator _it) internal pure returns (uint32) {
        return uint32(ListIterator.unwrap(_it));
    }

    function remainChunks(ListIterator _it) internal pure returns (uint32) {
        return uint32(ListIterator.unwrap(_it) >> 32);
    }

    function hasNext(ListIterator _it) internal pure returns (bool) {
        return remainChunks(_it) > 0 || (ListIterator.unwrap(_it) >> 64) != 0;
    }

    function loopNext(ListIterator _it, uint64 area, uint64 listId) internal view returns (ListIterator, uint64) {
        uint64 next = uint64(ListIterator.unwrap(_it) >> 64);
        if (next != 0) {
            return (ListIterator.wrap(((ListIterator.unwrap(_it) >> 128) << 64) | uint(uint64(ListIterator.unwrap(_it)))), next);
        }
        uint32 _chunkNum = _it.chunkNum();
        uint32 _remainChunks = _it.remainChunks();
        require(_remainChunks > 0, ExchangeErrors.NoNextElement()); // no next element
        _chunkNum += 1;
        _remainChunks -= 1;
        uint chunk = StorageUtils64.readListChunk(area, listId, _chunkNum);
        return ListIteratorLib.fromChunk(chunk, _remainChunks, _chunkNum);
    }
}
```

**ListIterator Bit Layout:**
- **Bits 0-31**: `chunkNum` (32 bits)
- **Bits 32-63**: `remainChunks` (32 bits)
- **Bits 64-127**: `item0` (64 bits)
- **Bits 128-191**: `item1` (64 bits)
- **Bits 192-255**: `item2` (64 bits)

**Iteration Flow:**
1. **Start**: Begin from chunk 0, position 0
2. **Next**: Move forward through chunks
3. **Track Progress**: `remainChunks` tracks remaining chunks

---

## Usage Examples

### Perp Position Lists

```352:365:BaseDEX/contracts/src/main/sol/CompositeExchangeStorage.sol
        (LifoIterator it, uint64 last) = StorageUtils64.lifoLoop(area, tokenId);
        bool listIsShort = (last & SHORT_COUNTER_PARTY) == 0;
        last = last & ~SHORT_COUNTER_PARTY;
        PerpMatch perpMatch = listIsShort ?  readPerpPosition(last, listUserId, tokenId) : readPerpPosition(listUserId, last, tokenId);
        while (perpMatch.quantity() == 0) {
            StorageUtils64.removeLast(area, tokenId);
            if (listIsShort) {
                perpMatch = perpMatch.removeShort();
            } else {
                perpMatch = perpMatch.removeLongAndZero();
            }
            listIsShort ? writePerpPosition(last, listUserId, tokenId, perpMatch) : writePerpPosition(listUserId, last, tokenId, perpMatch);
            if (it.hasNext()) {
                (it, last) = it.loopNext(area, tokenId);
```

**Usage**: Iterating perp positions in LIFO order for netting

**Flow:**
1. **Start LIFO Loop**: Get iterator from last position
2. **Iterate Backward**: Process positions from newest to oldest
3. **Net Positions**: Combine matching positions
4. **Clean Zeros**: Remove zero-quantity positions

### Lending Position Lists

Lending positions use similar list operations for managing borrower/lender positions.

---

## Comparison: StorageUtils vs StorageUtils64

| Feature | StorageUtils | StorageUtils64 |
|--------|--------------|----------------|
| **Area Type** | `uint32` (32 bits) | `uint64` (64 bits) |
| **Max Areas** | ~4.3 billion | ~18.4 quintillion |
| **Slot Formula** | `(pos << 32) \| area` | `(pos << 64) \| area` |
| **List Support** | No | Yes (LIFO/FIFO) |
| **Used By** | Orderbooks | Main exchange |
| **List Storage** | N/A | Chunk-based, 64-bit values |
| **Iterators** | N/A | `LifoIterator`, `ListIterator` |

---

## Storage Efficiency

### Chunk-Based Storage

**Advantages:**
- **Packed Values**: 4 values per `uint256` slot (64 bits each)
- **Gas Efficient**: Fewer storage reads/writes
- **Batch Operations**: Can manipulate multiple values in one slot

**Storage Layout Example** (List with 7 elements):
```
Chunk 0: [len=7] [val3] [val2] [val1]
Chunk 1: [val7] [val6] [val5] [val4]
```

**Gas Savings**: Reading 4 values costs 1 `sload` instead of 4

---

## List Operations Summary

| Operation | Function | Complexity | Usage |
|-----------|----------|------------|-------|
| **Append** | `appendList()` | O(1) | Add to end |
| **Remove Last** | `removeLast()` | O(1) | LIFO pop |
| **Remove Value** | `removeListValue()` | O(N) | Remove specific value |
| **Replace Value** | `replaceListValue()` | O(N) | Update specific value |
| **LIFO Iterate** | `lifoLoop()` | O(1) per step | Iterate backward |
| **FIFO Iterate** | `loopStart()` | O(1) per step | Iterate forward |
| **Check Empty** | `isListEmpty()` | O(1) | Quick empty check |

---

## Summary

`StorageUtils64.sol` provides:
- **64-bit Area Partitioning**: Larger namespace than `StorageUtils`
- **List Operations**: Chunk-based storage for efficient list management
- **LIFO Iteration**: `LifoIterator` for backward traversal
- **FIFO Iteration**: `ListIterator` for forward traversal
- **Gas Optimization**: Packed storage (4 values per slot)
- **Position Management**: Used for perp and lending positions

This enables efficient position management in BaseDEX, allowing the exchange to maintain and iterate through user positions while minimizing gas costs through optimized storage patterns.