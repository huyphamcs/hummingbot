# Hummingbot: Comprehensive Terminology and Concepts Guide

**Version:** 2.11.0
**Date:** January 2026
**Author:** Hummingbot Foundation

---

## Table of Contents

1. [Fundamental Trading Concepts](#fundamental-trading-concepts)
   - [Trading Pairs](#trading-pairs)
   - [Order Book (CLOB)](#order-book-clob)
   - [Order Types](#order-types)
   - [Trade Types](#trade-types)
   - [Price Calculation Methods](#price-calculation-methods)
2. [Trading Strategies](#trading-strategies)
   - [Market Making](#market-making)
   - [Arbitrage](#arbitrage)
   - [Cross-Exchange Market Making (XEMM)](#cross-exchange-market-making-xemm)
3. [Hummingbot Architecture](#hummingbot-architecture)
   - [Connector System](#connector-system)
   - [Clock System](#clock-system)
   - [Strategy V2 Architecture](#strategy-v2-architecture)
   - [Market Data Provider](#market-data-provider)
   - [Configuration System](#configuration-system)
   - [Database Persistence](#database-persistence)
4. [Advanced Concepts](#advanced-concepts)
   - [Risk Management](#risk-management)
   - [Performance Metrics](#performance-metrics)
   - [Order Lifecycle](#order-lifecycle)
   - [Event System](#event-system)

---

## FUNDAMENTAL TRADING CONCEPTS

### Trading Pairs

**Definition:** A trading pair represents two assets that can be exchanged for each other.

**Format:** `BASE-QUOTE` or `BASE/QUOTE`

**Components:**
- **Base Asset**: The asset you're buying or selling (left side)
  - Example: In `BTC-USDT`, BTC is the base asset
- **Quote Asset**: The asset used to price the base asset (right side)
  - Example: In `BTC-USDT`, USDT is the quote asset
  - If BTC price is 50,000 USDT, you need 50,000 USDT to buy 1 BTC

**Code Example:**
```python
trading_pair = "BTC-USDT"
base, quote = split_hb_trading_pair(trading_pair)
# base = "BTC", quote = "USDT"
```

---

### Order Book (CLOB)

**CLOB = Central Limit Order Book**

**Definition:** A digital ledger that records all buy and sell orders for a trading pair, organized by price level.

**Structure Example:**
```
ORDER BOOK for BTC-USDT
═══════════════════════════════════
ASKS (Sell Orders) - People wanting to SELL BTC
Price      | Amount  | Total
50,200 USDT| 0.5 BTC | 25,100 USDT
50,150 USDT| 1.2 BTC | 60,180 USDT
50,100 USDT| 0.8 BTC | 40,080 USDT
───────────────────────────────────
50,050 USDT ← Mid Price
───────────────────────────────────
BIDS (Buy Orders) - People wanting to BUY BTC
Price      | Amount  | Total
50,000 USDT| 1.0 BTC | 50,000 USDT
49,950 USDT| 0.7 BTC | 34,965 USDT
49,900 USDT| 2.0 BTC | 99,800 USDT
```

#### Key Terminology

**1. BID (Buy Order)**
- An order to BUY the base asset
- Represents demand
- Sorted from highest to lowest price
- **Best Bid**: The highest price someone is willing to pay (50,000 USDT in example)

**2. ASK (Sell Order)**
- An order to SELL the base asset
- Represents supply
- Sorted from lowest to highest price
- **Best Ask**: The lowest price someone is willing to sell at (50,100 USDT in example)

**3. SPREAD**
- The difference between Best Ask and Best Bid
- Formula: `Spread = Best Ask - Best Bid`
- Example: `50,100 - 50,000 = 100 USDT`
- **Narrower spread** = more liquid market
- **Wider spread** = less liquid market

**4. MID PRICE**
- The average of best bid and best ask
- Formula: `Mid Price = (Best Bid + Best Ask) / 2`
- Example: `(50,000 + 50,100) / 2 = 50,050 USDT`
- Used as reference point for placing orders

**5. LIQUIDITY**
- The amount of assets available to trade at various price levels
- **High Liquidity**: Many orders, easy to trade without moving price
- **Low Liquidity**: Few orders, trades significantly move price
- Measured by:
  - **Order Book Depth**: Total volume within a price range
  - **Bid/Ask Volume**: Total amount on each side

**Implementation:**
```python
class OrderBook:
    _bid_book: set[OrderBookEntry]  # All bid orders sorted by price
    _ask_book: set[OrderBookEntry]  # All ask orders sorted by price
    _best_bid: double               # Cached best bid price
    _best_ask: double               # Cached best ask price

    def get_price(is_buy):
        # Returns best_ask if buying, best_bid if selling

    def apply_diffs(bids, asks, update_id):
        # Apply incremental order book updates

    def simulate_buy(amount):
        # Calculate cost to buy given amount
```

---

### Order Types

#### 1. MARKET ORDER
```python
OrderType.MARKET
```
- **Executes immediately** at the best available price
- **Taker Order**: Takes liquidity from the order book
- **Advantages**: Guaranteed execution, fast
- **Disadvantages**: Price uncertainty (slippage)
- **Example**: "Buy 1 BTC at whatever the best price is right now"

#### 2. LIMIT ORDER
```python
OrderType.LIMIT
```
- **Executes only at specified price or better**
- Can be **Maker** (adds liquidity) or **Taker** (removes liquidity)
- **Maker Order**: Order sits in order book waiting to be filled
- **Advantages**: Price control, often lower fees
- **Disadvantages**: May not execute if price never reached
- **Example**: "Buy 1 BTC at 50,000 USDT (or lower)"

#### 3. LIMIT MAKER ORDER
```python
OrderType.LIMIT_MAKER
```
- Limit order that **only executes as a maker**
- Rejected if it would execute immediately (as taker)
- **Purpose**: Guarantee maker fees (usually lower than taker fees)

#### 4. AMM SWAP
```python
OrderType.AMM_SWAP
```
- Special order type for Automated Market Makers (DEXs)
- Uses liquidity pools instead of order books

---

### Trade Types

#### A. Spot Trading
- **Direct exchange of assets**
- Immediate settlement
- You own the actual asset
- **Example**: Buy 1 BTC with 50,000 USDT → you now own 1 BTC

#### B. Perpetual Futures (Perpetuals/Perps)
- **Derivative contracts** with no expiration date
- **Leverage**: Trade with borrowed funds
  - Example: 10x leverage means $1,000 controls $10,000 position
- **Long Position**: Betting price will go UP
- **Short Position**: Betting price will go DOWN
- **Funding Rate**: Periodic payment between longs and shorts
  - Positive funding: Longs pay shorts (more people are long)
  - Negative funding: Shorts pay longs (more people are short)

**Position Modes:**

**1. One-Way Mode (ONEWAY)**
```python
PositionMode.ONEWAY
```
- Can only be long OR short at a time
- Opening opposite position closes current position
- Example: Long 1 BTC, then sell 1 BTC = position closed

**2. Hedge Mode (HEDGE)**
```python
PositionMode.HEDGE
```
- Can be long AND short simultaneously
- Each side managed separately
- Example: Long 1 BTC AND Short 0.5 BTC = net long 0.5 BTC

**Position Actions:**
```python
PositionAction.OPEN   # Open new position
PositionAction.CLOSE  # Close existing position
PositionAction.NIL    # No position change (for spot)
```

---

### Price Calculation Methods

#### A. VWAP (Volume-Weighted Average Price)
- Average price **weighted by volume** traded at each price level
- **Formula**: `VWAP = Σ(Price × Volume) / Σ(Volume)`

**Example:** To buy 2 BTC:
```
Price      | Available | You take | Cost
50,100 USDT| 0.8 BTC  | 0.8 BTC  | 40,080 USDT
50,150 USDT| 1.2 BTC  | 1.2 BTC  | 60,180 USDT
────────────────────────────────────────────
Total: 2.0 BTC, Cost: 100,260 USDT
VWAP = 100,260 / 2.0 = 50,130 USDT per BTC
```

#### B. TWAP (Time-Weighted Average Price)
- Average price over a **time period**
- Divides large order into smaller orders over time
- **Purpose**: Reduce market impact, avoid moving price

**Example:**
```
Want to buy 10 BTC over 1 hour:
00:00 - Buy 2 BTC at 50,000 USDT
00:15 - Buy 2 BTC at 50,100 USDT
00:30 - Buy 2 BTC at 50,200 USDT
00:45 - Buy 2 BTC at 50,150 USDT
01:00 - Buy 2 BTC at 50,050 USDT
────────────────────────────────
TWAP = (50,000 + 50,100 + 50,200 + 50,150 + 50,050) / 5
     = 50,100 USDT
```

#### C. Price Types in Hummingbot
```python
class PriceType(Enum):
    MidPrice = 1        # (Best Bid + Best Ask) / 2
    BestBid = 2         # Highest buy order price
    BestAsk = 3         # Lowest sell order price
    LastTrade = 4       # Most recent trade price
    LastOwnTrade = 5    # Your last trade price
    InventoryCost = 6   # Average cost of your inventory
    Custom = 7          # User-defined calculation
```

---

## TRADING STRATEGIES

### Market Making

**Definition:** Provide liquidity by simultaneously offering to buy (bid) and sell (ask) an asset, profiting from the spread.

#### How It Works

**Step-by-Step Example:**
```
1. Current Market:
   Best Bid: 49,990 USDT
   Best Ask: 50,010 USDT
   Mid Price: 50,000 USDT

2. Market Maker Places Orders:
   BID ORDER:  Buy 1 BTC at 49,980 USDT (20 USDT below mid)
   ASK ORDER:  Sell 1 BTC at 50,020 USDT (20 USDT above mid)

   Spread captured: 50,020 - 49,980 = 40 USDT profit per BTC

3. Order Book Now Shows:
   ASKS:
   50,020 USDT | 1.0 BTC ← Your ask order
   50,010 USDT | 0.5 BTC
   ─────────────────────
   BIDS:
   49,990 USDT | 0.8 BTC
   49,980 USDT | 1.0 BTC ← Your bid order

4. When Both Orders Fill:
   - Bought 1 BTC for 49,980 USDT
   - Sold 1 BTC for 50,020 USDT
   - Profit: 40 USDT (minus fees)
```

#### Key Parameters

**1. Bid Spread / Ask Spread**
```yaml
bid_spread: 0.1  # 0.1% = Place bid 0.1% below mid price
ask_spread: 0.1  # 0.1% = Place ask 0.1% above mid price
```
- **Percentage distance** from mid price
- Smaller spread = more likely to fill, less profit per trade
- Larger spread = less likely to fill, more profit per trade

**2. Order Amount**
```yaml
order_amount: 1.0  # Size of each order in base asset
```
- How much to buy/sell in each order
- Affects capital requirements and risk

**3. Order Refresh Time**
```yaml
order_refresh_time: 60  # Seconds before canceling and replacing orders
```
- How often to update orders
- Shorter = respond faster to market changes, more API calls
- Longer = less overhead, slower response

**4. Inventory Skew**
```yaml
inventory_skew_enabled: true
inventory_target_base_pct: 50  # Target 50% base, 50% quote
```
- Adjusts order sizes to maintain target inventory ratio

**Example:**
```
Target: 50% BTC, 50% USDT
Current: 70% BTC, 30% USDT (too much BTC)

Adjustment:
- Increase sell order size (sell more BTC)
- Decrease buy order size (buy less BTC)

Result: Gradually rebalance toward 50/50
```

**5. Order Levels**
```yaml
order_levels: 3  # Place multiple orders at different prices
order_level_spread: 0.1  # Additional spread for each level
```

**Example:**
```
Level 1: Buy at 49,980 USDT (0.04% from mid)
Level 2: Buy at 49,930 USDT (0.14% from mid)
Level 3: Buy at 49,880 USDT (0.24% from mid)
```

**6. Ping Pong**
```yaml
ping_pong_enabled: true
```
- Alternate between buy and sell
- After buying, only place sell orders until sold
- **Purpose**: Avoid accumulating too much inventory

#### Implementation

```python
class PureMarketMakingStrategy:
    def c_tick(self, timestamp):
        # 1. Create base proposals
        proposal = self.c_create_base_proposal()

        # 2. Apply order level modifiers
        self.c_apply_order_levels_modifiers(proposal)

        # 3. Apply price modifiers
        self.c_apply_order_price_modifiers(proposal)

        # 4. Apply budget constraints
        self.apply_budget_constraint(proposal)

        # 5. Execute orders
        self.execute_orders_proposal(proposal)
```

---

### Arbitrage

**Definition:** Exploit price differences for the same asset across different markets/exchanges.

#### Types of Arbitrage

**A. Spatial Arbitrage (Cross-Exchange)**
```
Exchange A: BTC = 50,000 USDT
Exchange B: BTC = 50,200 USDT
Difference: 200 USDT per BTC

Strategy:
1. Buy BTC on Exchange A at 50,000 USDT
2. Sell BTC on Exchange B at 50,200 USDT
3. Profit: 200 USDT per BTC (minus fees and transfer costs)
```

**B. Statistical Arbitrage (Mean Reversion)**
- Trading pairs that are historically correlated
- When correlation breaks, bet on return to normal

**Example:**
```
ETH/BTC ratio usually around 0.06
Currently at 0.065 (ETH expensive relative to BTC)

Strategy:
- Sell ETH
- Buy BTC
- Wait for ratio to return to 0.06
```

**C. Triangular Arbitrage**
```
Convert through multiple trading pairs:
1. BTC → ETH at rate X
2. ETH → USDT at rate Y
3. USDT → BTC at rate Z

If X * Y * Z > 1.0, there's profit opportunity
```

#### Implementation

```python
class ArbitrageExecutor:
    buying_market: ConnectorPair   # Where to buy
    selling_market: ConnectorPair  # Where to sell
    order_amount: Decimal          # How much to arbitrage
    min_profitability: Decimal     # Minimum profit % to execute

    async def control_task(self):
        # Calculate current profitability
        await self.update_trade_pnl_pct()
        await self.update_tx_cost()

        profitability = (
            self._trade_pnl_pct * self.order_amount - self._last_tx_cost
        ) / self.order_amount

        # Execute if profitable
        if profitability > self.min_profitability:
            await self.execute_arbitrage()
```

---

### Cross-Exchange Market Making (XEMM)

**Definition:** Place maker orders on one exchange while hedging on another exchange.

#### How It Works

```
MAKER EXCHANGE (Less liquid, wider spreads):
Place orders: Buy at 49,900, Sell at 50,100
Earn: 200 USDT spread + maker rebate

TAKER EXCHANGE (More liquid):
When maker order fills:
- Maker buy filled at 49,900 → Immediately sell on taker at 50,000
- Maker sell filled at 50,100 → Immediately buy on taker at 50,000

Net per complete cycle:
- Maker spread: 200 USDT
- Taker loss: ~50-100 USDT (spread + fees)
- Net profit: 100-150 USDT per BTC
```

#### Key Parameters

```yaml
min_profitability: 0.2  # Minimum 0.2% profit required
order_amount: 1.0
adjust_order_enabled: true  # Adjust prices for profitability
```

---

## HUMMINGBOT ARCHITECTURE

### Connector System

**Definition:** A **connector** is a software module that translates Hummingbot's standardized API into exchange-specific API calls.

#### Why Connectors Are Needed

Each exchange has different:
- API endpoints
- WebSocket formats
- Order message structures
- Authentication methods
- Rate limits

#### Connector Types

**A. CLOB CEX (Central Limit Order Book - Centralized Exchange)**

**Characteristics:**
- Order book maintained by centralized entity
- Orders matched by exchange's matching engine
- **Custodial**: Exchange holds your funds

**Examples:** Binance, Coinbase, KuCoin, OKX, Gate.io

**Connection Method:**
```python
# API Keys required
api_keys = {
    "binance_api_key": "your_key",
    "binance_api_secret": "your_secret"
}

connector = create_connector(
    "binance",
    ["BTC-USDT"],
    trading_required=True,
    api_keys=api_keys
)
```

**B. CLOB DEX (Central Limit Order Book - Decentralized Exchange)**

**Characteristics:**
- On-chain order book (blockchain-based)
- **Non-custodial**: You control your private keys
- Orders signed with wallet, settled on blockchain

**Examples:** dYdX, Hyperliquid, Injective, XRPL

**Connection Method:**
```python
# Wallet private key required
api_keys = {
    "dydx_v4_perpetual_account_number": "0",
    "dydx_v4_perpetual_secret_phrase": "your mnemonic phrase..."
}
```

**C. AMM DEX (Automated Market Maker - Decentralized Exchange)**

**AMM = Automated Market Maker**

**Traditional Exchange (CLOB):**
```
Buyer matches with Seller via order book
Buy order: 1 BTC at 50,000 USDT
Sell order: 1 BTC at 50,000 USDT
→ Trade executed
```

**AMM:**
```
Trade against Liquidity Pool (smart contract)
Pool: 100 BTC + 5,000,000 USDT
Formula: x * y = k (constant product)

When you buy 1 BTC:
- Remove 1 BTC from pool
- Add USDT to pool (calculated to maintain k)
- Price determined by ratio in pool
```

**AMM Subtypes:**

**1. Traditional AMM (Constant Product)**
```
Formula: x * y = k
Where x = token A amount, y = token B amount, k = constant

Examples: Uniswap V2, PancakeSwap, SushiSwap
```
- Liquidity spread across all prices (0 to infinity)
- **Impermanent Loss**: Risk from providing liquidity

**2. CLMM (Concentrated Liquidity Market Maker)**
```
Examples: Uniswap V3, Raydium CLMM, Meteora
```
- Liquidity concentrated in specific price ranges

**Example:**
```
Instead of: 0 → ∞
Provide liquidity: 49,000 USDT → 51,000 USDT

Benefits:
- More capital efficient
- Higher fees in range
- Position goes inactive if price leaves range
```

**3. Router (DEX Aggregator)**
```
Examples: 0x Protocol, Jupiter
```
- Finds best price across multiple DEXs
- Splits order across exchanges for optimal execution

**Example:**
```
Want to buy 10 BTC:
- 4 BTC from Uniswap at 50,000 USDT
- 3 BTC from Sushiswap at 50,050 USDT
- 3 BTC from Curve at 50,100 USDT
Average: 50,045 USDT (better than single source)
```

#### Gateway Middleware

**Architecture:**
```
Hummingbot ←→ Gateway (TypeScript) ←→ Blockchain
```

**Gateway Handles:**
- Wallet management
- Transaction signing
- Gas estimation
- Chain-specific logic
- Smart contract interactions

**Connection:**
```yaml
# Uncomment in docker-compose.yml
gateway:
  restart: always
  container_name: gateway
  image: hummingbot/gateway:latest
  ports:
    - "15888:15888"
  environment:
    - GATEWAY_PASSPHRASE=admin
    - DEV=true  # Development mode (HTTP)
```

---

### Clock System

**Definition:** A **real-time event loop** that synchronizes all trading operations.

#### The "Tick" Concept

```
Tick = One iteration of the event loop
Default: 1 tick per second
```

#### What Happens Each Tick

```python
Clock.tick(timestamp):
    1. Update timestamp

    2. For each connector:
        connector.tick(timestamp)
        # - Fetches new order book data
        # - Updates balances
        # - Processes WebSocket messages

    3. For each strategy:
        strategy.tick(timestamp)
        # - Analyzes market
        # - Makes trading decisions
        # - Places/cancels orders

    4. For each executor:
        executor.tick(timestamp)
        # - Manages order lifecycle
        # - Updates position status
        # - Executes actions
```

#### Clock Modes

**1. REALTIME Mode**
```python
Clock(ClockMode.REALTIME, tick_size=1.0)
```
- Uses system time
- Sleeps until next tick

**Example:**
```
Current time: 12:00:00.000
Tick size: 1.0 second
Next tick: 12:00:01.000

12:00:01.000 → Tick 1
12:00:02.000 → Tick 2
12:00:03.000 → Tick 3
```

**2. BACKTEST Mode**
```python
Clock(ClockMode.BACKTEST,
      start_time=1640000000,
      end_time=1640086400)
```
- Uses historical data
- No sleeping, processes as fast as possible
- Simulates timestamps
- **Purpose**: Test strategies on past data

#### Implementation

```python
async def run(self):
    self._current_tick = (now // self._tick_size) * self._tick_size

    while True:
        now = time.time()

        # Calculate next tick time
        next_tick_time = ((now // self._tick_size) + 1) * self._tick_size

        # Sleep until next tick
        await asyncio.sleep(next_tick_time - now)

        self._current_tick = next_tick_time

        # Execute all child iterators
        for child_iterator in self._child_iterators:
            child_iterator.c_tick(self._current_tick)
```

---

### Strategy V2 Architecture

**Modern component-based design with separation of concerns.**

#### Component Hierarchy

```
StrategyV2Base (Top Level)
├── MarketDataProvider (Candles, Order Books)
├── ExecutorOrchestrator (Manages Executors)
│   └── Active Executors (Order/Position/Grid/DCA/TWAP/etc.)
└── Controllers (Trading Logic)
    ├── MarketMakingController
    ├── DirectionalTradingController
    └── GenericController
```

#### A. STRATEGY (Top Level)

```python
class StrategyV2Base:
    market_data_provider: MarketDataProvider
    executor_orchestrator: ExecutorOrchestrator
    controllers: List[ControllerBase]
```

**Responsibilities:**
- Coordinates all components
- Manages market data subscriptions
- Handles candle data feeds
- Orchestrates controller execution

#### B. CONTROLLER (Trading Logic)

**Definition:** High-level trading logic that analyzes market conditions and decides what actions to take.

```python
class ControllerBase:
    def determine_executor_actions() -> List[ExecutorAction]:
        # 1. Analyze market conditions
        # 2. Decide what to do
        # 3. Return list of actions
```

**What It Does:**

**1. Analyzes Market Data:**
- Candlestick data (OHLCV)
- Order book depth
- Technical indicators

**2. Makes Decisions:**
```python
if RSI < 30 and price < moving_average:
    return [CreateExecutorAction(
        executor_config=PositionExecutorConfig(
            side=TradeType.BUY,
            amount=1.0,
            take_profit=1.02,  # 2% profit target
            stop_loss=0.98     # 2% loss limit
        )
    )]
```

**3. Returns Actions:**
- `CreateExecutorAction`: Start new executor
- `StopExecutorAction`: Stop running executor
- `StoreExecutorAction`: Save executor state

**Controller Types:**

**1. Market Making Controller**
```python
class MarketMakingControllerBase:
    def create_actions_proposal():
        # Create maker orders at multiple levels
        levels = self.get_levels_to_execute()
        for level in levels:
            price, amount = self.get_price_and_amount(level)
            # Create executor for each level
```

**2. Directional Trading Controller**
```python
class DirectionalTradingControllerBase:
    def create_actions_proposal():
        # Analyze trend
        if bullish_signal():
            # Open long position
        elif bearish_signal():
            # Open short position
```

**3. Generic Controller**
- Flexible controllers for custom strategies
- Examples: Grid, Arbitrage, Hedge

#### C. EXECUTOR (Order Management)

**Definition:** A **self-contained order management unit** that handles the complete lifecycle of one or more orders.

**Base Executor Lifecycle:**
```python
class ExecutorBase(RunnableBase):
    status: RunnableStatus
    # NOT_STARTED → RUNNING → SHUTTING_DOWN → TERMINATED

    async def start():
        self.status = RunnableStatus.RUNNING
        # Begin control loop

    async def control_task():
        # Called repeatedly while RUNNING
        # Manage orders

    def stop():
        self.status = RunnableStatus.TERMINATED
        # Clean up
```

**Executor Types:**

**1. OrderExecutor (Single Order)**

**Purpose:** Manage a single order with various execution strategies

```python
class OrderExecutor(ExecutorBase):
    config: OrderExecutorConfig
    _order: TrackedOrder
```

**Execution Strategies:**

**A. Simple Limit**
```python
config = OrderExecutorConfig(
    connector_name="binance",
    trading_pair="BTC-USDT",
    side=TradeType.BUY,
    amount=Decimal("1.0"),
    order_type=OrderType.LIMIT,
    price=Decimal("50000"),
    execution_strategy=ExecutionStrategy.LIMIT
)
```
- Places limit order once
- Waits for fill
- Done

**B. Limit Chaser**
```python
config = OrderExecutorConfig(
    execution_strategy=ExecutionStrategy.LIMIT_CHASER,
    chaser_config=ChaserConfig(
        refresh_threshold=0.001  # 0.1% price movement triggers update
    )
)
```

**How Limit Chaser Works:**
```
Goal: Stay at top of order book without overpaying

1. Place buy order at 49,990 USDT
2. Market moves, best bid now 49,995 USDT
3. Our order no longer competitive
4. Cancel old order
5. Place new order at 49,996 USDT (just above best bid)
6. Repeat until filled

Benefits:
- Higher fill probability
- Better price than market order
- Responsive to market changes
```

**Implementation:**
```python
def control_limit_chaser(self):
    current_price = self.current_market_price
    threshold = self.config.chaser_config.refresh_threshold

    if self.config.side == TradeType.BUY:
        if current_price - self._order.order.price > (current_price * threshold):
            self.renew_order()  # Cancel and replace
```

**2. PositionExecutor (Full Position Management)**

**Purpose:** Manage a position with entry, take profit, stop loss, and time limit

```python
class PositionExecutor(ExecutorBase):
    open_order: TrackedOrder           # Entry order
    take_profit_order: TrackedOrder    # Exit with profit
    stop_loss_order: TrackedOrder      # Exit with loss
    time_limit_order: TrackedOrder     # Exit after time
```

**Triple Barrier Method:**
```
Price
  ↑
  │     TP ──────────────────── Take Profit (102% of entry)
  │
  │     ────────────────────── Entry Price (100%)
  │
  │     SL ──────────────────── Stop Loss (98% of entry)
  │
  └──────────────────────────→ Time
                          │
                    Time Limit (Exit after X seconds)

Whichever barrier is hit first closes the position
```

**Example Configuration:**
```python
config = PositionExecutorConfig(
    connector_name="binance_perpetual",
    trading_pair="BTC-USDT",
    side=TradeType.BUY,
    amount=Decimal("1.0"),

    # Entry
    entry_price=Decimal("50000"),

    # Exits (Triple Barrier)
    take_profit=Decimal("1.02"),      # Close at +2%
    stop_loss=Decimal("0.98"),        # Close at -2%
    time_limit=3600,                  # Close after 1 hour

    # Advanced
    leverage=Decimal("10"),           # 10x leverage
    trailing_stop_activation=Decimal("1.01"),   # Activate at +1%
    trailing_stop_trailing_delta=Decimal("0.005")  # Trail by 0.5%
)
```

**Trailing Stop Example:**
```
Entry: 50,000 USDT
Trailing Stop Activation: +1% = 50,500 USDT
Trailing Delta: 0.5% = 250 USDT

Price Movement:
50,000 → 50,500 USDT: Trailing stop ACTIVATED at 50,250
50,500 → 51,000 USDT: Stop moves up to 50,750
51,000 → 52,000 USDT: Stop moves up to 51,750
52,000 → 51,700 USDT: Stop TRIGGERED at 51,750!

Profit locked: +3.5% instead of fixed +2%
```

**3. GridExecutor (Grid Trading)**

**Purpose:** Place multiple orders at regular price intervals

```python
class GridExecutor(ExecutorBase):
    levels: List[GridLevel]
```

**Grid Trading Concept:**
```
Place buy and sell orders at regular intervals:

SELL 0.5 BTC at 52,000 ──┐
SELL 0.5 BTC at 51,000 ──┤
────────────────────────  ← Current Price: 50,000
BUY 0.5 BTC at 49,000 ───┤
BUY 0.5 BTC at 48,000 ───┘

As price oscillates:
- Buy when price drops (accumulate)
- Sell when price rises (take profit)
- Profit from volatility
```

**4. DCAExecutor (Dollar-Cost Averaging)**

**Purpose:** Buy fixed amount at regular intervals

```python
class DCAExecutor(ExecutorBase):
    orders_history: List[TrackedOrder]
```

**DCA Strategy:**
```
Buy fixed amount at regular intervals regardless of price:

Week 1: Buy $100 BTC at 50,000 USDT → 0.002 BTC
Week 2: Buy $100 BTC at 48,000 USDT → 0.00208 BTC
Week 3: Buy $100 BTC at 52,000 USDT → 0.00192 BTC
Week 4: Buy $100 BTC at 51,000 USDT → 0.00196 BTC

Total: $400 spent, 0.00796 BTC acquired
Average price: $400 / 0.00796 = $50,251 per BTC
```

**5. TWAPExecutor (Time-Weighted Average Price)**

**Purpose:** Split large order over time to minimize market impact

```python
class TWAPExecutor(ExecutorBase):
    target_quantity: Decimal      # Total amount to trade
    total_duration: int           # Time to spread over
    num_individual_orders: int    # Number of orders
```

**TWAP Strategy:**
```
Goal: Buy 10 BTC over 1 hour with minimal market impact

Split into 12 orders (every 5 minutes):
00:00 - Buy 0.833 BTC
00:05 - Buy 0.833 BTC
00:10 - Buy 0.833 BTC
...
00:55 - Buy 0.833 BTC

Why?
- Smaller orders don't move price as much
- Average out price volatility
- Less obvious to other traders
- Common for institutional trading
```

**6. ArbitrageExecutor**

**Purpose:** Execute arbitrage opportunities between markets

```python
class ArbitrageExecutor(ExecutorBase):
    buying_market: ConnectorPair
    selling_market: ConnectorPair
    buy_order: TrackedOrder
    sell_order: TrackedOrder
```

**7. XEMMExecutor (Cross-Exchange Market Making)**

**Purpose:** Market make on one exchange while hedging on another

```python
class XEMMExecutor(ExecutorBase):
    maker_connector: str
    taker_connector: str
```

#### D. EXECUTOR ORCHESTRATOR (Coordinator)

**Definition:** Manages all executors and tracks performance

```python
class ExecutorOrchestrator:
    active_executors: Dict[str, ExecutorBase]
    positions_held: Dict[str, PositionHold]
    cached_performance: Dict
```

**Responsibilities:**

**1. Manages Executor Lifecycle**
```python
def execute_actions(self, actions: List[ExecutorAction]):
    for action in actions:
        if isinstance(action, CreateExecutorAction):
            # Create new executor
            executor = self._create_executor(action.executor_config)
            executor.start()
            self.active_executors[executor.id] = executor

        elif isinstance(action, StopExecutorAction):
            # Stop executor
            executor = self.active_executors[action.executor_id]
            executor.stop()
```

**2. Tracks Positions**
```python
class PositionHold:
    """Represents a closed position being held for tracking"""
    connector_name: str
    trading_pair: str
    side: TradeType
    amount: Decimal
    entry_price: Decimal
    timestamp: float

    # Performance metrics
    cum_fees_quote: Decimal
    realized_pnl_quote: Decimal
```

**3. Calculates Performance**
```python
def get_position_summary(self, mid_price: Decimal) -> PositionSummary:
    """
    Calculates:
    - Realized PnL (from closed positions)
    - Unrealized PnL (from open positions)
    - Total fees paid
    - Return percentage
    """
```

---

### Market Data Provider

**Definition:** Centralized service for accessing market data

#### Features

**A. Candle Data (OHLCV)**
```python
class Candle:
    timestamp: int        # Unix timestamp
    open: Decimal        # Opening price
    high: Decimal        # Highest price
    low: Decimal         # Lowest price
    close: Decimal       # Closing price
    volume: Decimal      # Trading volume
    quote_volume: Decimal  # Volume in quote asset
```

**Example:**
```
1-Minute BTC-USDT Candle:
Timestamp: 2024-01-15 12:00:00
Open:  50,000 USDT  (price at start of minute)
High:  50,150 USDT  (highest price during minute)
Low:   49,950 USDT  (lowest price during minute)
Close: 50,100 USDT  (price at end of minute)
Volume: 125.5 BTC   (total BTC traded)
```

**B. Candle Configuration**
```python
CandlesConfig(
    connector="binance",
    trading_pair="BTC-USDT",
    interval="1m",         # 1m, 5m, 15m, 1h, 4h, 1d
    max_records=1000       # Keep last 1000 candles
)
```

**C. Technical Indicators**
- Moving Averages (SMA, EMA)
- RSI (Relative Strength Index)
- Bollinger Bands
- MACD (Moving Average Convergence Divergence)
- And more...

---

### Configuration System

**Pydantic-Based Type-Safe Configuration**

#### Why Pydantic

- Runtime type checking
- Automatic validation
- IDE autocomplete
- Self-documenting

#### Example

```python
from pydantic import BaseModel, Field
from decimal import Decimal

class MyStrategyConfig(BaseModel):
    # Field with validation
    trading_pair: str = Field(
        default="BTC-USDT",
        json_schema_extra={
            "prompt": "Enter trading pair:",
            "prompt_on_new": True
        }
    )

    # Decimal for precise financial calculations
    order_amount: Decimal = Field(
        default=Decimal("1.0"),
        gt=0,  # Must be greater than 0
        json_schema_extra={
            "prompt": "Enter order amount:",
            "prompt_on_new": True
        }
    )

    # Enum for restricted choices
    side: TradeType = Field(
        default=TradeType.BUY,
        json_schema_extra={
            "prompt": "Enter side (BUY/SELL):",
            "prompt_on_new": True
        }
    )
```

#### Validation Example

```python
# Valid
config = MyStrategyConfig(
    trading_pair="ETH-USDT",
    order_amount="2.5",  # Automatically converted to Decimal
    side="BUY"           # Automatically converted to TradeType
)

# Invalid - Raises validation error
config = MyStrategyConfig(
    order_amount="-1.0"  # ERROR: Must be > 0
)
```

---

### Database Persistence

**SQLite Database with SQLAlchemy ORM**

#### Key Tables

**A. TradeFill (Trade History)**
```python
class TradeFill:
    config_file_path: str    # Which strategy made this trade
    strategy: str            # Strategy name
    market: str              # Exchange name
    symbol: str              # Trading pair
    base_asset: str          # Base asset
    quote_asset: str         # Quote asset
    timestamp: int           # When trade occurred
    order_id: str            # Order identifier
    trade_type: str          # BUY or SELL
    order_type: str          # MARKET or LIMIT
    price: Decimal           # Execution price
    amount: Decimal          # Amount traded
    leverage: int            # Leverage used (perpetuals)
    trade_fee: Dict          # Fees paid
    position: str            # OPEN or CLOSE (perpetuals)
```

**Usage:**
```python
# Query trades
with db.get_new_session() as session:
    trades = session.query(TradeFill).filter(
        TradeFill.timestamp >= start_time,
        TradeFill.market == "binance"
    ).all()
```

**B. MarketData (Historical Market Data)**
```python
class MarketData:
    timestamp: int          # When data recorded
    exchange: str           # Exchange name
    trading_pair: str       # Trading pair
    mid_price: Decimal      # Mid price
    best_bid: Decimal       # Best bid
    best_ask: Decimal       # Best ask
    order_book: Dict        # Full order book snapshot
```

**C. Executors (Executor State)**
```python
class Executors:
    id: str                        # Executor ID
    timestamp: int                 # Creation time
    type: str                      # Executor type
    status: str                    # Status
    config: Dict                   # Configuration
    net_pnl_pct: Decimal          # P&L percentage
    net_pnl_quote: Decimal        # P&L in quote asset
    cum_fees_quote: Decimal       # Total fees
    filled_amount_quote: Decimal  # Total filled
    is_active: bool               # Active flag
    is_trading: bool              # Trading flag
    custom_info: Dict             # Custom data
    controller_id: str            # Parent controller
```

---

## ADVANCED CONCEPTS

### Risk Management

#### A. Kill Switch

**Definition:** Automatic emergency stop when losses exceed threshold

```python
class KillSwitch:
    kill_switch_rate: Decimal  # Loss percentage to trigger
```

**How It Works:**
```
Configuration:
kill_switch_rate: -3.0  # Stop if 3% loss

Monitoring:
Initial portfolio: $10,000
Current portfolio: $9,650
Loss: -3.5%

Action: KILL SWITCH TRIGGERED
- Cancel all orders
- Close all positions
- Stop strategy
- Send notification
```

#### B. Budget Constraint

**Definition:** Ensure sufficient balance for orders

```python
def apply_budget_constraint(self, proposal):
    """Ensure you have enough balance for orders"""

    # Check base asset balance
    base_balance = connector.get_available_balance(base_asset)

    # Check quote asset balance
    quote_balance = connector.get_available_balance(quote_asset)

    # Adjust order sizes if insufficient
    if total_buy_cost > quote_balance:
        # Reduce buy order amounts

    if total_sell_amount > base_balance:
        # Reduce sell order amounts
```

#### C. Position Sizing

**Kelly Criterion:**
```python
position_size = (win_rate - (1 - win_rate) / win_loss_ratio) * capital

# Example:
# Win rate: 60%
# Win/Loss ratio: 1.5 (average win is 1.5x average loss)
# Capital: $10,000
position_size = (0.6 - 0.4/1.5) * 10000 = $3,333
```

#### D. Leverage Management (Perpetuals)

```python
config = PositionExecutorConfig(
    leverage=Decimal("10"),  # 10x leverage
    amount=Decimal("1.0")    # 1 BTC
)

# With 10x leverage:
Position size: 1 BTC * 50,000 USDT = 50,000 USDT
Required margin: 50,000 / 10 = 5,000 USDT

# If BTC moves 1%:
Without leverage: 1% gain = $500
With 10x leverage: 1% gain = $5,000 (10x profit)
But also: 1% loss = $5,000 (10x loss!)

# Liquidation
If loss exceeds margin, position liquidated
10% adverse move = -100% of margin = LIQUIDATED
```

---

### Performance Metrics

#### A. Return Percentage

```python
Return % = (Current Value - Initial Value) / Initial Value * 100

Example:
Initial: $10,000
Current: $10,500
Return: ($10,500 - $10,000) / $10,000 * 100 = 5%
```

#### B. Sharpe Ratio

```
Sharpe = (Portfolio Return - Risk-Free Rate) / Standard Deviation

Measures risk-adjusted return
Higher Sharpe = Better risk-adjusted performance
```

#### C. Maximum Drawdown

```
Max Drawdown = (Trough Value - Peak Value) / Peak Value

Example:
Peak: $12,000
Trough: $9,000
Max DD: ($9,000 - $12,000) / $12,000 = -25%

Shows worst peak-to-trough decline
```

#### D. Win Rate

```
Win Rate = Winning Trades / Total Trades * 100

Example:
60 wins out of 100 trades = 60% win rate
```

---

### Order Lifecycle

**Complete Journey of an Order:**

```
1. ORDER CREATION
   Controller decides to create order
   ↓
2. ORDER SUBMISSION
   Executor sends order to connector
   connector.buy() or connector.sell()
   ↓
3. ORDER SENT TO EXCHANGE
   Connector formats API request
   Authenticates and sends
   ↓
4. EXCHANGE RECEIVES ORDER
   Returns acknowledgment
   Provides exchange_order_id
   ↓
5. ORDER IN ORDERBOOK (for limit orders)
   Status: OPEN
   Waiting for match
   ↓
6. PARTIAL FILL (optional)
   Status: PARTIALLY_FILLED
   executed_amount < total_amount
   ↓
7. COMPLETE FILL
   Status: FILLED
   executed_amount == total_amount
   ↓
8. ORDER COMPLETED EVENT
   Executor receives BuyOrderCompletedEvent
   or SellOrderCompletedEvent
   ↓
9. POST-FILL ACTIONS
   Update positions
   Calculate P&L
   Record in database
   ↓
10. CLEANUP
    Remove from active orders
    Update metrics
```

#### Order States

```python
class OrderStatus(Enum):
    NEW = "NEW"                           # Just created
    OPEN = "OPEN"                         # In order book
    PARTIALLY_FILLED = "PARTIALLY_FILLED" # Some filled
    FILLED = "FILLED"                     # Completely filled
    CANCELED = "CANCELED"                 # Canceled by user
    EXPIRED = "EXPIRED"                   # Time expired
    FAILED = "FAILED"                     # Failed to place
    PENDING_CREATE = "PENDING_CREATE"     # Waiting for confirmation
    PENDING_CANCEL = "PENDING_CANCEL"     # Cancellation pending
```

---

### Event System

**Pub/Sub Pattern for Component Communication**

#### How It Works

**Publisher:**
```python
class Connector:
    def _emit_order_filled_event(self, order):
        event = OrderFilledEvent(
            timestamp=time.time(),
            order_id=order.client_order_id,
            trading_pair=order.trading_pair,
            trade_type=order.trade_type,
            price=fill_price,
            amount=fill_amount
        )
        self.trigger_event(event)
```

**Subscriber:**
```python
class Strategy:
    def __init__(self):
        # Subscribe to events
        self.add_listener(
            OrderFilledEvent,
            self._handle_order_filled
        )

    def _handle_order_filled(self, event: OrderFilledEvent):
        # React to order being filled
        self.logger().info(f"Order {event.order_id} filled!")
```

#### Event Types

```python
# Order Events
BuyOrderCreatedEvent
SellOrderCreatedEvent
OrderFilledEvent
OrderCompletedEvent
OrderCancelledEvent
OrderFailedEvent
OrderExpiredEvent

# Balance Events
BalanceUpdateEvent

# Market Events
OrderBookEvent
OrderBookTradeEvent

# Strategy Events
StrategyStartedEvent
StrategyStoppedEvent
```

---

## Quick Reference

### Common Trading Formulas

```
Spread = Ask Price - Bid Price
Mid Price = (Bid Price + Ask Price) / 2
VWAP = Σ(Price × Volume) / Σ(Volume)
Return % = (Final - Initial) / Initial × 100
Profit = Sell Price - Buy Price - Fees
Position Value = Amount × Price
Margin = Position Value / Leverage
```

### Important Constants

```python
# Order Book
BEST_BID = Highest buy order price
BEST_ASK = Lowest sell order price
MID_PRICE = Average of best bid and ask

# Time Intervals
1m = 1 minute
5m = 5 minutes
15m = 15 minutes
1h = 1 hour
4h = 4 hours
1d = 1 day

# Tick Size
Default: 1.0 second per tick
```

### Status Values

```python
# Executor Status
NOT_STARTED = "NOT_STARTED"
RUNNING = "RUNNING"
SHUTTING_DOWN = "SHUTTING_DOWN"
TERMINATED = "TERMINATED"

# Order Status
OPEN = "OPEN"
FILLED = "FILLED"
CANCELED = "CANCELED"
FAILED = "FAILED"

# Position Side
LONG = "LONG"   # Betting price goes up
SHORT = "SHORT" # Betting price goes down
```

---

## Resources

- **Official Website:** https://hummingbot.org
- **Documentation:** https://hummingbot.org/installation/
- **Discord Community:** https://discord.gg/hummingbot
- **GitHub Repository:** https://github.com/hummingbot/hummingbot
- **YouTube Channel:** https://www.youtube.com/@hummingbot
- **Newsletter:** https://hummingbot.substack.com

---

## Glossary Quick Lookup

| Term | Definition |
|------|------------|
| **AMM** | Automated Market Maker - DEX using liquidity pools |
| **API** | Application Programming Interface |
| **Ask** | Sell order in order book |
| **Bid** | Buy order in order book |
| **CEX** | Centralized Exchange |
| **CLMM** | Concentrated Liquidity Market Maker |
| **CLOB** | Central Limit Order Book |
| **DCA** | Dollar-Cost Averaging |
| **DEX** | Decentralized Exchange |
| **OHLCV** | Open, High, Low, Close, Volume (candle data) |
| **Perp** | Perpetual futures contract |
| **P&L** | Profit and Loss |
| **TWAP** | Time-Weighted Average Price |
| **VWAP** | Volume-Weighted Average Price |
| **XEMM** | Cross-Exchange Market Making |

---

**End of Documentation**

*This guide is maintained by the Hummingbot community. Contributions welcome at https://github.com/hummingbot/hummingbot*
