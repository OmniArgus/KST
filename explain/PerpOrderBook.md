Examining `PerpOrderBook.sol` to understand its purpose and implementation.


[4 tools called]

Checking how perpetual futures differ from spot trading in balance checks and settlement:


[3 tools called]

## PerpOrderBook.sol - Explanation

`PerpOrderBook.sol` handles perpetual futures orders. It inherits from `TwoTokenOrderBook` and overrides sequestration, balance checks, and settlement to handle perps. Perps are settled in cash (base token), not physical delivery.

---

## Purpose and Architecture

```13:17:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
contract PerpOrderBook is TwoTokenOrderBook {

    constructor(address , address vault, TokenPair tokenPair, uint64 minOrderQuantity)
    TwoTokenOrderBook(vault, tokenPair, minOrderQuantity) {
    }
```

**Characteristics:**
- Inherits from `TwoTokenOrderBook`: reuses matching and orderbook logic
- Overrides key functions: sequestration, balance checks, settlement
- Uses same order structure: `SpotOrder` (aliased as `PerpOrder` for clarity)
- Cash-settled: positions settled in base token (e.g., USDC), not underlying asset

---

## Differences from Spot Trading

### 1. Notional-based sequestration

Unlike spot trading, perps sequester collateral based on notional value (price × quantity) in the base token, not the actual token.

#### Sell Order Sequestration

```29:39:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    // perp book does not sequester anything. It dynamically checks the orders versus available collateral
    function sellSequester(uint64 clientId, uint64 quantity, Price59EN5 price) internal override {
        sequesterBySlippageRisk(clientId, quantity, price);
    }

    function sequesterBySlippageRisk(uint64 clientId, uint64 quantity, Price59EN5 price) internal {
        SpotTokenConfig spotTokenConfig = readSpotTokenConfig();
        uint64 notional = price.convertFromToTo(quantity, spotTokenConfig.ftpdDiff());
        ICompExchForBooks v = ICompExchForBooks(msg.sender); // we've verified that this call comes from the exchange
        v.sequesterPerp(clientId, spotTokenConfig.fromTokenId(), notional);
    }
```

**Flow:**
1. Calculate notional: `notional = price.convertFromToTo(quantity, ftpdDiff)`
2. Sequester base token: `sequesterPerp(clientId, fromTokenId, notional)`

**Why notional?**
- Perps are leveraged positions
- Collateral is held in base token to cover P&L
- `fromTokenId` identifies the perp market; sequestration is in base token

#### Buy Order Sequestration

```48:50:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function buySequester(uint64 clientId, uint64 quantity, Price59EN5 price) internal override {
        sequesterBySlippageRisk(clientId, quantity, price);
    }
```

**Note**: Buy and sell orders use the same sequestration logic (both sequester base token).

#### Release Logic

```41:46:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function releaseBySlippageRisk(uint64 clientId, uint64 quantity, Price59EN5 price) internal {
        SpotTokenConfig spotTokenConfig = readSpotTokenConfig();
        uint64 notional = price.convertFromToTo(quantity, spotTokenConfig.ftpdDiff());
        ICompExchForBooks v = ICompExchForBooks(msg.sender); // we've verified that this call comes from the exchange
        v.releasePerp(clientId, notional);
    }
```

**Used in**: Order cancellation (`emitCancelSell`, `emitCancelBuy`)

---

### 2. Mark Price Integration

Perps use mark price (oracle price) for balance checks and risk management.

```52:55:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function gatherPricingData(address vaultAddress) internal view override returns (uint) {
        ICompExchForBooks v = ICompExchForBooks(vaultAddress); // we've verified that this call comes from the exchange
        return v.getMarkPrice(readSpotTokenConfig().fromTokenId()).raw();
    }
```

**Purpose**: Retrieves current mark price for the perp market.

**Usage**: Used in balance checks to evaluate position risk.

---

### 3. Advanced Balance Checking

Perp balance checks account for mark price, unrealized P&L, and fees.

#### Buy Order Balance Check

```57:67:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function checkBuyBalance(SpotNode buyNodeData, address vaultAddress, uint pricingData) internal override view returns (uint64) {
        Price59EN5 mark = Price59EN5.wrap(uint64(pricingData));
        ICompExchForBooks v = ICompExchForBooks(vaultAddress); // we've verified that this call comes from the exchange
        if (mark.greaterThanEquals(buyNodeData.price())) {
            // just check for fees
            return v.computePerpBalFeesOnly(buyNodeData, readMinOrderQuantity());
        } else {
            // check for fees and delta
            return v.computePerpBal(buyNodeData, mark, readMinOrderQuantity());
        }
    }
```

**Logic:**
- If `mark >= orderPrice`: profit or break-even; only check fees (`computePerpBalFeesOnly`)
- If `mark < orderPrice`: unrealized loss; check fees + P&L (`computePerpBal`)

**Why?**
- Long positions profit when price rises
- Must ensure sufficient collateral for fees and potential losses

#### Sell Order Balance Check

```84:94:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function checkSellBalance(SpotNode sellNodeData, address vaultAddress, uint pricingData) internal override view returns (uint64) {
        Price59EN5 mark = Price59EN5.wrap(uint64(pricingData));
        ICompExchForBooks v = ICompExchForBooks(vaultAddress); // we've verified that this call comes from the exchange
        if (mark.lessThanEquals(sellNodeData.price())) {
            // just check for fees
            return v.computePerpBalFeesOnly(sellNodeData, readMinOrderQuantity());
        } else {
            // check for fees and delta
            return v.computePerpBal(sellNodeData, mark, readMinOrderQuantity());
        }
    }
```

**Logic:**
- If `mark <= orderPrice`: profit or break-even for short; only check fees
- If `mark > orderPrice`: unrealized loss for short; check fees + P&L

**Why?**
- Short positions profit when price falls
- Must ensure sufficient collateral for fees and potential losses

**Implementation details** (`computePerpBal` in `CompositeExchange`):
- Calculates fees (maker fee rate)
- Calculates unrealized P&L: `(markPrice - orderPrice) × quantity`
- Ensures collateral covers fees + max(0, unrealized loss)
- Returns matchable quantity considering available collateral

---

### 4. Perp-Specific Settlement

Perps use different settlement functions that handle position creation/modification.

#### Seller-Initiated Trade

```69:76:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function settleSellerInitiatedTrade(uint64 matchQuantity, SpotOrder sellOrder, SpotNode buyNodeData,
        uint64 sellerFeeSoFar, uint pricingData) internal override returns (SpotMatchQuantities) {
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        MakerSpotMatch makerSpotMatch = MakerSpotMatchLib.newMakerSpotMatch(buyNodeData, true, readFirstOrder(SELL_ORDER_AREA).price(), sellOrder.liquidation().isLiquidation());
        TakerSpotMatch takerSpotMatch = TakerSpotMatchLib.newTakerSpotMatch(sellOrder.client(), matchQuantity, sellerFeeSoFar, uint64(pricingData));
        SpotMatchQuantities result = v.settlePerpMakerBuyer(makerSpotMatch, takerSpotMatch);
        return result;
    }
```

**Key differences:**
- Uses `settlePerpMakerBuyer()` instead of `settleSpotMakerBuyer()`
- Includes mark price in `TakerSpotMatch` (via `pricingData`)
- May include liquidation flag

#### Buyer-Initiated Trade

```96:103:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function settleBuyerInitiatedTrade(uint64 matchQuantity, SpotOrder buyOrder, SpotNode sellNodeData,
        uint64 buyerFeeSoFar, uint pricingData) internal override returns (SpotMatchQuantities) {
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        MakerSpotMatch makerSpotMatch = MakerSpotMatchLib.newMakerSpotMatch(sellNodeData, false, readFirstOrder(BUY_ORDER_AREA).price(), buyOrder.liquidation().isLiquidation());
        TakerSpotMatch takerSpotMatch = TakerSpotMatchLib.newTakerSpotMatch(buyOrder.client(), matchQuantity, buyerFeeSoFar, uint64(pricingData));
        SpotMatchQuantities result = v.settlePerpMakerSeller(makerSpotMatch, takerSpotMatch);
        return result;
    }
```

**Differences:**
- Uses `settlePerpMakerSeller()` instead of `settleSpotMakerSeller()`
- Includes mark price for P&L calculation

**What `settlePerpMakerBuyer`/`settlePerpMakerSeller` do:**
- Create or modify perp positions
- Calculate and settle P&L at mark price
- Handle position netting (offsetting long/short)
- Charge fees and adjust collateral
- Handle funding rate payments

---

### 5. Order Quantity Validation

```25:27:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function checkOrderQuantity(SpotOrder orderData) internal override view {
        require(orderData.quantity() % readMinOrderQuantity() == 0, ExchangeErrors.QuantityNotMultipleOfMin());
    }
```

**Requirement**: Quantity must be a multiple of `minOrderQuantity`.

**Why?**
- Ensures consistent position sizes
- Simplifies risk calculations
- Spot allows `>= minOrderQuantity`; perps require exact multiples

---

## Order Cancellation

### Cancel Sell Order

```19:23:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function emitCancelSell(Price59EN5 price, uint64 quantity, uint64 clientId, uint64 orderId) internal override {
        releaseBySlippageRisk(clientId, quantity, price);
        emit CancelSellOrder(SpotOrderDataLib.newAccountId(clientId),
            SpotOrderDataLib.newSpotOrderData(quantity, price.raw(), orderId));
    }
```

**Flow:**
1. Release sequestered base token collateral
2. Emit cancellation event

### Cancel Buy Order

```78:82:BaseDEX/contracts/src/main/sol/PerpOrderBook.sol
    function emitCancelBuy(Price59EN5 price, uint64 quantity, uint64 clientId, uint64 orderId) internal override {
        releaseBySlippageRisk(clientId, quantity, price);
        emit CancelBuyOrder(SpotOrderDataLib.newAccountId(clientId),
            SpotOrderDataLib.newSpotOrderData(quantity, price.raw(), orderId));
    }
```

**Same logic**: Both buy and sell use `releaseBySlippageRisk`.

---

## Comparison: Spot vs Perp

| Aspect | SpotOrderBook | PerpOrderBook |
|--------|---------------|----------------|
| **Sequestration** | Actual tokens (`sequester`) | Base token notional (`sequesterPerp`) |
| **Balance Check** | Simple token balance | Mark price + fees + P&L |
| **Settlement** | Token transfer (`settleSpotMaker*`) | Position management (`settlePerpMaker*`) |
| **Pricing Data** | Not needed (0) | Mark price (oracle) |
| **Quantity Check** | `>= minOrderQuantity` | `% minOrderQuantity == 0` |
| **Collateral** | Token being traded | Base token (USDC) |

---

## Perpetual Futures Mechanics

### Position Management
- Long vs Short: Created by matching buy/sell orders
- Netting: Long and short positions net automatically
- Mark-to-Market: Positions valued at mark price continuously
- Funding Rates: Periodic payments between longs and shorts

### Risk Management
- Collateral Requirements: Base token must cover unrealized losses
- Dynamic Checks: Balance checks use current mark price
- Liquidation: Positions can be liquidated if collateral insufficient

### Trading Flow
1. User places order: `newPerpBuyOrder()` or `newPerpSellOrder()`
2. Orderbook sequesters: Base token collateral based on notional
3. Matching: Orders matched using ladder pricing
4. Settlement: `settlePerpMaker*` creates/modifies positions
5. Position Updates: P&L calculated at mark price

---

## Summary

`PerpOrderBook.sol` adapts `TwoTokenOrderBook` for perpetual futures:
- Notional-based sequestration: Sequesters base token collateral, not underlying tokens
- Mark price integration: Uses oracle prices for balance checks and risk management
- Advanced balance checks: Considers fees and unrealized P&L
- Perp-specific settlement: Handles position creation, netting, and P&L
- Strict quantity validation: Requires exact multiples of `minOrderQuantity`

It enables leveraged perpetual futures trading with proper risk management and collateral requirements.