Examining `ITwoTokenBookForExch.sol` to understand its purpose and implementation.


[4 tools called]

Checking what `ISpotOrderBook` provides and how orderbooks implement this interface:


[4 tools called]

## ITwoTokenBookForExch.sol - Explanation

`ITwoTokenBookForExch.sol` defines the interface between `CompositeExchange` and spot/perp orderbooks (`SpotOrderBook` and `PerpOrderBook`). It extends `ISpotOrderBook` (read-only) with functions for placing, canceling, and matching orders, including liquidation handling.

---

## Purpose and Architecture

```1:7:BaseDEX/contracts/src/main/sol/ITwoTokenBookForExch.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "./pub/ISpotOrderBook.sol";
import "./CompactStruct.sol";

interface ITwoTokenBookForExch is ISpotOrderBook {
```

Key characteristics:
- Extends `ISpotOrderBook`: Adds order placement/cancellation to read-only methods
- Implemented by: `TwoTokenOrderBook` (abstract), `SpotOrderBook`, `PerpOrderBook`
- Used by: `CompositeExchange` for routing user orders to orderbooks
- Design pattern: Separation of concerns — orderbooks handle matching; exchange handles settlement

---

## Interface Functions

### Order Placement Functions

#### 1. Regular Order Placement
```9:11:BaseDEX/contracts/src/main/sol/ITwoTokenBookForExch.sol
    function cancelSpotSellOrder(SpotCancelOrderData cancelOrderData) external;

    function newSpotSellOrder(SpotOrder orderData) external returns (SellOrderResult);

    function newSpotSellOrderLiquidation(SpotOrder orderData) external returns (SellOrderResult);
```

```15:19:BaseDEX/contracts/src/main/sol/ITwoTokenBookForExch.sol
    function cancelSpotBuyOrder(SpotCancelOrderData cancelOrderData) external;

    function newSpotBuyOrder(SpotOrder orderData) external returns (BuyOrderResult);

    function newSpotBuyOrderLiquidation(SpotOrder orderData) external returns (BuyOrderResult);
```

Regular vs liquidation:
- Regular orders (`newSpotBuyOrder`, `newSpotSellOrder`): Call `orderData.clean()` to sanitize inputs
- Liquidation orders (`newSpotBuyOrderLiquidation`, `newSpotSellOrderLiquidation`): Accept raw order data and bypass price checks for bankruptcy
- Implementation: Both delegate to internal functions (`internalNewSpotBuyOrder`, `internalNewSpotSellOrder`), but regular orders sanitize first

Liquidation price validation:
```528:536:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function checkBuyPrice(SpotOrder orderData) internal view {
        ICompositeExchangePub exch = ICompositeExchangePub(msg.sender);
        uint32 tokenId = readSpotTokenConfig().fromTokenId();
        uint256 mark = exch.getMarkPrice(tokenId).toScaledU256raw();
        VaultTokenConfig config = exch.readTokenConfig(tokenId);
        uint diff = uint(config.riskSlippagePercentx10())*4;
        uint256 orderPrice = orderData.price().toScaledU256raw();
        require(orderPrice < mark*(1000+diff)/1000, ExchangeErrors.LiquidationPriceOutOfRange());
    }
```

- Buy liquidations: Price must be < mark * (1 + slippage tolerance)
- Sell liquidations: Price must be > mark * (1 - slippage tolerance)
- Bankruptcy liquidations: Skip price checks

### Batch Operations

```21:21:BaseDEX/contracts/src/main/sol/ITwoTokenBookForExch.sol
    function batchCancelRebook(uint64 userId, BatchCommand command, BatchTwoTokenCancelRebook rebook) external returns (bool);
```

Purpose: Atomic cancel-and-replace for spot and perp orders.

Batch structure:
- `BatchCommand`: `commandType` (BUY/SELL), `orderType`, `ignoreMissingForCancel`
- `BatchTwoTokenCancelRebook`: `cancelOrderId`, `quantity`, `price`, `insertionHint`

Flow:
1. Cancel existing order (`cancelOrderId`)
2. Place new order (`quantity`, `price`) if provided
3. Returns `true` if execution occurred (for liquidation checks)

Example usage in `CompositeExchange`:
```873:877:BaseDEX/contracts/src/main/sol/CompositeExchange.sol
                } else {
                    BatchTwoTokenCancelRebook rebook = BatchTwoTokenCancelRebook.wrap(commands[i]);
                    ITwoTokenBookForExch book = ITwoTokenBookForExch(command.bookAddress());
                    checkLiq = book.batchCancelRebook(userId, command, rebook) || checkLiq;
                }
```

---

## Relationship with Parent Interface

### ISpotOrderBook Functions

From `ISpotOrderBook` (read-only):
- `bestBidOffer()`: Best bid/ask
- `retrieveBuyDepthChart()`, `retrieveSellDepthChart()`: Orderbook depth
- `calcBuyInsertionHint()`, `calcSellInsertionHint()`: Insertion hints
- `retrieveBuyOrder()`, `retrieveSellOrder()`: Order retrieval
- `searchBuyOrders()`, `searchSellOrders()`: User order search

`ITwoTokenBookForExch` adds:
- State-modifying functions: order placement and cancellation
- Liquidation support: special liquidation functions
- Batch operations: `batchCancelRebook`

---

## Usage Pattern

### Call Flow

1. User → `CompositeExchange`: User calls `newSpotBuyOrder()` or `newPerpBuyOrder()`
2. Exchange → Orderbook: Exchange calls `ITwoTokenBookForExch.newSpotBuyOrder()`
3. Orderbook → Exchange: Orderbook calls `ICompExchForBooks.settleSpotMakerBuyer()` for matches
4. Exchange → Orderbook: Exchange returns `SellOrderResult`/`BuyOrderResult`

Example from `CompositeExchange`:
```219:225:BaseDEX/contracts/src/main/sol/CompositeExchange.sol
        checkTradingPermission(accountId);
        requireOrderBook(orderBook, SPOT_BOOK_TYPE);
        ITwoTokenBookForExch book = ITwoTokenBookForExch(orderBook);
        BuyOrderResult result = book.newSpotBuyOrder(orderData);
        if (result.wasExecuted()) {
            ensureAboveLiqThresh(accountId);
        } else {
```

---

## Return Types

### BuyOrderResult / SellOrderResult

Compact structs (`CompactStruct.sol`) containing:
- `wasExecuted()`: Whether order was matched
- `quantityFilled()`: Quantity matched
- `quantityResting()`: Quantity remaining in orderbook
- `orderId()`: Order ID for cancellation

---

## Implementation Details

### Access Control

Functions use `onlyExchangeAndTrading`:
- `onlyExchange`: Only `CompositeExchange` can call
- `onlyTrading`: Trading must not be halted

Both regular and liquidation order functions require exchange access.

### Order Types Supported

- Spot orders: `SpotOrderBook`
- Perp orders: `PerpOrderBook` (uses same interface, different sequestration)

### Liquidation Integration

Liquidation functions are called during:
- User liquidation: `CompositeExchangeExt.liquidateUser()`
- Perp position liquidation: `CompositeExchangeExt.liquidatePerpPosition()`
- Credit liability liquidation: `CompositeExchangeExt.computeCreditLiability()`

Example from `CompositeExchangeExt`:
```674:681:BaseDEX/contracts/src/main/sol/CompositeExchangeExt.sol
            VaultTokenConfig config = internalReadTokenConfig(tokenId);
            address perpAddr = internalGetPerpBook(tokenId);
            ITwoTokenBookForExch perpBook = ITwoTokenBookForExch(perpAddr);
            Price59EN5 mark = readMarkPrice(tokenId);
            if (perpAgg.quantity() < 0) {
                Price59EN5 price = mark.inflateBase1k(config.riskSlippagePercentx10()*4);
                SpotOrder order = SpotOrderLib.newSpotOrder(price, uint64(-perpAgg.quantity()), userId, ORDER_TYPE_FILL_PARTIAL_KILL_REST, LiquidationLib.create(_bankruptcy));
                perpBook.newSpotBuyOrderLiquidation(order);
```

---

## Summary

`ITwoTokenBookForExch.sol` is the exchange-to-orderbook interface for spot and perp orderbooks. It:
- Extends `ISpotOrderBook` with state-modifying functions
- Supports regular and liquidation orders
- Enables atomic batch operations
- Maintains separation: orderbooks handle matching; exchange handles settlement
- Provides a consistent interface for both `SpotOrderBook` and `PerpOrderBook`

This interface enables the exchange to route orders to orderbooks while keeping matching logic separate from financial settlement.