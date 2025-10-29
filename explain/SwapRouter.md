Searching for `SwapRouter.sol`.


[2 tools called]

Reading `SwapRouter.sol` to understand its implementation:


[3 tools called]

Reviewing related files for context:


[1 tool called]

Explanation of `SwapRouter.sol`:

## **SwapRouter.sol - Explanation**

### **Purpose and importance**

`SwapRouter` is a Uniswap V3-compatible swap interface for BaseDEX. It provides a simple API for swapping tokens against the base token (USD) on BaseDEX's spot orderbook.

### **Design**

1. Uniswap V3 compatibility: Implements `exactInputSingle` and `exactOutputSingle` to match Uniswap V3's interface.
2. Single account pattern: Creates one account on the exchange during construction and uses it for all swaps.
3. Decimal conversion: Handles conversions between ERC20 decimals, vault decimals, and position decimals.
4. Price estimation: Uses `PriceHelper` to estimate prices before executing swaps.

### **Key components**

#### **1. Constructor and initialization**

```12:25:BaseDEX/contracts/src/main/sol/SwapRouter.sol
contract SwapRouter {
    address public immutable exchangeAddress;
    address immutable priceHelperAddress;
    address immutable ops;
    uint64 public immutable accountId;
    uint32 constant BASE_TOKEN_ID = 1;

    constructor (address _exchangeAddress, address _priceHelperAddress, address _ops) {
        exchangeAddress = _exchangeAddress;
        priceHelperAddress = _priceHelperAddress;
        ops = _ops;
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        accountId = exch.createAccount();
    }
```

- Creates a persistent account on the exchange used for all swaps.
- Stores immutable references to the exchange, price helper, and operations address.

#### **2. Core swap functions**

##### **`swapByAmountInViaMinOut` (specify input amount)**

```124:181:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    // specify amountIn, amountOutMin. price is not specified\
    // swap is always between a token and base token (nominally usd)\
    // all amounts are in 64 bit position decimals as defined by the exchange\
    // if isBuy is true, amountIn represents base token value, otherwise, for a sell, amountIn represents the token being sold\
    // returns amountOut
    function swapByAmountInViaMinOut(SwapInputAmountIn orderParams) external returns (uint64) {
        return internalSwapByAmountInViaMinOut(orderParams, 0);
    }

    function internalSwapByAmountInViaMinOut(SwapInputAmountIn orderParams, uint inDelta) internal returns (uint64) {
        checkDeadline(orderParams.deadline());
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        (address spotBookAddress, ,) = exch.getSpotOrderBook(orderParams.tokenId(), BASE_TOKEN_ID);
        if (uint160(spotBookAddress) == 0) {
            revert SwapErrors.BadTokenId();
        }
        ISpotOrderBook spotBook = ISpotOrderBook(spotBookAddress);
        BestBidOffer bbo = spotBook.bestBidOffer();
        uint64 withdrawn = 0;
        if (orderParams.isBuy()) {
            // buy, specify in means amountIn is for the base token
            if (bbo.sellQuantity() == 0) {
                revert SwapErrors.CantSwapForAmount();
            }
            MarketOrderPriceResult mopr = priceIn(orderParams, bbo.sellPrice().times10(), PRICE_TYPE_BUY_IN, spotBookAddress);
            if (mopr.amountIn() > orderParams.amountIn()) {
                revert SwapErrors.CantSwapForAmount();
            }
            depositToExchange(BASE_TOKEN_ID, mopr.amountIn(), inDelta);
            SpotOrder order = SpotOrderLib.newSpotOrder(mopr.limitPrice(), mopr.orderQuantity(), accountId, ORDER_TYPE_FILL_ALL_OR_REVERT, LiquidationData.wrap(0));
            exch.newSpotBuyOrder(spotBookAddress, order);
            withdrawAllFromExchange(BASE_TOKEN_ID);
            withdrawn = withdrawAllFromExchange(orderParams.tokenId());
            if (withdrawn < orderParams.amountOutMin()) {
                revert SwapErrors.CantSwapForAmount();
            }
            transferToCaller(orderParams.tokenId(), withdrawn, 0);

        } else {
            // sell, specify in means amountIn is for the token being sold
            if (bbo.buyQuantity() == 0) {
                revert SwapErrors.CantSwapForAmount();
            }
            MarketOrderPriceResult mopr = priceIn(orderParams, bbo.buyPrice().divide10(), PRICE_TYPE_SELL_IN, spotBookAddress);
            if (mopr.orderQuantity() > orderParams.amountIn()) {
                revert SwapErrors.CantSwapForAmount();
            }
            depositToExchange(orderParams.tokenId(), mopr.orderQuantity(), inDelta);
            SpotOrder order = SpotOrderLib.newSpotOrder(mopr.limitPrice(), mopr.orderQuantity(), accountId, ORDER_TYPE_FILL_ALL_OR_REVERT, LiquidationData.wrap(0));
            exch.newSpotSellOrder(spotBookAddress, order);
            withdrawn = withdrawAllFromExchange(BASE_TOKEN_ID);
            if (withdrawn < orderParams.amountOutMin()) {
                revert SwapErrors.CantSwapForAmount();
            }
            transferToCaller(BASE_TOKEN_ID, withdrawn, 0);
        }
        return withdrawn;
    }
```

Flow:
1. Validate deadline and ensure the spot orderbook exists.
2. Get best bid/offer from the orderbook.
3. Estimate prices using `PriceHelper`.
4. Deposit input tokens to the exchange.
5. Place a market order (`ORDER_TYPE_FILL_ALL_OR_REVERT`).
6. Withdraw output tokens.
7. Transfer output tokens to the caller.

##### **`swapByAmountOutViaMaxIn` (specify output amount)**

```183:242:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    // specify amountOut (exact) and restrict amountIn via amountInMax\
    // if isBuy is true, amountOut represents token value being bought, otherwise, for a sell, amountOut represents the base token (nominally USD)\
    // swap is always between a token and base token (nominally USD)\
    // all amounts are in 64 bit position decimals as defined by the exchange\
    // returns amountIn
    function swapByAmountOutViaMaxIn(SwapInputAmountOut orderParams) external returns (uint64) {
        return internalSwapByAmountOutViaMaxIn(orderParams, 0);
    }

    function internalSwapByAmountOutViaMaxIn(SwapInputAmountOut orderParams, uint256 outDelta) internal returns (uint64) {
        checkDeadline(orderParams.deadline());
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        (address spotBookAddress, ,) = exch.getSpotOrderBook(orderParams.tokenId(), BASE_TOKEN_ID);
        if (uint160(spotBookAddress) == 0) {
            revert SwapErrors.BadTokenId();
        }
        ISpotOrderBook spotBook = ISpotOrderBook(spotBookAddress);
        BestBidOffer bbo = spotBook.bestBidOffer();
        uint64 amountIn = 0;
        if (orderParams.isBuy()) {
            // buy, specify in means amountOut is for the token
            if (bbo.sellQuantity() == 0) {
                revert SwapErrors.CantSwapForAmount();
            }
            MarketOrderPriceResult mopr = priceOut(orderParams, bbo.sellPrice().times10(), PRICE_TYPE_BUY_OUT, spotBookAddress);
            if (mopr.amountIn() > orderParams.amountInMax()) {
                revert SwapErrors.CantSwapForAmount();
            }
            amountIn = mopr.amountIn();
            depositToExchange(BASE_TOKEN_ID, mopr.amountIn(), 0);
            SpotOrder order = SpotOrderLib.newSpotOrder(mopr.limitPrice(), mopr.orderQuantity(), accountId, ORDER_TYPE_FILL_ALL_OR_REVERT, LiquidationData.wrap(0));
            exch.newSpotBuyOrder(spotBookAddress, order);
            withdrawAllFromExchange(BASE_TOKEN_ID);
            uint64 withdrawn = withdrawAllFromExchange(orderParams.tokenId());
            if (withdrawn < orderParams.amountOut()) {
                revert SwapErrors.CantSwapForAmount();
            }
            transferToCaller(orderParams.tokenId(), orderParams.amountOut(), outDelta);

        } else {
            // sell, specify in means amountIn is for the token being sold
            if (bbo.buyQuantity() == 0) {
                revert SwapErrors.CantSwapForAmount();
            }
            MarketOrderPriceResult mopr = priceOut(orderParams, bbo.buyPrice().divide10(), PRICE_TYPE_SELL_OUT, spotBookAddress);
            if (mopr.orderQuantity() > orderParams.amountInMax()) {
                revert SwapErrors.CantSwapForAmount();
            }
            amountIn = mopr.orderQuantity();
            depositToExchange(orderParams.tokenId(), mopr.orderQuantity(), 0);
            SpotOrder order = SpotOrderLib.newSpotOrder(mopr.limitPrice(), mopr.orderQuantity(), accountId, ORDER_TYPE_FILL_ALL_OR_REVERT, LiquidationData.wrap(0));
            exch.newSpotSellOrder(spotBookAddress, order);
            uint64 withdrawn = withdrawAllFromExchange(BASE_TOKEN_ID);
            if (withdrawn < orderParams.amountOut()) {
                revert SwapErrors.CantSwapForAmount();
            }
            transferToCaller(BASE_TOKEN_ID, orderParams.amountOut(), outDelta);
        }
        return amountIn;
    }
```

Similar flow, but:
- Specifies exact output amount.
- Caps input amount via `amountInMax`.
- Returns the actual input amount used.

#### **3. Uniswap V3-compatible functions**

##### **`exactInputSingle`**

```305:339:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    // one of the two tokens must be the exchange's base currency (nominally USD)\
    // sqrtPriceLimitX96 is not supported and must be zero\
    // recipient and fee values are ignored\
    function exactInputSingle(ExactInputSingleParams calldata params) external returns (uint256 amountOut) {
        checkDeadline(uint64(params.deadline));
        require(params.sqrtPriceLimitX96 == 0, SwapErrors.SqrtPriceNotSupported());
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        VaultTokenConfig tokenInConfig = exch.getDefaultErc20TokenConfig(params.tokenIn);
        VaultTokenConfig tokenOutConfig = exch.getDefaultErc20TokenConfig(params.tokenOut);
        if (tokenInConfig.tokenId() == tokenOutConfig.tokenId()) {
            revert SwapErrors.BadTokenId();
        }
        bool isBuy = false;
        uint32 tokenId;
        if (tokenInConfig.tokenId() == BASE_TOKEN_ID) {
            // this is a buy
            isBuy = true;
            tokenId = tokenOutConfig.tokenId();
        } else if (tokenOutConfig.tokenId() == BASE_TOKEN_ID) {
            // this is a sell
            tokenId = tokenInConfig.tokenId();
        } else {
            revert SwapErrors.BadTokenId();
        }
        uint powTen = 10**(tokenOutConfig.erc20Decimals() - tokenOutConfig.positionDecimals());
        uint64 amtOut = uint64(params.amountOutMinimum / powTen);
        if (params.amountOutMinimum % powTen != 0) {
            amtOut += 1;
        }
        uint64 amtIn = uint64(params.amountIn/10**(tokenInConfig.erc20Decimals() - tokenInConfig.positionDecimals()));

        return internalSwapByAmountInViaMinOut(SwapInputAmountInLib.create(amtIn,
            amtOut, uint64(params.deadline), tokenId, isBuy),
            params.amountIn % (10**(tokenInConfig.erc20Decimals() - tokenInConfig.positionDecimals()))) * powTen;
    }
```

- Converts ERC20 amounts to position decimals.
- Handles remainder (`inDelta`) for precision.
- Delegates to `internalSwapByAmountInViaMinOut`.

##### **`exactOutputSingle`**

```341:378:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    // one of the two tokens must be the exchange's base currency (nominally USD)\
    // sqrtPriceLimitX96 is not supported and must be zero\
    // recipient and fee values are ignored\
    // must not send any eth\
    function exactOutputSingle(ExactOutputSingleParams calldata params) external payable returns (uint256 amountIn) {
        checkDeadline(uint64(params.deadline));
        require(params.sqrtPriceLimitX96 == 0, SwapErrors.SqrtPriceNotSupported());
        require(msg.value == 0, SwapErrors.NativeCurrencyNotSupported());
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        VaultTokenConfig tokenInConfig = exch.getDefaultErc20TokenConfig(params.tokenIn);
        VaultTokenConfig tokenOutConfig = exch.getDefaultErc20TokenConfig(params.tokenOut);
        if (tokenInConfig.tokenId() == tokenOutConfig.tokenId()) {
            revert SwapErrors.BadTokenId();
        }
        bool isBuy = false;
        uint32 tokenId;
        if (tokenInConfig.tokenId() == BASE_TOKEN_ID) {
            // this is a buy
            tokenId = tokenOutConfig.tokenId();
            isBuy = true;
        } else if (tokenOutConfig.tokenId() == BASE_TOKEN_ID) {
            // this is a sell
            tokenId = tokenInConfig.tokenId();
        } else {
            revert SwapErrors.BadTokenId();
        }
        uint inPowTen = 10**(tokenInConfig.erc20Decimals() - tokenInConfig.positionDecimals());
        uint64 amtIn = uint64(params.amountInMaximum/ inPowTen);
        uint outPowTen = 10**(tokenOutConfig.erc20Decimals() - tokenOutConfig.positionDecimals());
        uint64 amtOut = uint64(params.amountOut/outPowTen);
        uint outDelta = 0;
        if (params.amountOut % outPowTen != 0) {
            amtOut += 1;
            outDelta = outPowTen - (params.amountOut % outPowTen);
        }
        return internalSwapByAmountOutViaMaxIn(SwapInputAmountOutLib.create(amtOut,
            amtIn, uint64(params.deadline), tokenId, isBuy), outDelta) * inPowTen;
    }
```

- Handles exact output amounts and remainder (`outDelta`).
- Delegates to `internalSwapByAmountOutViaMaxIn`.

#### **4. Price estimation functions**

##### **`priceIn` and `priceOut`**

```33:63:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    function priceIn(SwapInputAmountIn orderParams, Price59EN5 price, uint8 priceType, address bookAddress) internal returns (MarketOrderPriceResult) {
        uint[] memory encoded = new uint[](2);
        PriceAddressSpec ps = PriceAddressSpecLib.create(bookAddress, priceType);
        MarketOrderPrice mop = MarketOrderPriceLib.create(orderParams.amountIn(), 0, price, false);
        encoded[0] = ps.raw();
        encoded[1] = mop.raw();
        PriceHelper priceHelper = PriceHelper(priceHelperAddress);
        uint256[] memory result = priceHelper.estimatePrices(exchangeAddress, encoded);
        MarketOrderPriceResult mopr = MarketOrderPriceResult.wrap(result[0]);
        if (mopr.amountOut() < orderParams.amountOutMin()) {
            revert SwapErrors.CantSwapForAmount();
        }
        priceHelper.clear();
        return mopr;
    }

    function priceOut(SwapInputAmountOut orderParams, Price59EN5 price, uint8 priceType, address bookAddress) internal returns (MarketOrderPriceResult) {
        uint[] memory encoded = new uint[](2);
        PriceAddressSpec ps = PriceAddressSpecLib.create(bookAddress, priceType);
        MarketOrderPrice mop = MarketOrderPriceLib.create(0, orderParams.amountOut(), price, false);
        encoded[0] = ps.raw();
        encoded[1] = mop.raw();
        PriceHelper priceHelper = PriceHelper(priceHelperAddress);
        uint256[] memory result = priceHelper.estimatePrices(exchangeAddress, encoded);
        MarketOrderPriceResult mopr = MarketOrderPriceResult.wrap(result[0]);
        if (mopr.amountIn() > orderParams.amountInMax()) {
            revert SwapErrors.CantSwapForAmount();
        }
        priceHelper.clear();
        return mopr;
    }
```

- Use `PriceHelper` to estimate prices without modifying state.
- Validate that output meets minimum or input meets maximum.

#### **5. Decimal conversion helpers**

```91:122:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    function depositToExchange(uint32 tokenId, uint64 amount, uint256 delta) internal {
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        VaultTokenConfig tokenConfig = exch.readTokenConfig(tokenId);
        IERC20 erc20 = IERC20(tokenConfig.tokenAddress());
        uint ercAmount = uint(amount) * 10**(tokenConfig.erc20Decimals() - tokenConfig.positionDecimals());
        _erc20TransferFrom(erc20, address(this), ercAmount + delta);
        erc20.approve(exchangeAddress, ercAmount);
        exch.depositErc20(tokenConfig.tokenAddress(), ercAmount);
    }

    function checkDeadline(uint64 deadline) internal view {
        if (block.timestamp > deadline) {
            revert SwapErrors.DeadlinePassed();
        }
    }

    function withdrawAllFromExchange(uint32 tokenId) internal returns (uint64) {
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        VaultTokenConfig tokenConfig = exch.readTokenConfig(tokenId);
        (uint128 balVaultDeci, , ) = exch.getBalance(accountId, tokenId);
        uint erc20Amount = balVaultDeci * 10**(tokenConfig.erc20Decimals() - tokenConfig.vaultDecimals());
        exch.withdrawErc20(tokenConfig.tokenAddress(), erc20Amount);
        return tokenConfig.convertVaultToPosition(balVaultDeci);
    }

    function transferToCaller(uint32 tokenId, uint64 amount, uint256 delta) internal {
        ICompositeExchangePub exch = ICompositeExchangePub(exchangeAddress);
        VaultTokenConfig tokenConfig = exch.readTokenConfig(tokenId);
        IERC20 erc20 = IERC20(tokenConfig.tokenAddress());
        uint erc20Amount = amount * 10**(tokenConfig.erc20Decimals() - tokenConfig.positionDecimals()) - delta;
        _erc20Transfer(erc20, msg.sender, erc20Amount);
    }
```

These handle conversions between decimals:
- `depositToExchange`: Converts position decimals → ERC20 decimals, handles remainder.
- `withdrawAllFromExchange`: Converts vault decimals → ERC20 decimals → position decimals.
- `transferToCaller`: Converts position decimals → ERC20 decimals, adjusts for remainder.

#### **6. ERC20 transfer helpers**

```66:89:BaseDEX/contracts/src/main/sol/SwapRouter.sol
    function callIgnoreReturn(IERC20 erc20, bytes memory data) private {
        assembly ("memory-safe") {
            let success := call(gas(), erc20, 0, add(data, 0x20), mload(data), 0, 0x20)
            // bubble errors, but otherwise ignore return value.
            // balances are checked instead.
            if iszero(success) {
                let ptr := mload(0x40)
                returndatacopy(ptr, 0, returndatasize())
                revert(ptr, returndatasize())
                }
            }
        }

    function _erc20Transfer(IERC20 erc20, address dest, uint256 amount) internal {
        uint balBefore = erc20.balanceOf(dest);
        callIgnoreReturn(erc20, abi.encodeCall(erc20.transfer, (dest, amount)));
        require(balBefore + amount == erc20.balanceOf(dest), SwapErrors.Erc20TransferFailed());
    }

    function _erc20TransferFrom(IERC20 erc20, address dest, uint256 amount) internal {
        uint balBefore = erc20.balanceOf(dest);
        callIgnoreReturn(erc20, abi.encodeCall(erc20.transferFrom, (msg.sender, dest, amount)));
        require(balBefore + amount == erc20.balanceOf(dest), SwapErrors.Erc20TransferFailed());
    }
```

- Uses assembly for gas efficiency.
- Validates transfers via balance checks.

### **Key features**

1. Only supports swaps between a token and the base token (USD).
2. Uses `ORDER_TYPE_FILL_ALL_OR_REVERT` (executes immediately or reverts).
3. Handles decimal precision and remainders.
4. Uniswap V3-compatible interface for easier integration.
5. Price estimation before execution to avoid slippage failures.

### **Limitations**

1. `sqrtPriceLimitX96` not supported (must be zero).
2. `fee` and `recipient` parameters are ignored.
3. Native ETH not supported (`msg.value` must be zero).
4. Swaps only between token and base token (no direct token-to-token).

### **Usage pattern**

1. User calls `exactInputSingle` or `exactOutputSingle` with ERC20 addresses.
2. Router converts ERC20 decimals to position decimals.
3. Estimates price using `PriceHelper`.
4. Deposits input tokens to the exchange.
5. Places a market order on the spot orderbook.
6. Withdraws output tokens.
7. Transfers output tokens to the user.

### **Relationship to other contracts**

- `CompositeExchange`: Core exchange contract for deposits, withdrawals, and orders.
- `PriceHelper`: Price estimation using temporary storage.
- `SpotOrderBook`: Executes the actual trades.
- `PublicStruct`: Compact struct definitions for swap parameters.

In summary, `SwapRouter` provides a user-friendly, Uniswap V3-compatible interface for swapping tokens on BaseDEX's spot orderbook, handling decimal conversions, price estimation, and order execution.