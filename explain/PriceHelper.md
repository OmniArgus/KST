Examining `PriceHelper.sol` to understand its purpose and implementation.


[4 tools called]

## PriceHelper.sol - Explanation

`PriceHelper.sol` is a helper contract used by orderbooks for fee estimation, market order price estimation, and temporary settlement simulation. It implements `IPriceHelperForBooks` (extends `ICompExchForBooks`) and uses temporary storage to simulate state changes without modifying the exchange.

---

## Purpose and Architecture

```10:10:BaseDEX/contracts/src/main/sol/PriceHelper.sol
contract PriceHelper is IPriceHelperForBooks {
```

**Key Characteristics:**
- **Implements `IPriceHelperForBooks`**: Extends `ICompExchForBooks` with fee estimation
- **Temporary Storage**: Uses `TempStorageUtils64` for simulation
- **Used by Orderbooks**: Called by `TwoTokenOrderBook` for market order estimation
- **Settlement Simulation**: Provides settlement functions for orderbook testing
- **Reentrancy Protection**: Prevents multiple calls in the same transaction

**Why Temporary Storage?**
- Orderbooks need to simulate settlements without modifying exchange state
- Market order estimation requires fee calculations
- Balance checks need to account for fees and conversions

---

## Temporary Storage Management

### Storage Structure

```13:18:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    uint64 constant AREA_PARAMS = 0;
    uint64 constant PARAM_EXCHANGE = 3;
    uint64 constant AREA_LEDGER = 30;
    uint64 constant USER_OPS = 1;
    uint64 constant REENTRANCY_GUARD = 1000000;
    uint64 constant LEDGER_COUNT = 1000001;
```

**Storage Areas:**
- `AREA_PARAMS`: Configuration (exchange address, reentrancy guard, ledger count)
- `AREA_LEDGER`: Temporary ledger state (30 + ledgerCount offset)

**Key Slots:**
- `PARAM_EXCHANGE`: Exchange contract address
- `REENTRANCY_GUARD`: Prevents multiple calls in same transaction
- `LEDGER_COUNT`: Tracks ledger area iteration

### Exchange Address Management

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

**Purpose**: Stores and retrieves exchange address for API calls.

**Usage**: Called by `estimatePrices()` to initialize exchange address.

---

## Fee Rate Calculation

### Get Fee Rate

```36:40:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function getFeeRate(OrderBookConfig config, uint64 userId, bool isMaker, bool isLiquidation) internal pure returns (uint16) {
        if (!isMaker && isLiquidation) return 0;
        uint16 feeRate = isMaker ? config.makerFeeBip() : config.takerFeeBip();
        return adjustFeeRateForUser(feeRate, userId);
    }
```

**Logic:**
- **Taker liquidation**: 0% fee (taker is liquidator)
- **Maker**: Uses `makerFeeBip()`
- **Taker**: Uses `takerFeeBip()`
- **User adjustment**: May adjust fee rate for specific users

### Fee Rate Adjustment

```42:48:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function adjustFeeRateForUser(uint16 feeRate, uint64 userId) internal pure returns(uint16) {
        if (userId == USER_OPS) {
            return 0;
        }
        //todo: how do we make sure this is the same as the exchange?
        return feeRate;
    }
```

**Special Cases:**
- `USER_OPS`: Zero fee (operations account)
- **TODO**: Should match exchange's user fee schedule

### Maker Fee Calculation

```50:68:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function makerFee(uint64 fullQuantity, uint64 matchQuantity, uint16 feeRate) internal pure returns(uint64) {
        unchecked{
            if (feeRate == 0) {
                return 0;
            }
            uint64 makerTotalFee = uint64(uint(fullQuantity) * uint(feeRate) / ConfigConstants.FEE_DIVISOR);
            if (makerTotalFee == 0) {
                makerTotalFee = 1;
            }
            if (fullQuantity == matchQuantity) {
                return makerTotalFee;
            }
            uint64 leftOverFee = makerTotalFee - uint64(uint(matchQuantity) * uint(feeRate) / ConfigConstants.FEE_DIVISOR);
            if (leftOverFee == 0) {
                leftOverFee = 1;
            }
            return makerTotalFee - leftOverFee;
        }
    }
```

**Purpose**: Calculates maker fee for partial fills.

**Logic:**
- **Total Fee**: `fullQuantity × feeRate / FEE_DIVISOR`
- **Partial Fill**: Fee is proportional to `matchQuantity / fullQuantity`
- **Minimum Fee**: Ensures at least 1 unit if `feeRate > 0`

**Why Minimum Fee?**
- Prevents zero fees due to rounding
- Ensures fee collection for small orders

---

## Fee Estimation for Market Orders

### Estimate Fee From Quantity

```342:350:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function estimateFeeFromQuantity(uint64 quantity, uint64 feeSoFar) external view returns (uint64) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        uint16 feeRate = orderBookConfig.takerFeeBip();
        uint64 fee = uint64(uint(quantity) * uint(feeRate) / (ConfigConstants.FEE_DIVISOR - uint(feeRate)));
        if (fee + feeSoFar > orderBookConfig.fromMaxFee()) {
            fee = orderBookConfig.fromMaxFee() - feeSoFar;
        }
        return fee;
    }
```

**Purpose**: Estimates taker fee for market orders specifying exact output.

**Formula**: `fee = quantity × feeRate / (FEE_DIVISOR - feeRate)`

**Why this formula?**
- Market orders specify exact output (`amountOut`)
- Fee is deducted from input, so: `amountIn = amountOut + fee`
- Solving: `fee = amountOut × feeRate / (1 - feeRate/FEE_DIVISOR)`

**Usage**: Called by `TwoTokenOrderBook.computeBuyMarketOrderSpecifyOut()` to estimate required input.

**Example Usage**:
```341:348:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
                    IPriceHelperForBooks priceHelper = IPriceHelperForBooks(msg.sender);
                    uint64 estimatedFees = priceHelper.estimateFeeFromQuantity(orderData.amountOut(), 0);
                    uint64 buyQuantity = orderData.amountOut() + estimatedFees;
```

---

## Market Order Price Estimation

### Estimate Prices

```304:334:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function estimatePrices(address exchangeAddress, uint256[] calldata batch) external returns (uint256[] memory) {
        uint called = TempStorageUtils64.readStorage(AREA_PARAMS, REENTRANCY_GUARD);
        require(called == 0, "Reentrancy not supported");
        TempStorageUtils64.writeStorage(AREA_PARAMS, REENTRANCY_GUARD, 1);
        internalStoreExchange(exchangeAddress);
        uint[] memory result = new uint[](batch.length/2);
        require(batch.length & 1 == 0);
        for(uint i = 0; i < batch.length; i += 2) {
            PriceAddressSpec pas = PriceAddressSpec.wrap(batch[i]);
            MarketOrderPrice mop = MarketOrderPrice.wrap(batch[i + 1]);
            if (pas.priceType() == PRICE_TYPE_SELL_IN) {
                IOrderBookAbstract obs = IOrderBookAbstract(pas.orderbook());
                result[i/2] = obs.computeSellMarketOrderSpecifyIn(mop).raw();
            }
            else if (pas.priceType() == PRICE_TYPE_BUY_OUT) {
                IOrderBookAbstract obs = IOrderBookAbstract(pas.orderbook());
                result[i/2] = obs.computeBuyMarketOrderSpecifyOut(mop).raw();
            }
            else if (pas.priceType() == PRICE_TYPE_SELL_OUT) {
                IOrderBookAbstract obs = IOrderBookAbstract(pas.orderbook());
                result[i/2] = obs.computeSellMarketOrderSpecifyOut(mop).raw();
            }
            else if (pas.priceType() == PRICE_TYPE_BUY_IN) {
                IOrderBookAbstract obs = IOrderBookAbstract(pas.orderbook());
                result[i/2] = obs.computeBuyMarketOrderSpecifyIn(mop).raw();
            } else {
                revert WrongPriceType();
            }
        }
        return result;
    }
```

**Purpose**: Estimates execution prices for multiple market orders in batch.

**Input Format**:
- Alternating: `PriceAddressSpec`, `MarketOrderPrice`
- `PriceAddressSpec`: Orderbook address + price type
- `MarketOrderPrice`: Order parameters (amount, limit price, liquidation flag)

**Price Types**:
- `PRICE_TYPE_SELL_IN`: Sell market order specifying input amount
- `PRICE_TYPE_BUY_OUT`: Buy market order specifying output amount
- `PRICE_TYPE_SELL_OUT`: Sell market order specifying output amount
- `PRICE_TYPE_BUY_IN`: Buy market order specifying input amount

**Output**: Array of `MarketOrderPriceResult` with estimated execution prices and quantities.

**Reentrancy Protection**: Prevents multiple calls in same transaction.

**Usage**: Called by `SwapRouter` to estimate swap prices before execution.

### Clear Function

```336:340:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function clear() external {
        TempStorageUtils64.writeStorage(AREA_PARAMS, REENTRANCY_GUARD, 0);
        uint ledgerCount = TempStorageUtils64.readStorage(AREA_PARAMS, LEDGER_COUNT);
        TempStorageUtils64.writeStorage(AREA_PARAMS, LEDGER_COUNT, ledgerCount + 1);
    }
```

**Purpose**: Clears reentrancy guard and advances ledger area.

**Usage**: Called after `estimatePrices()` completes to reset state.

---

## Temporary Ledger Management

### Read Ledger

```96:103:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function internalReadLedger(uint64 userId, uint32 tokenId) internal view returns(VaultLedger) {
        uint128 key = (uint128(userId) << 32) | uint128(tokenId);
        uint256 tempLedger = TempStorageUtils64.readStorage128(internalReadAreaLedger(), key);
        if (tempLedger == 0) {
            (tempLedger,, ) = internalReadExchange().getBalance(userId, tokenId);
        }
        return VaultLedgerLib.newVaultLedger(uint128(tempLedger), 1, 1);
    }
```

**Purpose**: Reads ledger from temporary storage or exchange.

**Logic:**
- **Check Temp Storage**: If exists, use cached value
- **Fallback**: Read from exchange and cache
- **Constructor**: Creates `VaultLedger` with temporary values for seq balances

**Why Caching?**
- Reduces exchange calls during simulation
- Enables temporary state modifications

### Write Ledger

```105:108:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function writeLedger(uint64 userId, uint32 tokenId, VaultLedger ledger) internal {
        uint128 key = (uint128(userId) << 32) | uint128(tokenId);
        return TempStorageUtils64.writeStorage128(internalReadAreaLedger(), key, ledger.raw());
    }
```

**Purpose**: Writes ledger to temporary storage.

**Usage**: Used by transfer functions to simulate state changes.

### Transfer Function

```110:125:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function transfer(uint32 token, uint128 amount, uint64 from, uint64 to) internal {
        if (amount == 0 || from == to) {
            return;
        }
        if (from != 0) {
            VaultLedger fromLedger = internalReadLedger(from, token);
            require(fromLedger.userBalance() >= amount, ExchangeErrors.BalanceTooLow()); //balance too low
            fromLedger = fromLedger.decrementBalance(amount);
            writeLedger(from, token, fromLedger);
        }
        if (to != 0) {
            VaultLedger toLedger = internalReadLedger(to, token);
            toLedger = toLedger.incrementBalance(amount);
            writeLedger(to, token, toLedger);
        }
    }
```

**Purpose**: Simulates token transfer in temporary storage.

**Features:**
- **Balance Check**: Ensures sender has sufficient balance
- **Dual Updates**: Updates both sender and receiver ledgers
- **Zero Address Handling**: Allows transfers to/from zero address (for fees)

---

## Spot Trade Settlement

### Settle Spot Maker Buyer

```134:153:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function settleSpotMakerBuyer(MakerSpotMatch makerSpotMatch, TakerSpotMatch takerSpotMatch) external returns (SpotMatchQuantities) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        uint64 toQuantity = makerSpotMatch.makerPrice().convertFromToTo(takerSpotMatch.matchQuantity(), orderBookConfig.ftpdDiff());
        uint16 buyerFeeRate = getFeeRate(orderBookConfig, makerSpotMatch.makerId(), true, false);
        uint64 buyerFee = makerFee(makerSpotMatch.makerQuantity(), takerSpotMatch.matchQuantity(), buyerFeeRate);
        if (orderBookConfig.fromMaxFee() != 0 && orderBookConfig.fromMaxFee() < buyerFee) {
            buyerFee = orderBookConfig.fromMaxFee();
        }
        uint16 sellerFeeRate = getFeeRate(orderBookConfig, takerSpotMatch.takerId(), false, makerSpotMatch.isLiquidation());
        uint64 sellerFee = uint64(uint(toQuantity) * uint(sellerFeeRate) / ConfigConstants.FEE_DIVISOR);
        if (sellerFee == 0 && sellerFeeRate != 0) {
            sellerFee = 1;
        }
        uint64 totalSellerFee = takerSpotMatch.feeSoFar() + sellerFee;
        if (orderBookConfig.toMaxFee() != 0 && orderBookConfig.toMaxFee() < totalSellerFee) {
            sellerFee = orderBookConfig.toMaxFee() - takerSpotMatch.feeSoFar();
        }
        transferForSpotMatch(orderBookConfig, takerSpotMatch.matchQuantity(), toQuantity, makerSpotMatch.makerId(), takerSpotMatch.takerId(), buyerFee, sellerFee);
        return SpotMatchQuantitiesLib.newSpotMatchQuantities(toQuantity - sellerFee, takerSpotMatch.matchQuantity() - buyerFee, sellerFee, buyerFee);
    }
```

**Purpose**: Simulates spot trade settlement when maker is buyer.

**Flow:**
1. **Calculate Output**: Convert seller's quantity to buyer's token
2. **Calculate Fees**: Maker fee (proportional) and taker fee (on output)
3. **Apply Fee Caps**: Respect `fromMaxFee` and `toMaxFee`
4. **Transfer Tokens**: Simulate transfers in temp storage
5. **Return Quantities**: Net quantities after fees

**Fee Calculation:**
- **Buyer Fee**: Maker fee on `matchQuantity` (proportional)
- **Seller Fee**: Taker fee on `toQuantity` (accumulated if multiple matches)

### Settle Spot Maker Seller

```155:178:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function settleSpotMakerSeller(MakerSpotMatch makerSpotMatch, TakerSpotMatch takerSpotMatch) external returns (SpotMatchQuantities) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        uint64 fullToQuantity = makerSpotMatch.makerPrice().convertFromToTo(makerSpotMatch.makerQuantity(), orderBookConfig.ftpdDiff());
        uint64 toQuantity = fullToQuantity - makerSpotMatch.makerPrice().convertFromToTo(makerSpotMatch.makerQuantity() - takerSpotMatch.matchQuantity(), orderBookConfig.ftpdDiff());
        if (toQuantity == 0) {
            toQuantity = 1;
        }
        uint16 sellerFeeRate = getFeeRate(orderBookConfig, makerSpotMatch.makerId(), true, false);
        uint64 sellerFee = makerFee(fullToQuantity, toQuantity, sellerFeeRate);
        if (orderBookConfig.toMaxFee() !=0 && orderBookConfig.toMaxFee() < sellerFee) {
            sellerFee = orderBookConfig.toMaxFee();
        }
        uint16 buyerFeeRate = getFeeRate(orderBookConfig, takerSpotMatch.takerId(), false, makerSpotMatch.isLiquidation());
        uint64 buyerFee = uint64(uint(takerSpotMatch.matchQuantity()) * uint(buyerFeeRate) / ConfigConstants.FEE_DIVISOR);
        if (buyerFee == 0 && buyerFeeRate != 0) {
            buyerFee = 1;
        }
        uint64 totalBuyerFee = takerSpotMatch.feeSoFar() + buyerFee;
        if (orderBookConfig.fromMaxFee() != 0 && orderBookConfig.fromMaxFee() < totalBuyerFee) {
            buyerFee = orderBookConfig.fromMaxFee() - takerSpotMatch.feeSoFar();
        }
        transferForSpotMatch(orderBookConfig, takerSpotMatch.matchQuantity(), toQuantity, takerSpotMatch.takerId(), makerSpotMatch.makerId(), buyerFee, sellerFee);
        return SpotMatchQuantitiesLib.newSpotMatchQuantities(toQuantity - sellerFee, takerSpotMatch.matchQuantity() - buyerFee, sellerFee, buyerFee);
    }
```

**Purpose**: Simulates spot trade settlement when maker is seller.

**Key Difference**: Calculates `toQuantity` as difference between full order and remaining order.

**Transfer For Spot Match**

```127:132:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function transferForSpotMatch(OrderBookConfig orderBookConfig, uint64 fromQuantity, uint64 toQuantity, uint64 buyerId, uint64 sellerId, uint64 buyerFee, uint64 sellerFee) internal {
        transfer(orderBookConfig.fromTokenId(), uint128(uint256(fromQuantity - buyerFee) * 10**(orderBookConfig.fromVaultMinusPositionDecimals())), sellerId, buyerId);
        transfer(orderBookConfig.fromTokenId(), uint128(uint256(buyerFee) * 10**(orderBookConfig.fromVaultMinusPositionDecimals())), sellerId, USER_OPS);
        transfer(orderBookConfig.toTokenId(), uint128(uint256(toQuantity - sellerFee) * 10**(orderBookConfig.toVaultMinusPositionDecimals())), buyerId, sellerId);
        transfer(orderBookConfig.toTokenId(), uint128(uint256(sellerFee) * 10**(orderBookConfig.toVaultMinusPositionDecimals())), buyerId, USER_OPS);
    }
```

**Purpose**: Simulates all transfers for a spot match.

**Transfers:**
1. **Principal from**: Seller → Buyer (minus buyer fee)
2. **Buyer fee**: Seller → USER_OPS
3. **Principal to**: Buyer → Seller (minus seller fee)
4. **Seller fee**: Buyer → USER_OPS

---

## Perp Trade Settlement

### Settle Perp Maker Buyer

```193:198:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function settlePerpMakerBuyer(MakerSpotMatch makerSpotMatch, TakerSpotMatch takerSpotMatch) external returns (SpotMatchQuantities) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        uint64 sellerFee = chargePerpFee(orderBookConfig, makerSpotMatch, takerSpotMatch, false);
        uint64 buyerFee = chargePerpFee(orderBookConfig, makerSpotMatch, takerSpotMatch, true);
        return SpotMatchQuantitiesLib.newSpotMatchQuantities(0, takerSpotMatch.matchQuantity(), sellerFee, buyerFee);
    }
```

**Purpose**: Simulates perp trade settlement when maker is buyer.

**Differences from Spot:**
- **No Token Transfer**: Perps don't transfer underlying tokens
- **Fee Only**: Charges fees to both parties
- **Position Update**: Position updates handled by exchange (not simulated here)

### Charge Perp Fee

```180:191:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function chargePerpFee(OrderBookConfig config, MakerSpotMatch makerSpotMatch, TakerSpotMatch takerSpotMatch, bool maker) internal returns (uint64) {
        uint64 clientId = maker ? makerSpotMatch.makerId() : takerSpotMatch.takerId();
        uint16 feeRate = getFeeRate(config, clientId, maker, makerSpotMatch.isLiquidation());
        if (feeRate == 0) {
            return 0;
        }
        uint64 fromFeeQuantity = uint64(uint(takerSpotMatch.matchQuantity()) * uint(feeRate) / ConfigConstants.FEE_DIVISOR);
        uint64 toFeeQuantity = makerSpotMatch.makerPrice().convertFromToTo(fromFeeQuantity, config.ftpdDiff());
        transfer(config.toTokenId(), uint128(uint256(toFeeQuantity) * 10**(config.toVaultMinusPositionDecimals())), clientId, USER_OPS);

        return toFeeQuantity;
    }
```

**Purpose**: Charges perp fee and transfers to USER_OPS.

**Logic:**
- **Calculate Fee**: `fromFeeQuantity × feeRate / FEE_DIVISOR`
- **Convert to Base Token**: Convert fee to base token (USDC)
- **Transfer**: Transfer fee to USER_OPS

---

## Perp Balance Calculations

### Compute Perp Bal Fees Only

```223:256:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function computePerpBalFeesOnly(SpotNode node, uint64 minOrderQuantity) external view returns(uint64) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        // fees for the transaction, maker:
        uint16 makerFeeRate = getFeeRate(orderBookConfig, node.client(), true, false);
        if (makerFeeRate == 0) {
            return node.quantity();
        }
        uint64 fromFeeQuantity = uint64(node.quantity() * (uint(makerFeeRate)) / ConfigConstants.FEE_DIVISOR);
        uint64 toFeeQuantity = node.price().convertFromToTo(fromFeeQuantity, orderBookConfig.ftpdDiff());
        if (orderBookConfig.toMaxFee() !=0 && toFeeQuantity > orderBookConfig.toMaxFee()) {
            toFeeQuantity = orderBookConfig.toMaxFee();
        }
        VaultTokenConfig baseConfig = internalReadTokenConfig(orderBookConfig.toTokenId());
        VaultLedger ledger = internalReadLedger(node.client(), orderBookConfig.toTokenId());
        uint64 bal64 = uint64(ledger.userBalance() / 10** (baseConfig.vaultDecimals() - baseConfig.positionDecimals()));
        if (bal64 <= toFeeQuantity) {
            return 0; // can't even pay the fees. Just cancel
        }
        bal64 -= toFeeQuantity;
        VaultTokenConfig tokenConfig = internalReadTokenConfig(orderBookConfig.fromTokenId());
        uint64 toQuantityAsMaker = node.price().convertFromToTo(node.quantity(), orderBookConfig.ftpdDiff());
        uint64 owed = uint64(tokenConfig.perpSequesterAmount(toQuantityAsMaker));

        if (bal64 >= owed) {
            return node.quantity();
        }
        unchecked {
            uint64 availQuantity = uint64(uint256(bal64) * uint256(node.quantity()) / uint256(owed));
            if (availQuantity > node.quantity()) {
                availQuantity = node.quantity();
            }
            return multipleOf(availQuantity, minOrderQuantity);
        }
    }
```

**Purpose**: Calculates matchable quantity for perp orders when mark price is favorable (no unrealized loss).

**Logic:**
1. **Calculate Fees**: Maker fee on order quantity
2. **Check Balance**: Verify user can pay fees
3. **Calculate Sequestration**: Required collateral for position
4. **Proportional Match**: If insufficient collateral, match proportionally

**Used When**: `markPrice >= orderPrice` (for buy orders) or `markPrice <= orderPrice` (for sell orders)

### Compute Perp Bal

```258:296:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function computePerpBal(SpotNode node, Price59EN5 mark, uint64 minOrderQuantity) external view returns(uint64) {
        OrderBookConfig orderBookConfig = internalReadExchangeAsRO().readOrderBookConfig(msg.sender);
        // fees for the transaction, maker:
        uint16 makerFeeRate = getFeeRate(orderBookConfig, node.client(), true, false);
        uint64 fromFeeQuantity = uint64(node.quantity() * (uint(makerFeeRate)) / ConfigConstants.FEE_DIVISOR);
        uint64 toFeeQuantity = node.price().convertFromToTo(fromFeeQuantity, orderBookConfig.ftpdDiff());
        if (orderBookConfig.toMaxFee() !=0 && toFeeQuantity > orderBookConfig.toMaxFee()) {
            toFeeQuantity = orderBookConfig.toMaxFee();
        }
        VaultTokenConfig baseConfig = internalReadTokenConfig(orderBookConfig.toTokenId());
        VaultLedger ledger = internalReadLedger(node.client(), orderBookConfig.toTokenId());
        uint64 bal64 = uint64(ledger.userBalance() / 10** (baseConfig.vaultDecimals() - baseConfig.positionDecimals()));
        if (bal64 <= toFeeQuantity) {
            return 0; // can't even pay the fees. Just cancel
        }
        bal64 -= toFeeQuantity;
        uint64 toQuantityAsMark = mark.convertFromToTo(node.quantity(), orderBookConfig.ftpdDiff());
        uint64 toQuantityAsMaker = node.price().convertFromToTo(node.quantity(), orderBookConfig.ftpdDiff());
        uint64 owed = 0;
        if (toQuantityAsMark >= toQuantityAsMaker) {
            owed = toQuantityAsMark - toQuantityAsMaker;
        } else {
            owed = toQuantityAsMaker - toQuantityAsMark;
        }

        VaultTokenConfig tokenConfig = internalReadTokenConfig(orderBookConfig.fromTokenId());
        owed += uint64(tokenConfig.perpSequesterAmount(toQuantityAsMaker));

        if (bal64 >= owed) {
            return node.quantity();
        }
        unchecked {
            uint64 availQuantity = uint64(uint256(bal64) * uint256(node.quantity()) / uint256(owed));
            if (availQuantity > node.quantity()) {
                availQuantity = node.quantity();
            }
            return multipleOf(availQuantity, minOrderQuantity);
        }
    }
```

**Purpose**: Calculates matchable quantity for perp orders when mark price causes unrealized loss.

**Additional Logic:**
- **Calculate Unrealized P&L**: `|toQuantityAsMark - toQuantityAsMaker|`
- **Total Owed**: Unrealized loss + sequestration amount
- **Proportional Match**: Match proportionally if insufficient collateral

**Used When**: `markPrice < orderPrice` (for buy orders) or `markPrice > orderPrice` (for sell orders)

---

## Lending Settlement

### Settle Lend Match

```207:213:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function settleLendMatch(LendMatch lendMatch, uint64 /*totalBuyerQuantity*/) external returns(uint64) {
        // transfer the funds
        VaultTokenConfig tokenConfig = internalReadTokenConfig(lendMatch.tokenId());
        uint128 vaultQuantity = tokenConfig.convertPositionToVault(lendMatch.quantity());
        transfer(lendMatch.tokenId(), vaultQuantity, lendMatch.lenderAccountId(), lendMatch.borrowerAccountId());
        return 0;
    }
```

**Purpose**: Simulates lending match settlement.

**Logic:**
- **Convert Quantity**: Convert position quantity to vault quantity
- **Transfer**: Transfer from lender to borrower
- **No Fee**: Lending fees handled separately

---

## Unimplemented Functions

These functions revert to indicate they're not used in this context:

```70:85:BaseDEX/contracts/src/main/sol/PriceHelper.sol
    function sequester(uint64 /*user*/, uint32 /*token*/, uint64 /*amount*/) external pure {
        revert NotImplemented();
    }

    function release(uint64 /*user*/, uint32 /*token*/, uint64 /*amount*/) external pure {
        revert NotImplemented();
    }


    function sequesterPerp(uint64 /*user*/, uint32 /*token*/,uint64 /*amount*/) external pure {
        revert NotImplemented();
    }

    function releasePerp(uint64 /*user*/, uint64 /*amount*/) external pure {
        revert NotImplemented();
    }
```

**Why?** `PriceHelper` implements `ICompExchForBooks` for interface compatibility, but sequestration is handled by the exchange, not `PriceHelper`.

---

## Summary

`PriceHelper.sol` is a helper contract that provides:
- **Fee Estimation**: Estimates fees for market orders and trading
- **Price Estimation**: Batch estimates market order execution prices
- **Settlement Simulation**: Simulates trade settlements using temporary storage
- **Balance Calculations**: Calculates available balances for perp orders
- **Temporary State Management**: Uses temporary storage to simulate state changes without modifying exchange

It enables orderbooks to:
- Estimate market order prices before execution
- Calculate fees for market orders specifying exact output
- Simulate trade settlements for testing and estimation
- Check collateral requirements for perp orders

This contract bridges orderbooks and the exchange, providing necessary calculations and simulations while maintaining separation of concerns.