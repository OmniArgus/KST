Searching for `TempStorageUtils64.sol`.


[2 tools called]

Reading `TempStorageUtils64.sol` and checking its usage:


[3 tools called]

Reviewing `PriceHelper.sol` to understand usage:


[2 tools called]

Reviewing more of `PriceHelper.sol` to see how temporary storage is used:


[1 tool called]

## **TempStorageUtils64.sol - Explanation**

### **Purpose and importance**

`TempStorageUtils64` is a library for transient storage in Solidity. It uses EVM opcodes `tstore` and `tload` (EIP-1153) to store temporary data that persists only during a transaction and is cleared at the end. Used in `PriceHelper` to simulate state changes during price estimation without affecting the exchange.

### **Core concept: transient storage**

Transient storage (`tstore`/`tload`):
- Lives only during the transaction
- Cleared automatically at the end
- Cheaper than regular storage (`sstore`/`sload`)
- Useful for temporary state simulation

### **Difference from StorageUtils64**

| Feature | `StorageUtils64` | `TempStorageUtils64` |
|---------|------------------|----------------------|
| **Opcode** | `sload`/`sstore` | `tload`/`tstore` |
| **Persistence** | Permanent (survives transaction) | Temporary (cleared after transaction) |
| **Gas Cost** | Higher (20,000 gas for write) | Lower (~100 gas for write) |
| **Use Case** | Real state changes | Simulation/testing |

### **Implementation details**

#### **1. Basic read/write functions**

```9:23:BaseDEX/contracts/src/main/sol/TempStorageUtils64.sol
    function readStorage(uint64 area, uint64 pos) internal view returns (uint) {
        uint slot = (uint(pos) << 64) | uint(area);
        uint result;
        assembly {
            result := tload(slot)
        }
        return result;
    }

    function writeStorage(uint64 area, uint64 pos, uint value) internal {
        uint slot = (uint(pos) << 64) | uint(area);
        assembly {
            tstore(slot, value)
        }
    }
```

- Slot calculation: `slot = (pos << 64) | area` (same as `StorageUtils64`)
- Uses `tload`/`tstore` instead of `sload`/`sstore`

#### **2. Support for 128-bit positions**

```25:39:BaseDEX/contracts/src/main/sol/TempStorageUtils64.sol
    function readStorage128(uint64 area, uint128 pos) internal view returns (uint) {
        uint slot = (uint(pos) << 64) | uint(area);
        uint result;
        assembly {
            result := tload(slot)
        }
        return result;
    }

    function writeStorage128(uint64 area, uint128 pos, uint value) internal {
        uint slot = (uint(pos) << 64) | uint(area);
        assembly {
            tstore(slot, value)
        }
    }
```

- Same slot formula, supporting 128-bit positions

#### **3. Address-based storage**

```41:55:BaseDEX/contracts/src/main/sol/TempStorageUtils64.sol
    function readStorageForAddress(uint64 area, address adr) internal view returns (uint) {
        uint slot = (uint(uint160(adr)) << 64) | uint(area);
        uint result;
        assembly {
            result := tload(slot)
        }
        return result;
    }

    function writeStorageForAddress(uint64 area, address adr, uint value) internal {
        uint slot = (uint(uint160(adr)) << 64) | uint(area);
        assembly {
            tstore(slot, value)
        }
    }
```

- Computes slots from addresses

### **Primary usage: PriceHelper**

`PriceHelper` uses temporary storage to simulate ledger state during price estimation:

#### **1. Storing exchange address**

```20:30:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function internalStoreExchange(address exch) internal {
        TempStorageUtils64.writeStorage(AREA_PARAMS, PARAM_EXCHANGE, uint160(exch));
    }

    function internalReadExchangeAsRO() internal view returns (ICompositeExchangeReadOnly) {
        return ICompositeExchangeReadOnly(address(uint160(TempStorageUtils64.readStorage(AREA_PARAMS, PARAM_EXCHANGE))));
    }

    function internalReadExchange() internal view returns (ICompExchForBooks) {
        return ICompExchForBooks(address(uint160(TempStorageUtils64.readStorage(AREA_PARAMS, PARAM_EXCHANGE))));
    }
```

- Temporarily stores the exchange address for the transaction

#### **2. Simulating ledger state**

```92:108:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function internalReadAreaLedger() internal view returns (uint64) {
        return uint64(TempStorageUtils64.readStorage(AREA_PARAMS, LEDGER_COUNT)) + AREA_LEDGER;
    }

    function internalReadLedger(uint64 userId, uint32 tokenId) internal view returns(VaultLedger) {
        uint128 key = (uint128(userId) << 32) | uint128(tokenId);
        uint256 tempLedger = TempStorageUtils64.readStorage128(internalReadAreaLedger(), key);
        if (tempLedger == 0) {
            (tempLedger,, ) = internalReadExchange().getBalance(userId, tokenId);
        }
        return VaultLedgerLib.newVaultLedger(uint128(tempLedger), 1, 1);
    }

    function writeLedger(uint64 userId, uint32 tokenId, VaultLedger ledger) internal {
        uint128 key = (uint128(userId) << 32) | uint128(tokenId);
        return TempStorageUtils64.writeStorage128(internalReadAreaLedger(), key, ledger.raw());
    }
```

Simulation pattern:
1. Check temporary storage first
2. If missing, read from the exchange
3. Write simulated changes to temporary storage
4. Later reads use the simulated state

#### **3. Transaction isolation**

```304:308:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function estimatePrices(address exchangeAddress, uint256[] calldata batch) external returns (uint256[] memory) {
        uint called = TempStorageUtils64.readStorage(AREA_PARAMS, REENTRANCY_GUARD);
        require(called == 0, "Reentrancy not supported");
        TempStorageUtils64.writeStorage(AREA_PARAMS, REENTRANCY_GUARD, 1);
```

- Reentrancy guard stored in temporary storage; cleared at transaction end

#### **4. Cleanup**

```336:340:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function clear() external {
        TempStorageUtils64.writeStorage(AREA_PARAMS, REENTRANCY_GUARD, 0);
        uint ledgerCount = TempStorageUtils64.readStorage(AREA_PARAMS, LEDGER_COUNT);
        TempStorageUtils64.writeStorage(AREA_PARAMS, LEDGER_COUNT, ledgerCount + 1);
    }
```

- While temporary storage clears automatically, `clear()` can reset guards/counters if needed

### **Benefits**

1. State simulation: Estimate prices without modifying exchange state
2. Gas savings: `tstore`/`tload` are cheaper than `sstore`/`sload`
3. Transaction isolation: Data cleared automatically at transaction end
4. Reentrancy protection: Guards stored in temporary storage
5. Safe testing: Can simulate complex operations without side effects

### **Use cases**

1. Price estimation: Simulate trades to compute expected outcomes
2. Fee calculation: Estimate fees without changing balances
3. Market order simulation: Predict execution prices before submitting
4. Balance checks: Simulate transfers to check if trades would succeed

### **Technical details**

#### **Slot calculation formula**

```solidity
slot = (position << 64) | area
```

- `position`: 64-bit or 128-bit position identifier
- `area`: 64-bit area code for partitioning
- Result: Unique 256-bit storage slot

#### **Assembly implementation**

Uses inline assembly:
```solidity
assembly {
    result := tload(slot)  // Load from transient storage
    tstore(slot, value)    // Store to transient storage
}
```

### **Relationship to other contracts**

- `StorageUtils64`: Permanent storage equivalent
- `PriceHelper`: Primary consumer, simulates ledger state
- `CompositeExchange`: Exchange contract being simulated
- `SwapRouter`: Uses `PriceHelper` for price estimation

### **Limitations**

1. Transaction-scoped: Data does not persist between transactions
2. Single transaction: Cannot share data across multiple transactions
3. EIP-1153 required: Requires a network that supports these opcodes

### **Summary**

`TempStorageUtils64` provides temporary storage using `tstore`/`tload` for simulating state changes during a transaction. It enables `PriceHelper` to estimate prices and simulate trades without modifying the exchange, with lower gas costs and automatic cleanup, making it well-suited for read-only price estimation and simulation.