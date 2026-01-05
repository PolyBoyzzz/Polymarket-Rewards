# Polymarket Automated Market Making Bot Documentation

**Language / 语言**: English | [中文](README.md)

## For complete code, please contact TG: [@polyboy123](https://t.me/polyboy123)
<img width="3840" height="2160" alt="80afc575bba1c6193d9457b387bc6ba9" src="https://github.com/user-attachments/assets/2b404e92-5c8f-4208-a577-bf3bc537be34" />
<img width="1342" height="859" alt="9d8aaf842cab273b23d24d25abd30cbe" src="https://github.com/user-attachments/assets/7ed47581-fdc1-4ddd-8940-edc14605d0a5" />
## I. System Overview

### 1.1 Core Features

This system is an automated market making strategy program based on liquidity rewards, intelligently placing orders to obtain liquidity rewards from the Polymarket platform. The system can:

- Automatically scan and filter markets with liquidity rewards
- Intelligently place orders at reward range boundaries
- Automatically manage orders (replenish orders, adjust prices, hedge selling)
- Real-time risk control and exposure management
- Use Redis cache to optimize data retrieval performance

### 1.2 Technical Architecture

- **Language**: Python 3.12
- **Main Dependencies**: py-clob-client (Polymarket official SDK), Redis, requests
- **Data Storage**: Redis (orderbook cache, market data cache)
- **API Communication**: HTTP REST API (Polymarket API, orderbook service)

---

## II. Core Module Description

### 2.1 Main Program Module (main.py)

**Responsibility**: Program entry point, coordinates all modules, implements main loop logic

**Main Flow**:

1. **Initialization Phase**:
   - Initialize all components (API client, market manager, order manager, risk manager, market making strategy)
   - Cancel all existing buy orders (ensure clean startup)
   - Check orderbook data service status

2. **Initial Market Scanning and Filtering**:
   - Scan all markets with liquidity rewards
   - Use multi-dimensional filtering (spread, trading volume, reward share, time limits, blacklist, etc.)
   - Calculate reward ratios and sort
   - Place orders for selected markets

3. **Main Loop** (three parallel tasks):
   - **Order Status Check** (every 0.5 seconds):
     - Real-time trade detection (`check_trades_and_hedge`): Detect buy order fills through `get_trades()` and immediately hedge sell
     - Position check (`check_positions_and_hedge`): Ensure all positions have hedge sell orders
     - Order check (`check_orders`): Detect order fill status, handle replenishment logic

   - **Price Adjustment** (every 0.5 seconds):
     - Get real-time orderbook data
     - Calculate new reward range boundary prices
     - If price deviation exceeds threshold, cancel old orders and re-place orders

   - **Market Re-scanning** (every 1200 seconds, 20 minutes):
     - Cancel all unfilled orders for currently active markets
     - Re-scan and filter markets
     - Place orders for new opportunity markets

4. **Graceful Shutdown**:
   - Cancel all buy orders
   - Display final statistics

---

### 2.2 Market Manager (market_manager.py)

**Responsibility**: Market data management, market filtering, reward ratio calculation

**Core Functions**:

1. **Market Scanning** (`scan_rewards_markets`):
   - Get all markets with liquidity rewards from Redis cache or API
   - Support automatic pagination

2. **Market Filtering** (`filter_markets`):
   - **Multi-dimensional Filtering**:
     - Spread range filtering (real-time calculation)
     - 24-hour trading volume filtering
     - Minimum reward share filtering
     - Daily reward rate filtering
     - Market end time filtering (avoid high volatility markets near end)
     - Question blacklist filtering (keyword matching)

   - **Reward Ratio Calculation** (`calculate_reward_ratio`):
     - Formula: `Reward Ratio = (Our Order Share × 2) / (Competitor Total Share) × Daily Reward Amount`
     - Our order share = `rewards_min_size × order_size_multiplier × 2` (bilateral orders)
     - Competitor total share = sum of shares at bid1, bid2, ask1, ask2

   - **Order Placement Safety Check** (`_check_market_can_place_orders`):
     - Determine if bilateral orders are needed (any token's mid price ≤ 0.10)
     - Risk management check for each token (price cliff, protection share, volatility, etc.)
     - Markets requiring bilateral orders: all tokens must pass checks
     - Normal markets: at least one token must pass checks

3. **Market Data Caching**:
   - Cache market data to avoid duplicate queries

---

### 2.3 Order Manager (order_manager.py)

**Responsibility**: Order lifecycle management, automatic replenishment, hedge selling, price adjustment

**Core Functions**:

1. **Order Placement** (`place_market_orders`):
   - Place buy and sell orders for each token in the market
   - Place orders at reward range boundaries (bid2 price or reward lower boundary)
   - Risk management checks (price cliff, protection share, volatility)
   - Exposure limit checks

2. **Order Check** (`check_orders`):
   - **Dual Detection Mechanism**:
     - **Primary Method**: Use position information detection (buy order fills increase positions)
     - **Fallback Method**: Use trade history query

   - **Partial Fill Tracking**:
     - Track remaining portions of partially filled orders
     - Immediately place hedge sell orders for newly filled portions

   - **Processing Logic**:
     - Fully filled: Remove from active orders, trigger replenishment and hedge selling
     - Partially filled: Add to tracking list, immediately hedge sell filled portions

3. **Real-time Trade Detection** (`check_trades_and_hedge`):
   - **Primary Hedging Mechanism**:
     - Call `get_trades(after=last_trade_check_time)` to get new trades
     - Identify own buy order fills (match by address/API key)
     - Handle segmented buys (use `order_filled_tracking` to track cumulative fills)
     - Immediately call `place_hedge_sell()` for each trade's filled share
     - If hedging fails, add to async retry queue

4. **Position Check** (`check_positions_and_hedge`):
   - **Fallback Hedging Mechanism**:
     - Query all positions
     - Check if each position already has a hedge sell order
     - If position share doesn't match placed sell order share, re-place full position sell order
     - Cooldown mechanism: no retry within 30 seconds after hedging failure

5. **Hedge Selling** (`place_hedge_sell`):
   - **Price Calculation Strategy** (by priority):
     1. **Smart Price Layer Selection**: Select the lowest price layer from orderbook that can cover all sell shares (within `max_bid_gap` threshold)
     2. **Fallback to Bid1**: If bid1 price difference from buy price ≤ `max_bid_gap`
     3. **Original Price Logic**: `max(buy_price, current_market_price) + minimum_profit`

   - **Position Verification**:
     - Query actual positions
     - Calculate available position = actual position - placed sell order share
     - Ensure sufficient position before placing sell order

6. **Price Adjustment** (`adjust_orders_to_reward_boundaries`):
   - Get real-time orderbook data
   - Calculate new reward range boundary prices
   - If price deviation exceeds threshold (`price_deviation_threshold_bps`), cancel old orders and re-place orders

7. **Async Retry Mechanism**:
   - Failed hedge sell tasks added to retry queue
   - Background thread periodically retries (fixed delay 1 second)
   - Maximum retry count: 10 times

---

### 2.4 Market Making Strategy Module (market_making_strategy.py)

**Responsibility**: Price calculation, risk management checks, order size calculation

**Core Functions**:

1. **Price Calculation**:
   - **Mid Price Calculation** (`calculate_mid_price`): `(best_bid + best_ask) / 2`
   - **Reward Range Calculation** (`calculate_reward_range`):
     - Buy price = mid price - `(rewards_max_spread - 1) / 100`
     - Sell price = mid price + `(rewards_max_spread - 1) / 100`
   - **Price Normalization** (`normalize_price`):
     - Round down according to market's minimum price tick size (`orderPriceMinTickSize`)
     - Limit to valid range [0.01, 1.0]

2. **Actual Order Price Calculation** (`calculate_actual_buy_price`):
   - If orderbook only has bid1, return None (skip)
   - If bid2 ≥ reward lower boundary, return bid2
   - If bid2 < reward lower boundary, return reward lower boundary

3. **Risk Management Check** (`can_place_buy_order_safely`):
   - **Volatility Check**:
     - Get volatility data from Redis
     - If volatility exceeds threshold, reject order placement

   - **Position Check**:
     - If using reward lower boundary, check if order becomes bid2 after placement
     - If using bid2, skip position check

   - **Price Cliff Check**:
     - Check if bid1, bid2 exist
     - Check if price difference exceeds threshold (`price_cliff_threshold`)

   - **Protection Share Check**:
     - Bid1 + bid2 total shares ≥ `order_size × min_protection_size_multiplier`

   - **Hedge Sell Share Check**:
     - Bid3, bid4, etc. (within `max_bid_gap` threshold) cumulative shares ≥ `order_size × min_protection_size_multiplier_hedge`

4. **Order Size Calculation** (`calculate_order_size`):
   - Actual order size = `rewards_min_size × order_size_multiplier`
   - At least market minimum size

5. **Hedge Sell Price Calculation** (`calculate_hedge_sell_price`):
   - Smart price layer selection (priority)
   - Fallback to bid1
   - Original price logic (last resort)

---

### 2.5 Risk Manager (risk_manager.py)

**Responsibility**: Exposure tracking, maximum exposure limits

**Core Functions**:

1. **Exposure Calculation** (`calculate_exposure`):
   - Buy order exposure = `order_price × order_size`
   - Sell orders don't calculate exposure (selling held tokens, no new capital required)

2. **Exposure Limit Check** (`can_place_order`):
   - Sell orders directly allowed
   - Buy orders need check: `current_exposure + new_order_exposure ≤ max_exposure_per_market_usdc`

3. **Exposure Management**:
   - **When Placing Order** (`add_exposure`): Add pending order exposure
   - **When Canceling Order** (`remove_exposure`): Remove pending order exposure
   - **When Order Fills** (`add_filled_order_exposure`): Add filled order exposure (capital occupied)
   - **When Hedge Selling** (`remove_filled_order_exposure`): Remove filled order exposure (release capital after sell order fills)

---

### 2.6 API Client (api_client.py)

**Responsibility**: Communication with Polymarket API

**Core Functions**:

1. **Market Data Retrieval**:
   - `get_rewards_markets()`: Get markets with liquidity rewards (supports pagination)
   - `get_all_rewards_markets()`: Automatically handle pagination, get all markets (prefer Redis cache)

2. **Orderbook Data Retrieval**:
   - `get_orderbook(token_id)`: Get single token's orderbook
   - `get_markets_orderbooks(markets)`: Batch get multiple markets' orderbooks (prefer Redis cache)

3. **Order Operations** (via py-clob-client):
   - Place orders, cancel orders, query orders, query positions, query trade history

---

### 2.7 Orderbook Data Service (orderbook_data_service.py / redis_orderbook_client.py)

**Responsibility**: Orderbook data caching, market data caching, volatility calculation

**Core Functions**:

1. **Data Caching**:
   - Market list cache (TTL: 5 minutes)
   - Orderbook data cache (TTL: 300 seconds)
   - Complete market details cache (TTL: 7 days)

2. **Volatility Calculation**:
   - Sliding window calculation (window size: 30 data points, ~15 minutes)
   - Use coefficient of variation (standard deviation/mean) as volatility indicator
   - If volatility exceeds threshold, mark as dangerous market

3. **Data Updates**:
   - Periodically scan market list (every 5 minutes)
   - Periodically update orderbook data (every 3 seconds)
   - Batch get market details (20 per batch)

---

## III. Key Business Processes

### 3.1 Market Filtering Process

```
1. Scan all markets with liquidity rewards
   ↓
2. Blacklist filtering (problem keywords)
   ↓
3. Trading volume filtering (24-hour volume range)
   ↓
4. Reward share filtering (rewards_min_size range)
   ↓
5. Batch get orderbook data (from Redis cache)
   ↓
6. Real-time spread filtering (calculated from orderbook)
   ↓
7. Time filtering (market end time limit)
   ↓
8. Daily reward rate filtering
   ↓
9. Calculate reward ratio (consider competition)
   ↓
10. Order placement safety check (price cliff, protection share, volatility)
   ↓
11. Sort by reward ratio descending
   ↓
12. Select top N markets
```

### 3.2 Order Placement Process

```
1. Get market data (tokens, rewards_max_spread, etc.)
   ↓
2. Determine if bilateral orders needed (any token mid price ≤ 0.10)
   ↓
3. For each token:
   a. Get real-time orderbook data
   b. Calculate mid price
   c. Calculate reward range boundaries (buy_price, sell_price)
   d. Calculate actual order price (bid2 or reward lower boundary)
   e. Risk management check (price cliff, protection share, volatility)
   f. Exposure limit check
   g. Place buy and sell orders
```

### 3.3 Order Check and Hedging Process

```
Order check in main loop (every 0.5 seconds):
   ↓
1. Real-time trade detection (check_trades_and_hedge)
   - Get new trade records (get_trades)
   - Identify own buy order fills
   - Immediately hedge sell (place_hedge_sell)
   ↓
2. Position check (check_positions_and_hedge)
   - Query all positions
   - Check if hedge sell orders exist
   - If position doesn't match sell order, re-place full position sell order
   ↓
3. Order check (check_orders)
   - Detect order fill status (position info + trade history)
   - Handle fully filled: replenish + hedge sell
   - Handle partially filled: track + hedge sell
```

### 3.4 Price Adjustment Process

```
1. Get current active market list
   ↓
2. For each market:
   a. Get real-time orderbook data
   b. Calculate new reward range boundary prices
   c. Check if price deviation exceeds threshold
   d. If exceeds threshold:
      - Cancel old orders
      - Re-place orders (using new prices)
```

### 3.5 Market Re-scanning Process

```
1. Cancel all unfilled orders for currently active markets
   ↓
2. Re-check positions and re-place sell orders (using new TP/SL strategy)
   ↓
3. Re-scan all markets with liquidity rewards
   ↓
4. Re-filter markets (using latest data)
   ↓
5. Only place orders for new markets (exclude markets still in active list)
```

---

## IV. Key Configuration Parameters

### 4.1 Market Making Strategy Configuration

| Parameter                      | Description                      | Default Value |
| ------------------------------ | -------------------------------- | ------------- |
| `order_size_multiplier`        | Order size multiplier            | 1.0           |
| `max_exposure_per_market_usdc` | Max exposure per market (USDC)   | 25            |
| `max_markets`                  | Maximum number of selected markets | 10         |
| `min_reward_ratio`             | Minimum reward ratio threshold   | 0             |
| `min_profit_margin_bps`        | Minimum profit for hedge sell (bps) | 5          |

### 4.2 Risk Management Configuration

| Parameter                           | Description                      | Default Value |
| ----------------------------------- | -------------------------------- | ------------- |
| `price_cliff_threshold`             | Price cliff threshold (absolute difference) | 0.05    |
| `min_protection_size_multiplier`    | Minimum protection size multiplier | 1.0        |
| `min_protection_size_multiplier_hedge` | Hedge protection size multiplier | 5.0      |

### 4.3 Hedge Selling Configuration

| Parameter                      | Description              | Default Value |
| ------------------------------ | ------------------------ | ------------- |
| `hedge_sell.max_bid_gap`       | Maximum bid gap threshold | 0.05          |
| `hedge_sell.stop_loss_pct`     | Stop loss percentage     | 0.10          |
| `hedge_sell.take_profit_pct`   | Take profit percentage   | 0.10          |
| `hedge_sell.dynamic_follow_enabled` | Enable dynamic following | true      |

### 4.4 Market Filtering Configuration

| Parameter                | Description                      | Default Value           |
| ------------------------ | -------------------------------- | ----------------------- |
| `spread_range`           | Spread filtering range           | `{min: null, max: 0.05}` |
| `volume_24hr_range`      | 24-hour volume filtering range   | `{min: null, max: null}` |
| `rewards_min_size_range` | Minimum reward share filtering range | `{min: null, max: 20}` |
| `rate_per_day_range`     | Daily reward rate filtering range | `{min: 5, max: null}` |
| `min_days_until_end`     | Minimum remaining days            | 7                       |
| `question_blacklist`      | Question blacklist (keyword list) | `["Trump"]`             |

### 4.5 Main Loop Configuration

| Parameter                           | Description                      | Default Value |
| ----------------------------------- | -------------------------------- | ------------- |
| `update_interval_seconds`           | Market scan update interval (seconds) | 1200       |
| `order_check_interval_seconds`      | Order status check interval (seconds) | 0.5        |
| `orderbook_update_interval_seconds` | Orderbook monitoring update interval (seconds) | 0.5 |
| `price_deviation_threshold_bps`     | Price deviation threshold (bps)  | 0.0001        |

### 4.6 Orderbook Data Service Configuration

| Parameter                                      | Description                      | Default Value |
| ---------------------------------------------- | -------------------------------- | ------------- |
| `orderbook_service.enabled`                    | Enable orderbook data service    | true          |
| `orderbook_service.market_scan_interval`       | Market scan interval (seconds)   | 300           |
| `orderbook_service.orderbook_update_interval`  | Orderbook update interval (seconds) | 3          |
| `orderbook_service.redis.host`                 | Redis host                       | localhost     |
| `orderbook_service.redis.port`                 | Redis port                       | 6379          |
| `orderbook_service.volatility_check.enabled`   | Enable volatility check          | true          |
| `orderbook_service.volatility_check.volatility_threshold` | Volatility threshold | 0.03      |

---

## V. Data Structures

### 5.1 Market Data Structure

```python
{
    "market_id": "570361",
    "condition_id": "0xcb11...",
    "question": "Market question",
    "tokens": [
        {
            "token_id": "0x...",
            "outcome": "Yes"
        }
    ],
    "rewards_config": [
        {
            "rate_per_day": 10.5  # Daily reward amount (USDC)
        }
    ],
    "rewards_max_spread": 3.5,  # Maximum reward spread (cents)
    "rewards_min_size": 200,  # Minimum reward share
    "volume_24hr": 5000.0,  # 24-hour trading volume (USDC)
    "spread": 0.03,  # Spread
    "reward_ratio": 0.001234  # Reward ratio (calculated)
}
```

### 5.2 Order Data Structure

```python
{
    "order_id": "0x...",
    "token_id": "0x...",
    "market_id": "570361",
    "side": "BUY",  # or "SELL"
    "price": 0.65,
    "size": 200,
    "status": "OPEN",  # or "FILLED", "CANCELLED"
    "filled_size": 100,  # Filled share
    "created_at": "2025-01-01T00:00:00Z"
}
```

### 5.3 Active Order Tracking Structure

```python
active_orders = {
    "market_id": {
        "token_id": {
            "BUY": {
                "order_id": "0x...",
                "price": 0.65,
                "size": 200,
                ...
            },
            "SELL": {
                "order_id": "0x...",
                "price": 0.70,
                "size": 200,
                ...
            }
        }
    }
}
```

---

## VI. Risk Control Mechanisms

### 6.1 Exposure Limits

- Maximum exposure limit per market (`max_exposure_per_market_usdc`)
- Buy orders calculate exposure, sell orders don't calculate exposure
- Filled orders occupy exposure until hedge sell fills

### 6.2 Price Cliff Check

- Check if bid1, bid2 exist
- Check if price difference exceeds threshold (`price_cliff_threshold`)
- If price cliff exists, reject order placement

### 6.3 Protection Share Check

- Bid1 + bid2 total shares must be ≥ `order_size × min_protection_size_multiplier`
- Ensure sufficient protection, not easily consumed

### 6.4 Hedge Sell Share Check

- Bid3, bid4, etc. (within `max_bid_gap` threshold) cumulative shares must be ≥ `order_size × min_protection_size_multiplier_hedge`
- Ensure sufficient liquidity for hedge selling

### 6.5 Volatility Check

- Use sliding window to calculate price volatility (coefficient of variation)
- If volatility exceeds threshold, reject order placement
- Avoid risks in high volatility markets

### 6.6 Market Filtering

- Spread range filtering
- Trading volume filtering
- Reward share filtering
- Time filtering (avoid high volatility markets near end)
- Blacklist filtering (avoid markets with specific themes)

---

## VII. Performance Optimization

### 7.1 Redis Cache

- Market list cache (reduce API calls)
- Orderbook data cache (improve data retrieval speed)
- Complete market details cache (reduce duplicate queries)

### 7.2 Batch Operations

- Batch get orderbook data
- Batch get market details
- Batch cancel orders

### 7.3 Async Processing

- Failed hedge sell tasks added to async retry queue
- Background thread periodically retries, doesn't block main loop

### 7.4 Incremental Check

- Only check new trades since last check (`get_trades(after=last_trade_check_time)`)
- Avoid duplicate processing of already processed trades

### 7.5 Lock Optimization

- Copy data structures outside locks, avoid holding locks for long
- Use fine-grained locks, reduce lock contention

---

## VIII. Error Handling and Retry Mechanisms

### 8.1 API Request Retry

- Exponential backoff retry (max 3 times)
- Timeout setting (30 seconds)

### 8.2 Order Operation Retry

- Order retry count: 3 times
- Order retry delay: 1.0 seconds

### 8.3 Hedge Sell Retry

- Async retry queue
- Maximum retry count: 10 times
- Fixed retry delay: 1.0 seconds
- Cooldown mechanism: no retry within 30 seconds after failure

### 8.4 Exception Handling

- All critical operations have exception handling
- Exceptions don't interrupt main loop
- Detailed error log recording

---

## IX. Logging System

### 9.1 Log Levels

- **Console log level**: INFO (default)
- **File log level**: WARNING (default)

### 9.2 Log Files

- Classified by date and module:
  - `logs/YYYY-MM-DD/main.log` - Main program logs
  - `logs/YYYY-MM-DD/order_manager.log` - Order manager logs
  - `logs/YYYY-MM-DD/market_manager.log` - Market manager logs
  - `logs/YYYY-MM-DD/api_client.log` - API client logs
  - `logs/YYYY-MM-DD/risk_manager.log` - Risk manager logs
  - `logs/YYYY-MM-DD/market_making_strategy.log` - Market making strategy logs

### 9.3 Performance Logs

- Record execution time of key operations
- Example: `[Performance] check_trades_and_hedge() took: 0.123 seconds`

---

## X. Deployment and Running

### 10.1 Environment Requirements

- Python 3.12+
- Redis server (for data caching)
- Network connection (access Polymarket API)

### 10.2 Environment Variables

```bash
POLYMARKET_PRIVATE_KEY=your_private_key_here
POLYMARKET_FUNDER=your_funder_address_here
POLYMARKET_API_KEY=your_api_key_here  # Optional, for order matching
```

### 10.3 Configuration File

- `config.yaml`: Main configuration file
- Contains all configurable parameters

### 10.4 Running Methods

```bash
# Start orderbook data service (for data caching and volatility calculation)
python start_orderbook_service.py
```


```bash
# Run in foreground
python main.py

# Run in background (daemon mode, Linux/Mac only)
python main.py --daemon

# Stop mode (cancel all buy orders then exit)
python main.py --stop
```

---

## XI. Key Features Summary

### 11.1 Intelligent Market Filtering

- Multi-dimensional filtering (spread, trading volume, reward share, time, blacklist)
- Reward ratio calculation (consider competition)
- Order placement safety check (price cliff, protection share, volatility)

### 11.2 Intelligent Order Placement Strategy

- Place orders at reward range boundaries (maximize reward acquisition probability)
- Bid2 or reward lower boundary (ensure becoming bid2)
- Risk management checks (ensure safety)

### 11.3 Automatic Order Management

- Automatically replenish orders after fills
- Immediately hedge sell after buy order fills (dual mechanism: real-time trade detection + position check)
- Real-time order price adjustment (maintain at reward range boundaries)

### 11.4 Risk Control

- Maximum exposure limit per market
- Price cliff check
- Protection share check
- Volatility check
- Market filtering

### 11.5 Performance Optimization

- Redis cache (reduce API calls)
- Batch operations (improve efficiency)
- Async processing (don't block main loop)
- Incremental check (avoid duplicate processing)

---

## XII. Important Notes

### 12.1 Security

- Private keys and Funder addresses should be set via environment variables, don't commit to code repository
- Ensure configuration file permissions are set correctly

### 12.2 Risk

- Market making strategy has market risks, please set parameters reasonably according to capital situation
- Recommend testing in test environment or with small capital first
- Regularly check logs, discover and solve problems in time

### 12.3 Network Stability

- Ensure stable network connection, avoid order operation failures
- If network is unstable, recommend increasing retry count and delay

### 12.4 Monitoring

- Regularly check log files
- Monitor order status and position situation
- Pay attention to exposure and risk indicators

---

## XIII. Technical Support

For issues, please check:

- `README.md`: Detailed usage instructions
- `订单检查和对冲止损逻辑总结.md`: Detailed explanation of order check and hedging logic
- Log files: View detailed running logs

---

**Document Version**: 1.0  
**Last Updated**: 2026-01-03


