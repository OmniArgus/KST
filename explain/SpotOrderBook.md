Examining `SpotOrderBook.sol` to understand its purpose and implementation.


[4 tools called]

## SpotOrderBook.sol - Explanation

`SpotOrderBook.sol` handles spot token trading. It extends `TwoTokenOrderBook` and overrides functions for spot-specific sequestration, balance checks, and settlement. Unlike `PerpOrderBook`, it transfers actual tokens rather than managing perpetual positions.

---

## Purpose and Architecture

```13:17:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
contract SpotOrderBook is TwoTokenOrderBook {

    constructor(address , address vault, TokenPair tokenPair, uint64 minOrderQuantity)
    TwoTokenOrderBook(vault, tokenPair, minOrderQuantity) {
    }
```

**Characteristics:**
- **Concrete Implementation**: Extends abstract `TwoTokenOrderBook`
- **Token Pair Trading**: Handles spot trades between two tokens
- **Physical Delivery**: Transfers actual tokens (not cash-settled)
- **Same Order Structure**: Uses `SpotOrder` like perps, but with different semantics

**Key Differences from PerpOrderBook:**
- **Sequestration**: Locks actual tokens, not base token notional
- **Settlement**: Transfers tokens between users
- **Balance Checks**: Simple token balance verification
- **No Mark Price**: Doesn't need oracle prices for matching

---

## Token Sequestration

### Sell Order Sequestration (Seller Locks "From" Token)

```26:29:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function sellSequester(uint64 clientId, uint64 quantity, Price59EN5) internal override {
        ICompExchForBooks v = ICompExchForBooks(msg.sender); // we've verified that this call comes from the exchange
        v.sequester(clientId, readSpotTokenConfig().fromTokenId(), quantity);
    }
```

**Flow:**
1. **Identify Token**: Uses `fromTokenId` (token being sold)
2. **Lock Tokens**: Sequesters actual quantity being sold
3. **Purpose**: Ensures seller has tokens available

**Example**: Selling 100 ETH for USDC
- Sequesters: 100 ETH (fromTokenId)
- Not: USDC (doesn't sequester what buyer pays)

### Buy Order Sequestration (Buyer Locks "To" Token)

```75:80:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function buySequester(uint64 clientId, uint64 quantity, Price59EN5 price) internal override {
        SpotTokenConfig spotTokenConfig = readSpotTokenConfig();
        uint64 toSequester = price.convertFromToTo(quantity, spotTokenConfig.ftpdDiff());
        ICompExchForBooks v = ICompExchForBooks(msg.sender); // onlyExchange call
        v.sequester(clientId, spotTokenConfig.toTokenId(), toSequester);
    }
```

**Flow:**
1. **Calculate Required Payment**: `toSequester = price.convertFromToTo(quantity, ftpdDiff)`
2. **Convert Tokens**: Converts from "from" token quantity to "to" token quantity
3. **Lock Payment Token**: Sequesters "to" token (token being purchased with)

**Example**: Buying 100 ETH with USDC at price 3000 USDC/ETH
- Order quantity: 100 ETH (fromTokenId)
- Sequestered: 300,000 USDC (toTokenId = price × quantity)
- Not: ETH (doesn't sequester what buyer wants)

**Why Price Conversion?**
- Buy orders specify quantity of "from" token (100 ETH)
- Payment required in "to" token (300,000 USDC)
- Must convert using price to determine required payment

---

## Order Cancellation and Release

### Cancel Sell Order

```19:24:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function emitCancelSell(Price59EN5 price, uint64 quantity, uint64 clientId, uint64 orderId) internal override {
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        v.release(clientId, readSpotTokenConfig().fromTokenId(), quantity);
        emit CancelSellOrder(SpotOrderDataLib.newAccountId(clientId),
            SpotOrderDataLib.newSpotOrderData(quantity, price.raw(), orderId));
    }
```

**Flow:**
1. **Release Tokens**: Releases sequestered "from" token
2. **Emit Event**: Notifies cancellation

### Cancel Buy Order

```66:73:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function emitCancelBuy(Price59EN5 price, uint64 quantity, uint64 clientId, uint64 orderId) internal override {
        SpotTokenConfig spotTokenConfig = readSpotTokenConfig();
        uint64 toRelease = price.convertFromToTo(quantity, spotTokenConfig.ftpdDiff());
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        v.release(clientId, spotTokenConfig.toTokenId(), toRelease);
        emit CancelBuyOrder(SpotOrderDataLib.newAccountId(clientId),
            SpotOrderDataLib.newSpotOrderData(quantity, price.raw(), orderId));
    }
```

**Flow:**
1. **Calculate Release**: Converts quantity to payment token amount
2. **Release Payment**: Releases sequestered "to" token
3. **Emit Event**: Notifies cancellation

---

## Balance Checks

### Buy Order Balance Check

```35:49:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function checkBuyBalance(SpotNode buyNodeData, address vaultAddress, uint) internal override view returns (uint64) {
        SpotTokenConfig config = readSpotTokenConfig();
        ICompositeExchangeReadOnly vault = ICompositeExchangeReadOnly(vaultAddress);
        (uint128 bal,, ) = vault.getBalance(buyNodeData.client(), config.toTokenId());
        uint64 positionBal = uint64(bal / 10**(config.toVaultPdDiff()));
        if (positionBal == 0) {
            return 0;
        }
        if (buyNodeData.price().convertFromToTo(buyNodeData.quantity(), config.ftpdDiff()) <= positionBal) {
            // check the forward conversion to avoid rounding errors from division below
            return buyNodeData.quantity();
        }
        uint64 maxQuantity = buyNodeData.price().convertToToFrom(positionBal, config.ftpdDiff());
        return maxQuantity >= buyNodeData.quantity() ? buyNodeData.quantity() : maxQuantity;
    }
```

**Logic:**
1. **Get Balance**: Check buyer's balance of "to" token (payment token)
2. **Convert Order**: Convert order quantity to payment amount
3. **Check Sufficiency**:
   - If payment amount ≤ balance: Return full order quantity
   - Otherwise: Calculate max buyable quantity

**Example**: Buying 1 ETH at 3000 USDC/ETH
- Order: 1 ETH
- Required payment: 3000 USDC
- Buyer balance: 1500 USDC
- Result: 0.5 ETH (maximum matchable)

### Sell Order Balance Check

```82:91:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function checkSellBalance(SpotNode sellNodeData, address vaultAddress, uint) internal override view returns (uint64) {
        SpotTokenConfig config = readSpotTokenConfig();
        ICompositeExchangeReadOnly vault = ICompositeExchangeReadOnly(vaultAddress);
        (uint128 bal,, ) = vault.getBalance(sellNodeData.client(), config.fromTokenId());
        uint64 maxQuantity = uint64(bal / 10**(config.toVaultPdDiff()));
        if (maxQuantity == 0) {
            return 0;
        }
        return maxQuantity >= sellNodeData.quantity() ? sellNodeData.quantity() : maxQuantity;
    }
```

**Logic:**
1. **Get Balance**: Check seller's balance of "from" token (token being sold)
2. **Simple Check**: Return minimum of order quantity and available balance

**Example**: Selling 2 ETH
- Order: 2 ETH
- Seller balance: 1.5 ETH
- Result: 1.5 ETH (maximum matchable)

---

## Trade Settlement

### Seller-Initiated Trade

```57:64:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function settleSellerInitiatedTrade(uint64 matchQuantity, SpotOrder sellOrder, SpotNode buyNodeData,
        uint64 sellerFeeSoFar, uint /* pricingData */) internal override returns (SpotMatchQuantities) {
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        MakerSpotMatch makerSpotMatch = MakerSpotMatchLib.newMakerSpotMatch(buyNodeData, true, sellOrder.liquidation().isLiquidation());
        TakerSpotMatch takerSpotMatch = TakerSpotMatchLib.newTakerSpotMatch(sellOrder.client(), matchQuantity, sellerFeeSoFar);
        SpotMatchQuantities result = v.settleSpotMakerBuyer(makerSpotMatch, takerSpotMatch);
        return result;
    }
```

**Settlement Flow:**
1. **Create Match Data**: Packages maker (buyer) and taker (seller) match information
2. **Call Exchange**: `settleSpotMakerBuyer()` handles token transfers
3. **Returns Quantities**: Net quantities after fees

**What `settleSpotMakerBuyer` Does** (in CompositeExchange):
- Releases buyer's sequestered "to" token
- Transfers "from" token: Seller → Buyer (minus buyer fee)
- Transfers "to" token: Buyer → Seller (minus seller fee)
- Transfers fees: Both parties → USER_OPS
- Calculates fees and handles decimal conversions

### Buyer-Initiated Trade

```93:100:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function settleBuyerInitiatedTrade(uint64 matchQuantity, SpotOrder buyOrder, SpotNode sellNodeData,
        uint64 buyerFeeSoFar, uint /*pricingData*/) internal override returns (SpotMatchQuantities) {
        ICompExchForBooks v = ICompExchForBooks(msg.sender);
        MakerSpotMatch makerSpotMatch = MakerSpotMatchLib.newMakerSpotMatch(sellNodeData, false, buyOrder.liquidation().isLiquidation());
        TakerSpotMatch takerSpotMatch = TakerSpotMatchLib.newTakerSpotMatch(buyOrder.client(), matchQuantity, buyerFeeSoFar);
        SpotMatchQuantities result = v.settleSpotMakerSeller(makerSpotMatch, takerSpotMatch);
        return result;
    }
```

**Difference:**
- Uses `settleSpotMakerSeller()` instead
- Maker is seller, taker is buyer
- Fee calculations slightly different (maker vs taker roles)

---

## Order Quantity Validation

```51:55:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function checkOrderQuantity(SpotOrder orderData) internal override view {
        if (!orderData.liquidation().isLiquidation()) {
            require(orderData.quantity() >= readMinOrderQuantity(), ExchangeErrors.QuantityTooLow());
        }
    }
```

**Validation:**
- **Regular Orders**: Must be ≥ `minOrderQuantity`
- **Liquidation Orders**: No minimum (allows liquidating small positions)

**Difference from Perps:**
- Spot: `>= minOrderQuantity` (at least minimum)
- Perp: `% minOrderQuantity == 0` (exact multiple)

---

## Pricing Data

```31:33:BaseDEX/contracts/src/main/sol/SpotOrderBook.sol
    function gatherPricingData(address) internal pure override returns (uint) {
        return 0;
    }
```

**Purpose**: Overrides abstract function in `TwoTokenOrderBook`.

**Why Return 0?**
- Spot trading doesn't need mark prices for matching
- Matching uses order prices directly
- Perps need mark prices for unrealized P&L and balance checks

**Usage**: `pricingData` is passed to settlement functions but unused for spot.

---

## Comparison: Spot vs Perp

| Aspect | SpotOrderBook | PerpOrderBook |
|--------|----------------|----------------|
| **Sequestration** | Actual tokens (`sequester`) | Base token notional (`sequesterPerp`) |
| **Sell Order** | Locks "from" token | Locks base token notional |
| **Buy Order** | Locks "to" token (price × quantity) | Locks base token notional |
| **Settlement** | Token transfer (`settleSpotMaker*`) | Position creation (`settlePerpMaker*`) |
| **Balance Check** | Simple token balance | Mark price + fees + P&L |
| **Pricing Data** | Not needed (0) | Mark price required |
| **Order Quantity** | `>= minOrderQuantity` | `% minOrderQuantity == 0` |
| **Physical Delivery** | Yes (actual tokens) | No (cash-settled positions) |

---

## Example Trade Flow

### Scenario: User sells 1 ETH for 3000 USDC

1. **User Places Order**: 
   - `newSpotSellOrder()` with quantity=1 ETH, price=3000 USDC/ETH

2. **Orderbook Sequesters**: 
   - `sellSequester()` sequesters 1 ETH from user

3. **Matching**:
   - Order matched against buy order at 3000 USDC/ETH
   - `checkBuyBalance()` verifies buyer has 3000 USDC

4. **Settlement**:
   - `settleSpotMakerBuyer()` executes:
     - Releases buyer's sequestered 3000 USDC
     - Transfers 1 ETH: Seller → Buyer (minus buyer fee)
     - Transfers 3000 USDC: Buyer → Seller (minus seller fee)
     - Transfers fees to USER_OPS

5. **Result**:
   - Seller receives: ~2997 USDC (after fees)
   - Buyer receives: ~0.999 ETH (after fees)
   - Exchange receives fees from both parties

---

## Token Conversion and Decimals

### Decimal Handling

Spot trading handles multiple decimal systems:
- **Native Token Decimals**: Actual ERC20 token decimals
- **Vault Decimals**: Internal vault representation
- **Position Decimals**: Trading/position representation

**Conversion Functions:**
- `convertFromToTo()`: Converts quantity from "from" token to "to" token using price
- `ftpdDiff()`: Difference in position decimals between tokens
- `toVaultPdDiff()`: Difference between vault and position decimals

**Example**: Trading ETH (18 decimals) for USDC (6 decimals)
- Order quantity: 1 ETH (in position decimals, e.g., 10^8)
- Price conversion accounts for different decimal precisions
- Vault conversion handles internal representation

---

## Summary

`SpotOrderBook.sol` implements spot trading:

- **Token Sequestration**: Locks actual tokens (not notional)
  - Sellers: Lock "from" token
  - Buyers: Lock "to" token (price × quantity)
- **Simple Balance Checks**: Verifies token balances without mark price
- **Physical Delivery**: Transfers actual tokens between users
- **Fee Handling**: Fees calculated and transferred during settlement
- **Order Validation**: Minimum quantity requirements (≥ minOrderQuantity)

This provides decentralized spot trading with immediate token delivery, proper fund sequestration, and fee handling, enabling users to trade token pairs directly on-chain.