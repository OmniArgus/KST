Searching for `TwoTokenOrderBook.sol` and reviewing its implementation.


[2 tools called]

Reading the full `TwoTokenOrderBook.sol` file:


[2 tools called]

## **TwoTokenOrderBook.sol - Explanation**

### **Purpose and importance**

`TwoTokenOrderBook` is an abstract contract that implements a Central Limit Order Book (CLOB) for two-token pairs. It is the base for `SpotOrderBook` and `PerpOrderBook`. It handles order matching, linked list management, ladder pricing, and market order simulation.

### **Core architecture**

#### **1. Inheritance structure**

```17:17:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
abstract contract TwoTokenOrderBook is OrderBookAbstract, ITwoTokenBookForExch {
```

- Inherits from `OrderBookAbstract` (sequence numbers, linked lists, access control)
- Implements `ITwoTokenBookForExch` (interface for exchange interaction)
- Abstract; child contracts implement sequestration and settlement

#### **2. Storage areas**

```30:31:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
        StorageUtils.writeStorage(BUY_ORDER_AREA, 0, 1); // length initialize to 1
        StorageUtils.writeStorage(SELL_ORDER_AREA, 0, 1); // length initialize to 1
```

- `BUY_ORDER_AREA`: Buy orders (descending price)
- `SELL_ORDER_AREA`: Sell orders (ascending price)
- Both use doubly linked lists for O(1) insertion/deletion

### **Order types and lifecycle**

#### **1. Order types**

- `ORDER_TYPE_LIMIT` (0): Standard limit order, may fill partially or fully
- `ORDER_TYPE_FILL_ALL_OR_REVERT` (2): Must fill completely or revert
- `ORDER_TYPE_FILL_PARTIAL_KILL_REST` (3): Fill partial, cancel remainder

#### **2. Order lifecycle**

**Sell order flow:**

```108:175:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function newSpotSellOrder(SpotOrder orderData) external onlyExchangeAndTrading returns(SellOrderResult) {
        return internalNewSpotSellOrder(orderData.clean());
    }

    function newSpotSellOrderLiquidation(SpotOrder orderData) external onlyExchangeAndTrading returns(SellOrderResult) {
        return internalNewSpotSellOrder(orderData);
    }

    function checkSellPrice(SpotOrder orderData) internal view {
        ICompositeExchangePub exch = ICompositeExchangePub(msg.sender);
        uint32 tokenId = readSpotTokenConfig().fromTokenId();
        uint256 mark = exch.getMarkPrice(tokenId).toScaledU256raw();
        VaultTokenConfig config = exch.readTokenConfig(tokenId);
        uint diff = uint(config.riskSlippagePercentx10())*4;
        uint256 orderPrice = orderData.price().toScaledU256raw();
        require(orderPrice > mark*(1000-diff)/1000, ExchangeErrors.LiquidationPriceOutOfRange());
    }

    function internalNewSpotSellOrder(SpotOrder orderData) internal returns(SellOrderResult) {
        // See SpotOrder in CompactStruct
        require(orderData.price().isNormalized(), ExchangeErrors.PriceNotNormalized()); // price not normalized
        checkOrderQuantity(orderData);
        if (orderData.liquidation().isLiquidation()) {
            if (!orderData.liquidation().isBankruptcy()) {
                checkSellPrice(orderData);
            }
        }
        uint64 clientId = orderData.client();
        uint64 orderId = createOrderId(findNextSellListNode(), nextOrderSeq());
        Price59EN5 sellPrice = orderData.price();
        emit ISpotOrderBook.NewSellOrder(SpotOrderDataLib.newAccountId(clientId),
            SpotOrderDataLib.newSpotOrderDataWithType(orderData.quantity(), sellPrice.raw(), orderId, orderData.orderType()));
        SellOrderResult sellResult = SellOrderResultLib.newSellOrderResult(false);
        uint32 pos = 0;
        if (orderData.insertionHint() == 0) {
            SpotOrder remaining = checkBuyMatch(orderData, clientId, orderId);
            if (remaining.isBlank()) {
                return SellOrderResultLib.newSellOrderResult(true);
            }
            require(orderData.orderType() != ORDER_TYPE_FILL_ALL_OR_REVERT, ExchangeErrors.UnableToFillCompletely());
            sellResult = SellOrderResultLib.newSellOrderResult(remaining.quantity() != orderData.quantity());
            if (orderData.orderType() == ORDER_TYPE_FILL_PARTIAL_KILL_REST) {
                emit ISpotOrderBook.CancelSellOrder(SpotOrderDataLib.newAccountId(clientId),
                    SpotOrderDataLib.newSpotOrderDataWithType(remaining.quantity(), remaining.price().raw(), orderId, ORDER_TYPE_FILL_PARTIAL_KILL_REST));
                return sellResult;
            }
            orderData = remaining;
        } else {
            require(orderData.orderType() == ORDER_TYPE_LIMIT, ExchangeErrors.OnlyLimitWithInsertionHint());
            pos = uint32(orderData.insertionHint());
            pos = checkSellInsertionHint(pos, sellPrice);
            if (pos == 0) {
                pos = uint32(orderData.insertionHint() >> 32);
                pos = checkSellInsertionHint(pos, sellPrice);
            }
            require(pos != 0, ExchangeErrors.InsertionHintStale());
        }
        sellSequester(clientId, orderData.quantity(), sellPrice);
        uint32 linkedListNodePos = allocateSellLinkedListNode();
        // nodeData = prev 32 | next 32 | order seq 20 | clientId 44 | sellQuantity 64 | sellPrice 64
        SpotNode nodeData = SpotNode.wrap((uint(orderId >> 32) << 172) | orderData.clientQuantityPrice());
        if (pos != 0) {
            insertIntoSellListFrom(pos, linkedListNodePos, nodeData, sellPrice);
        } else {
            insertIntoSellList(linkedListNodePos, nodeData, sellPrice);
        }
        return sellResult;
    }
```

Steps:
1. Validate price normalization
2. Check order quantity requirements
3. If liquidation, validate price within allowed range
4. Generate order ID (sequence + position)
5. If no insertion hint, attempt immediate matching
6. Sequester funds (virtual function, overridden by children)
7. Insert into sorted sell order list
8. Return result

### **Order matching algorithm**

#### **1. Ladder pricing**

Matches against multiple price levels, respecting each limit price:

```182:249:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function checkBuyMatch(SpotOrder orderData, uint64 clientId, uint64 orderId) internal returns (SpotOrder) {
        ListMeta linkedListMeta = ListMetaLib.read(BUY_ORDER_AREA);
        // meta = empty 32 bit | head 32 | empty 128bits | free pointer 32 | length pointer 32
        uint32 head = linkedListMeta.head();
        Price59EN5 sellPrice = orderData.price();
        uint64 sellQuantity = orderData.quantity();
        if (head == 0) {
            return orderData;
        }
        SpotNode buyNodeData = SpotNodeLib.read(BUY_ORDER_AREA, head);
        if (buyNodeData.price().lessThan(sellPrice)) {
            return orderData;
        }
        // we have a match! use ladder pricing
        uint64 sellerFeeSoFar = 0;
        uint pricingData = gatherPricingData(msg.sender);
        while (sellPrice.lessThanEquals(buyNodeData.price()) && sellQuantity != 0 && head != 0) {
            uint64 buyQuantity = 0;
            if (clientId == buyNodeData.client()) {
                require(orderData.liquidation().isLiquidation(), ExchangeErrors.NoSelfDealing());
                // buyQuantity is zero here, which will cause the buy side to be canceled.
            } else {
                buyQuantity = checkBuyBalance(buyNodeData, msg.sender, pricingData);
            }

            if (sellQuantity >= buyQuantity) {
                if (buyQuantity > 0) {
                    sellQuantity = sellQuantity - buyQuantity;
                    sellerFeeSoFar = handleSellerInitiatedTrade(buyQuantity, orderData, buyNodeData, head, orderId, sellerFeeSoFar, pricingData);
                }
                if (buyQuantity < buyNodeData.quantity()) {
                    emitCancelBuy(buyNodeData.price(), buyNodeData.quantity() - buyQuantity, buyNodeData.client(), createOrderId(head, buyNodeData.orderSeq()));
                }
                linkedListMeta = updateListMetaDataForFreedNode(linkedListMeta, BUY_ORDER_AREA, head);
                head = buyNodeData.next(); // head = head.next
                linkedListMeta = linkedListMeta.withHead(head);
                if (head != 0) {
                    buyNodeData = SpotNodeLib.read(BUY_ORDER_AREA, head);
                    StorageUtils.writeStorage(BUY_ORDER_AREA, head, buyNodeData.zeroOutPrev().raw()); // set the head.prev to zero
                }
            } else {
                sellerFeeSoFar = handleSellerInitiatedTrade(sellQuantity, orderData, buyNodeData, head, orderId, sellerFeeSoFar, pricingData);
                buyQuantity = buyNodeData.quantity() - sellQuantity; // don't reuse buyQuantity, as it may be lower than buyNodeData.quantity()
                sellQuantity = 0;
                if (buyQuantity < readMinOrderQuantity() &&
                    buyNodeData.price().convertFromToTo(buyQuantity, readSpotTokenConfig().ftpdDiff()) == 0) {
                    // this order is now so small that it has nothing sequestered!
                    emitCancelBuy(buyNodeData.price(), buyQuantity, buyNodeData.client(), createOrderId(head, buyNodeData.orderSeq()));
                    linkedListMeta = updateListMetaDataForFreedNode(linkedListMeta, BUY_ORDER_AREA, head);
                    head = buyNodeData.next(); // head = head.next
                    linkedListMeta = linkedListMeta.withHead(head);
                    if (head != 0) {
                        buyNodeData = SpotNodeLib.read(BUY_ORDER_AREA, head);
                        StorageUtils.writeStorage(BUY_ORDER_AREA, head, buyNodeData.zeroOutPrev().raw()); // set the head.prev to zero
                    }
                } else {
                    buyNodeData = buyNodeData.withQuantity(buyQuantity);
                    StorageUtils.writeStorage(BUY_ORDER_AREA, head, buyNodeData.zeroOutPrev().raw()); // set the head.prev to zero
                }
            }
        }
        StorageUtils.writeStorage(BUY_ORDER_AREA, 0, linkedListMeta.raw());
        if (sellQuantity == 0) {
            return SpotOrderLib.blank();
        }
        orderData = orderData.withQuantity(sellQuantity);
        return orderData;
    }
```

Algorithm:
1. Check if buy orders exist
2. Verify price compatibility (`sellPrice <= buyPrice`)
3. While matching conditions hold:
   - Check buyer balance (virtual function)
   - Prevent self-dealing (unless liquidation)
   - Match quantity (partial or full)
   - Update or remove matched orders
4. Return remaining unmatched quantity

**Buy order matching** (`checkSellMatch`) follows the same pattern but against the sell side.

### **Market order price computation**

#### **1. Seller-initiated market orders**

```254:295:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function computeSellMarketOrderSpecifyIn(MarketOrderPrice orderData) external returns (MarketOrderPriceResult) {
        require(msg.sender != readVaultAddress());
        ListMeta linkedListMeta = ListMetaLib.read(BUY_ORDER_AREA);
        uint32 head = linkedListMeta.head();
        Price59EN5 sellPrice = orderData.limitPrice();
        uint64 sellQuantity = orderData.amountIn();
        if (head == 0) {
            return MarketOrderPriceResultLib.empty();
        }
        SpotNode buyNodeData = SpotNodeLib.read(BUY_ORDER_AREA, head);
        if (!sellPrice.isZero() && buyNodeData.price().lessThan(sellPrice)) {
            return MarketOrderPriceResultLib.empty();
        }
        // we have a match! use ladder pricing
        uint64 sellerFeeSoFar = 0;
        uint pricingData = gatherPricingData(msg.sender);
        uint64 amountOut = 0;
        while ((sellPrice.isZero() || sellPrice.lessThanEquals(buyNodeData.price())) && sellQuantity != 0 && head != 0) {
            uint64 buyQuantity = 0;
            if (orderData.client() != buyNodeData.client()) {
                buyQuantity = checkBuyBalance(buyNodeData, msg.sender, pricingData);
            } else {
                continue;
            }
            if (sellQuantity >= buyQuantity) {
                if (buyQuantity > 0) {
                    sellQuantity = sellQuantity - buyQuantity;
                    SpotMatchQuantities smq = settleSellerInitiatedTrade(buyQuantity, orderForPricing(orderData.isLiquidation()), buyNodeData, sellerFeeSoFar, pricingData);
                    sellerFeeSoFar += smq.toFee();
                    amountOut += smq.toQuantity();
                }
                head = buyNodeData.next(); // head = head.next
            } else {
                SpotMatchQuantities smq = settleSellerInitiatedTrade(sellQuantity, orderForPricing(orderData.isLiquidation()), buyNodeData, sellerFeeSoFar, pricingData);
                sellerFeeSoFar += smq.toFee();
                amountOut += smq.toQuantity();
                buyQuantity = buyNodeData.quantity() - sellQuantity; // don't reuse buyQuantity, as it may be lower than buyNodeData.quantity()
                sellQuantity = 0;
            }
        }
        return MarketOrderPriceResultLib.create(orderData.amountIn() - sellQuantity, amountOut, buyNodeData.price(), orderData.amountIn() - sellQuantity);
    }
```

- Simulates matching without execution
- Used by `PriceHelper` for price estimation
- Calculates expected output for a given input amount

#### **2. Seller-initiated market orders (specify output)**

```300:369:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function computeSellMarketOrderSpecifyOut(MarketOrderPrice orderData) external returns (MarketOrderPriceResult) {
        require(msg.sender != readVaultAddress());
        ListMeta linkedListMeta = ListMetaLib.read(BUY_ORDER_AREA);
        SpotTokenConfig spotTokenConfig = readSpotTokenConfig();
        uint32 head = linkedListMeta.head();
        Price59EN5 sellPrice = orderData.limitPrice();
        uint64 amountOut = orderData.amountOut();
        if (head == 0) {
            return MarketOrderPriceResultLib.empty();
        }
        SpotNode buyNodeData = SpotNodeLib.read(BUY_ORDER_AREA, head);
        if (!sellPrice.isZero() && buyNodeData.price().lessThan(sellPrice)) {
            return MarketOrderPriceResultLib.empty();
        }
        // we have a match! use ladder pricing
        uint64 sellerFeeSoFar = 0;
        uint64 amountIn = 0;
        uint pricingData = gatherPricingData(msg.sender);
        while ((sellPrice.isZero() || sellPrice.lessThanEquals(buyNodeData.price())) && amountOut > 1 && head != 0) {
            uint64 buyQuantity = 0;
            if (orderData.client() != buyNodeData.client()) {
                buyQuantity = checkBuyBalance(buyNodeData, msg.sender, pricingData);
            } else {
                continue;
            }
            uint64 fullToQuantity = buyNodeData.price().convertFromToTo(buyQuantity, spotTokenConfig.ftpdDiff());
            if (fullToQuantity < amountOut) {
                SpotMatchQuantities smq = settleSellerInitiatedTrade(buyQuantity, orderForPricing(orderData.isLiquidation()), buyNodeData, sellerFeeSoFar, pricingData);
                sellerFeeSoFar += smq.toFee();
                if (smq.toQuantity() <= amountOut) {
                    amountOut -= smq.toQuantity();
                } else {
                    amountOut = 0;
                }
                amountIn += smq.fromQuantity() + smq.fromFee();
            } else {
                while(buyQuantity > 0 && amountOut > 1) { // " > 1" because the fee is at least 1
                    uint64 partialQuantity = uint64(uint(buyQuantity) * uint(amountOut) / uint(fullToQuantity));
                    if (partialQuantity == 0) {
                        break;
                    }
                    IPriceHelperForBooks priceHelper = IPriceHelperForBooks(msg.sender);
                    uint64 estimatedFees = priceHelper.estimateFeeFromQuantity(partialQuantity, sellerFeeSoFar);
                    partialQuantity += estimatedFees;
                    if (partialQuantity > buyQuantity) {
                        partialQuantity = buyQuantity;
                    }
                    uint64 partialToQuantity = buyNodeData.price().convertFromToTo(partialQuantity, spotTokenConfig.ftpdDiff());
                    if (partialToQuantity == 0) {
                        break;
                    }
                    SpotMatchQuantities smq = settleSellerInitiatedTrade(partialQuantity, orderForPricing(orderData.isLiquidation()), buyNodeData, sellerFeeSoFar, pricingData);
                    if (smq.toQuantity() == 0) {
                        break;
                    }
                    sellerFeeSoFar += smq.toFee();
                    amountIn += smq.fromQuantity() + smq.fromFee();
                    if (smq.toQuantity() <= amountOut) {
                        amountOut -= smq.toQuantity();
                    } else {
                        amountOut = 0;
                        amountIn -= 1;
                    }
                    buyQuantity -= partialQuantity;
                }
            }
            head = buyNodeData.next();
        }
        return MarketOrderPriceResultLib.create(amountIn, orderData.amountOut() - amountOut, buyNodeData.price(), amountIn);
    }
```

- Computes required input for a desired output (with iterative fee estimation for partial fills)

### **Linked list management**

#### **1. Sorted insertion**

**Sell orders** (ascending price):

```414:425:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function insertIntoSellListFrom(uint32 start, uint32 linkedListNodePos,SpotNode nodeData, Price59EN5 sellPrice) internal {
        uint32 prev = 0;
        uint32 cur = start;
        // sell orders are min price to max price sorted (head points to min)
        SpotNode curNodeData = SpotNodeLib.read(SELL_ORDER_AREA, cur);
        while (sellPrice.greaterThanEquals(curNodeData.price()) && cur != 0) {
            prev = cur;
            cur = curNodeData.next();
            curNodeData = SpotNodeLib.read(SELL_ORDER_AREA, cur);
        }
        insertIntoList(SELL_ORDER_AREA, linkedListNodePos, prev, cur, nodeData.asGeneric());
    }
```

**Buy orders** (descending price):

```836:847:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function insertIntoBuyListFrom(uint32 start, uint32 linkedListNodePos,SpotNode nodeData, Price59EN5 buyPrice) internal {
        uint32 prev = 0;
        uint32 cur = start;
        //buy orders are max price to min price sorted (head points to max)
        SpotNode curNodeData = SpotNodeLib.read(BUY_ORDER_AREA, cur);
        while (buyPrice.lessThanEquals(curNodeData.price()) && cur != 0) {
            prev = cur;
            cur = curNodeData.next();
            curNodeData = SpotNodeLib.read(BUY_ORDER_AREA, cur);
        }
        insertIntoList(BUY_ORDER_AREA, linkedListNodePos, prev, cur, nodeData.asGeneric());
    }
```

### **Abstract functions (overridden by children)**

1. `sellSequester` / `buySequester`: Lock funds for orders
2. `emitCancelSell` / `emitCancelBuy`: Release funds on cancellation
3. `checkBuyBalance` / `checkSellBalance`: Verify sufficient balance
4. `gatherPricingData`: Collect pricing info (e.g., mark price)
5. `settleSellerInitiatedTrade` / `settleBuyerInitiatedTrade`: Handle settlement
6. `checkOrderQuantity`: Validate order quantity (e.g., multiples of min)

### **Read-only functions**

#### **1. Best bid/offer**

```46:50:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function bestBidOffer() public view returns (BestBidOffer) {
        SpotNode bestBuy = readFirstOrder(BUY_ORDER_AREA);
        SpotNode bestSell = readFirstOrder(SELL_ORDER_AREA);
        return BestBidOfferPlib.fromSpotNodes(bestBuy, bestSell);
    }
```

#### **2. Depth chart retrieval**

```878:883:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function retrieveBuyDepthChart(uint32 maxDepth, uint64 restartPosition) external view returns (uint[] memory) {
        return retrieveDepthChart(maxDepth, restartPosition, BUY_ORDER_AREA, false);
    }

    function retrieveSellDepthChart(uint32 maxDepth, uint64 restartPosition) external view returns (uint[] memory) {
        return retrieveDepthChart(maxDepth, restartPosition, SELL_ORDER_AREA, true);
    }
```

- Returns aggregated price levels with quantities
- Supports pagination via `restartPosition`

#### **3. Order search**

```980:986:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function searchBuyOrders(uint64 userId, uint32 maxDepth, uint32 maxOrders, uint64 restartPosition) external view returns (uint[] memory) {
        return searchOrders(userId, maxDepth, maxOrders, restartPosition, BUY_ORDER_AREA);
    }

    function searchSellOrders(uint64 userId, uint32 maxDepth, uint32 maxOrders, uint64 restartPosition) external view returns (uint[] memory) {
        return searchOrders(userId, maxDepth, maxOrders, restartPosition, SELL_ORDER_AREA);
    }
```

- Finds all orders for a given user, with pagination

### **Batch operations**

```494:518:BaseDEX/contracts/src/main/sol/TwoTokenOrderBook.sol
    function batchCancelRebook(uint64 userId, BatchCommand command, BatchTwoTokenCancelRebook rebook) external onlyExchange returns (bool) {
        bool checkLiq = false;
        if (command.commandType() == COMMAND_TYPE_BUY) {
            if (rebook.cancelOrderId() != 0) {
                internalCancelSpotBuyOrder(rebook.asCancelOrderData(userId, command.ignoreMissingForCancel()));
            }
            if (rebook.quantity() != 0) {
                require(!readConfigParams().tradingHalted(), ExchangeErrors.TradingHalted());
                BuyOrderResult result = internalNewSpotBuyOrder(rebook.asSpotOrder(userId, command.orderType()));
                checkLiq = checkLiq || result.wasExecuted();
            }
        } else if (command.commandType() == COMMAND_TYPE_SELL) {
            if (rebook.cancelOrderId() != 0) {
                internalCancelSpotSellOrder(rebook.asCancelOrderData(userId, command.ignoreMissingForCancel()));
            }
            if (rebook.quantity() != 0) {
                require(!readConfigParams().tradingHalted(), ExchangeErrors.TradingHalted());
                SellOrderResult result = internalNewSpotSellOrder(rebook.asSpotOrder(userId, command.orderType()));
                checkLiq = checkLiq || result.wasExecuted();
            }
        } else {
            revert ExchangeErrors.InvalidBatchCommand();
        }
        return checkLiq;
    }
```

- Allows atomically canceling and placing a new order (rebook)

### **Design choices**

1. Ladder pricing: Respects individual limit prices
2. Separation of concerns: Matching here, settlement in children/exchange
3. Gas optimization: Compact structs, efficient linked lists, insertion hints
4. Self-dealing prevention: Blocks matching own orders (except liquidations)
5. Partial fills: Orders can be partially filled across levels

### **Summary**

`TwoTokenOrderBook` implements a CLOB matching engine for two-token pairs. It handles order management, matching, price simulation, and read-only queries. Child contracts (`SpotOrderBook`, `PerpOrderBook`) implement settlement and sequestration for their specific asset types. This design enables on-chain limit order trading with price-time priority and efficient matching.