# Equities Copy Trading System: Learnings from Solana Copy Trader

## Executive Summary

This document analyzes a production-grade Solana copy trading bot to extract architectural patterns, API strategies, and implementation approaches applicable to building an **equities copy trading system** that:
1. Connects to **Leader API** (read leader trades on US equities platform)
2. Connects to **Follower API** (execute trades for followers)

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [API Connection Patterns](#2-api-connection-patterns)
3. [Leader Trade Monitoring](#3-leader-trade-monitoring)
4. [Follower Trade Execution](#4-follower-trade-execution)
5. [Position & Risk Management](#5-position--risk-management)
6. [Configuration Management](#6-configuration-management)
7. [Data Models & State Tracking](#7-data-models--state-tracking)
8. [Error Handling & Retry Logic](#8-error-handling--retry-logic)
9. [Performance Optimizations](#9-performance-optimizations)
10. [Equities-Specific Adaptations](#10-equities-specific-adaptations)

---

## 1. System Architecture Overview

### Solana Bot Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Main Application                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐      ┌──────────────────┐             │
│  │   Config Loader │      │ Background Tasks │             │
│  │   (.env vars)   │      │ - Blockhash      │             │
│  └─────────────────┘      │ - Cache Cleanup  │             │
│                            │ - Health Checks  │             │
│                            └──────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│              ┌────────────────────────────┐                 │
│              │  Dual Monitoring System    │                 │
│              ├────────────────────────────┤                 │
│              │  1. Target Wallet Monitor  │◄───gRPC Stream  │
│              │  2. DEX Activity Monitor   │◄───gRPC Stream  │
│              └────────────────────────────┘                 │
├─────────────────────────────────────────────────────────────┤
│         ┌──────────────┐      ┌────────────────┐           │
│         │  Transaction │      │    Protocol    │           │
│         │    Parser    │      │    Adapters    │           │
│         └──────────────┘      └────────────────┘           │
├─────────────────────────────────────────────────────────────┤
│    ┌────────────────┐  ┌────────────────┐  ┌─────────┐    │
│    │  Swap Engine   │  │  Risk Manager  │  │ Selling │    │
│    │  (Execution)   │  │                │  │ Engine  │    │
│    └────────────────┘  └────────────────┘  └─────────┘    │
├─────────────────────────────────────────────────────────────┤
│              Global State (Lock-Free DashMaps)              │
│  - Position Tracking  - Price Monitoring  - Blacklists     │
└─────────────────────────────────────────────────────────────┘
```

### Proposed Equities Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Equities Copy Trader                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐      ┌──────────────────┐             │
│  │   Config Loader │      │ Background Tasks │             │
│  │  (env/secrets)  │      │ - Market Data    │             │
│  └─────────────────┘      │ - Position Sync  │             │
│                            │ - Health Checks  │             │
│                            └──────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│              ┌────────────────────────────┐                 │
│              │   Dual API Connections     │                 │
│              ├────────────────────────────┤                 │
│              │  1. Leader API (READ)      │◄───REST/WS     │
│              │     - Trade notifications  │                 │
│              │     - Position updates     │                 │
│              │  2. Follower API (WRITE)   │◄───REST/WS     │
│              │     - Order submission     │                 │
│              │     - Position management  │                 │
│              └────────────────────────────┘                 │
├─────────────────────────────────────────────────────────────┤
│         ┌──────────────┐      ┌────────────────┐           │
│         │    Trade     │      │    Broker      │           │
│         │  Normalizer  │      │    Adapters    │           │
│         └──────────────┘      └────────────────┘           │
├─────────────────────────────────────────────────────────────┤
│    ┌────────────────┐  ┌────────────────┐  ┌─────────┐    │
│    │  Order Engine  │  │  Risk Manager  │  │ Exit    │    │
│    │  (Execution)   │  │                │  │ Strategy│    │
│    └────────────────┘  └────────────────┘  └─────────┘    │
├─────────────────────────────────────────────────────────────┤
│              Global State (Thread-Safe Storage)             │
│  - Position Tracking  - Price Cache  - Execution History   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. API Connection Patterns

### 2.1 Solana Implementation

**Connection Type**: Real-time gRPC Stream (Yellowstone)
- **Low latency**: Sub-100ms notification of new transactions
- **Filtered subscriptions**: Only receive relevant wallet/program transactions
- **Bidirectional**: Supports heartbeat/ping to maintain connection
- **Automatic reconnection**: Handles network failures gracefully

**Code Pattern** (from `src/processor/sniper_bot.rs`):
```rust
// Create gRPC client with TLS
let mut client = GeyserGrpcClient::build_from_shared(yellowstone_grpc_http)
    .tls_config(ClientTlsConfig::default())
    .connect().await;

// Subscribe with filters
let subscribe_request = SubscribeRequest {
    transactions: Some(SubscribeRequestFilterTransactions {
        vote: Some(false),
        account_include: vec![target_wallet.clone()],
        ..Default::default()
    }),
    commitment: Some(CommitmentLevel::Processed as i32),
    ..Default::default()
};

// Process incoming stream with heartbeat
loop {
    tokio::select! {
        Some(update) = stream.next() => {
            // Process transaction update
            match update.update_oneof {
                Some(UpdateOneof::Transaction(tx_update)) => {
                    // Parse and execute
                }
            }
        }
        _ = time::sleep(PING_INTERVAL) => {
            send_heartbeat_ping(&subscribe_tx).await;
        }
    }
}
```

### 2.2 Equities Adaptation

#### Leader API (Reading Trades)

**Recommended Approaches**:

1. **WebSocket Subscriptions** (Preferred)
   - Real-time trade notifications
   - Position updates
   - Order status changes

2. **REST Polling with Smart Intervals**
   - Every 1-5 seconds for active hours
   - Long polling where supported
   - Rate limit aware

3. **Webhook Endpoints** (If Supported)
   - Receive push notifications
   - Zero polling overhead
   - Requires public endpoint

**Equities API Examples**:

| Platform | Read API Type | Best Practice |
|----------|---------------|---------------|
| Alpaca | WebSocket + REST | Use WebSocket for real-time, REST for history |
| Interactive Brokers | TWS API (socket) | Use IB Gateway for stability |
| TD Ameritrade | WebSocket (streaming) | Authenticate with OAuth2 + refresh tokens |
| E*TRADE | REST + OAuth | Poll every 2-5 seconds during market hours |
| TradeStation | WebSocket + REST | WebSocket for executions, REST for positions |

**Implementation Pattern**:
```python
# Example: WebSocket-based leader monitoring
class LeaderMonitor:
    def __init__(self, api_key, leader_account_id):
        self.ws_client = AlpacaStreamClient(api_key)
        self.leader_id = leader_account_id

    async def start_monitoring(self):
        # Subscribe to leader's trade stream
        await self.ws_client.subscribe_trades(
            account_id=self.leader_id,
            callback=self.on_leader_trade
        )

        # Maintain connection with heartbeat
        asyncio.create_task(self.heartbeat_loop())

    async def on_leader_trade(self, trade_event):
        """
        Process incoming trade from leader
        Extract: symbol, side, quantity, price, timestamp
        """
        normalized_trade = {
            'symbol': trade_event['symbol'],
            'side': trade_event['side'],  # BUY/SELL
            'quantity': trade_event['qty'],
            'price': trade_event['price'],
            'timestamp': trade_event['timestamp'],
            'order_type': trade_event['order_type']
        }

        # Pass to execution engine
        await self.execute_follower_trade(normalized_trade)
```

#### Follower API (Executing Trades)

**Key Requirements**:
1. **Order placement** with specific parameters
2. **Position querying** for verification
3. **Account balance checks**
4. **Order status monitoring**

**Implementation Pattern**:
```python
class FollowerExecutor:
    def __init__(self, api_credentials):
        self.client = BrokerClient(api_credentials)
        self.position_cache = {}  # Thread-safe cache

    async def execute_trade(self, trade_params):
        """
        Execute follower trade based on leader's action
        """
        # 1. Pre-execution checks
        if not await self.validate_trade(trade_params):
            return None

        # 2. Calculate position size (proportional to account)
        follower_qty = self.calculate_position_size(
            leader_qty=trade_params['quantity'],
            leader_account_size=self.leader_portfolio_value,
            follower_account_size=self.follower_portfolio_value
        )

        # 3. Submit order
        order = await self.client.submit_order(
            symbol=trade_params['symbol'],
            qty=follower_qty,
            side=trade_params['side'],
            type='market',  # or 'limit' with calculated price
            time_in_force='day'
        )

        # 4. Track order
        self.position_cache[trade_params['symbol']] = {
            'order_id': order.id,
            'entry_price': None,  # Will update when filled
            'entry_time': time.time(),
            'quantity': follower_qty
        }

        return order
```

### 2.3 Connection Resilience

**Solana Pattern**:
- Automatic reconnection on stream failure
- Fallback RPC endpoints
- Health check background service

**Equities Adaptation**:
```python
class ResilientAPIConnection:
    def __init__(self, primary_endpoint, fallback_endpoints):
        self.primary = primary_endpoint
        self.fallbacks = fallback_endpoints
        self.current_connection = None
        self.reconnect_attempts = 0
        self.max_reconnects = 5

    async def connect_with_fallback(self):
        """
        Try primary, then fallbacks with exponential backoff
        """
        endpoints = [self.primary] + self.fallbacks

        for endpoint in endpoints:
            try:
                connection = await self.establish_connection(endpoint)
                self.current_connection = connection
                self.reconnect_attempts = 0
                return connection
            except ConnectionError:
                await asyncio.sleep(2 ** self.reconnect_attempts)
                self.reconnect_attempts += 1
                continue

        raise MaxReconnectAttemptsExceeded()

    async def monitor_health(self):
        """
        Background task to monitor connection health
        """
        while True:
            if not await self.is_connection_healthy():
                await self.connect_with_fallback()
            await asyncio.sleep(30)  # Check every 30s
```

---

## 3. Leader Trade Monitoring

### 3.1 Solana Implementation

The bot uses **dual monitoring streams**:

1. **Target Wallet Monitoring** (`start_target_wallet_monitoring()`):
   - Subscribes to all transactions where target wallets are signers
   - Parses transaction logs to extract trade details
   - Identifies protocol (PumpFun, Raydium, etc.)
   - Executes copycat trade immediately

2. **DEX Monitoring** (`start_dex_monitoring()`):
   - Monitors all DEX transactions (not just target wallets)
   - Builds 20-slot price/volume timeseries
   - Detects market opportunities (post-drop bottoms)
   - Enables "sniper" entries

**Key Code** (simplified from `sniper_bot.rs`):
```rust
// Target wallet monitoring
pub async fn start_target_wallet_monitoring(config: Arc<Config>) {
    loop {
        // Get transaction update from stream
        if let Some(UpdateOneof::Transaction(tx_update)) = update.update_oneof {
            // Extract signer
            let signer = extract_signer_from_transaction(&tx_update);

            // Check if signer is in target list
            if target_addresses.contains(&signer) {
                // Parse transaction to extract trade info
                let trade_info = parse_transaction_data(&tx_update);

                // Filter excluded addresses
                if !excluded_addresses.contains(&trade_info.mint) {
                    // Check counter limit
                    if current_positions < counter_limit {
                        // Execute swap
                        execute_swap(trade_info).await;
                    }
                }
            }
        }
    }
}
```

### 3.2 Equities Adaptation

**Dual Monitoring Approach**:

1. **Leader Account Monitoring**:
   - Direct subscription to leader's order executions
   - Parse order details (symbol, side, quantity, price)
   - Apply filters (min size, symbol whitelist/blacklist)
   - Route to execution engine

2. **Market Data Monitoring** (Optional):
   - Real-time quotes for positions
   - Detect significant price movements
   - Update position P&L
   - Trigger exit strategies

**Implementation Pattern**:
```python
class EquitiesLeaderMonitor:
    def __init__(self, leader_config):
        self.leader_accounts = leader_config['account_ids']
        self.excluded_symbols = set(leader_config.get('excluded_symbols', []))
        self.min_trade_value = leader_config.get('min_trade_value', 100)
        self.position_limit = leader_config.get('max_positions', 10)

    async def monitor_leader_trades(self):
        """
        Main monitoring loop for leader trades
        """
        async for trade_event in self.leader_api.stream_trades():
            # Extract trade information
            trade = self.normalize_trade(trade_event)

            # Apply filters
            if self.should_copy_trade(trade):
                # Check position limits
                current_positions = len(self.position_tracker.active_positions)

                if current_positions < self.position_limit:
                    # Queue for execution
                    await self.execution_queue.put(trade)
                else:
                    logger.warning(f"Position limit reached, skipping {trade['symbol']}")

    def should_copy_trade(self, trade):
        """
        Filter logic for determining whether to copy a trade
        """
        # 1. Check excluded symbols
        if trade['symbol'] in self.excluded_symbols:
            return False

        # 2. Check minimum trade value
        trade_value = trade['quantity'] * trade['price']
        if trade_value < self.min_trade_value:
            return False

        # 3. Check if it's a closing trade and we don't have position
        if trade['side'] == 'SELL':
            if trade['symbol'] not in self.position_tracker.positions:
                return False  # Don't short if leader is just closing

        return True

    def normalize_trade(self, raw_event):
        """
        Normalize trade data from broker API to standard format
        """
        return {
            'symbol': raw_event.get('symbol') or raw_event.get('ticker'),
            'side': self.normalize_side(raw_event.get('side')),
            'quantity': float(raw_event.get('qty') or raw_event.get('quantity')),
            'price': float(raw_event.get('price') or raw_event.get('fill_price')),
            'timestamp': self.parse_timestamp(raw_event.get('timestamp')),
            'order_type': raw_event.get('order_type', 'market'),
            'time_in_force': raw_event.get('time_in_force', 'day')
        }
```

**Trade Filtering Logic**:

| Filter Type | Purpose | Example |
|------------|---------|---------|
| Symbol Exclusion | Avoid certain tickers | Exclude penny stocks, ETFs |
| Minimum Value | Filter out small trades | Only copy trades > $500 |
| Position Limit | Risk management | Max 10 simultaneous positions |
| Market Hours | Prevent after-hours execution | Only copy during RTH |
| Account Balance | Prevent overtrading | Require 20% free cash |
| Symbol Type | Asset class filtering | Only stocks, no options |

---

## 4. Follower Trade Execution

### 4.1 Solana Implementation

**Execution Flow**:
```
1. Receive trade signal
2. Validate token accounts exist
3. Build swap instruction (protocol-specific)
4. Add recent blockhash
5. Sign transaction
6. Submit via RPC or ZeroSlot (MEV protection)
7. Monitor confirmation
8. Update position tracking
```

**Protocol Adapter Pattern** (`src/dex/pump_fun.rs`):
```rust
impl Pump {
    pub fn build_buy_instruction(
        &self,
        wallet: &Keypair,
        mint: &Pubkey,
        sol_amount: u64,
        max_slippage: u64,
    ) -> Result<Instruction> {
        // 1. Calculate expected tokens with bonding curve
        let expected_tokens = self.calculate_buy_token_amount(
            sol_amount,
            virtual_sol_reserves,
            virtual_token_reserves
        );

        // 2. Apply slippage tolerance
        let min_tokens = expected_tokens * (10000 - max_slippage) / 10000;

        // 3. Build instruction with proper accounts
        let accounts = vec![
            AccountMeta::new(*global, false),
            AccountMeta::new(*fee_recipient, false),
            AccountMeta::new(*mint, false),
            // ... more accounts
        ];

        Ok(Instruction {
            program_id: PUMP_FUN_PROGRAM,
            accounts,
            data: instruction_data,
        })
    }
}
```

### 4.2 Equities Adaptation

**Execution Flow**:
```
1. Receive leader trade signal
2. Calculate follower position size
3. Validate account buying power
4. Check symbol tradability
5. Submit order to broker API
6. Monitor order status
7. Update position tracking
8. Record execution for P&L
```

**Position Sizing Strategies**:

| Strategy | Description | Formula |
|----------|-------------|---------|
| Fixed Ratio | Maintain same portfolio % as leader | follower_qty = (follower_account / leader_account) × leader_qty |
| Fixed Dollar | Use fixed dollar amount per trade | follower_qty = fixed_amount / current_price |
| Kelly Criterion | Risk-based sizing | position_size = (win_rate × avg_win - loss_rate × avg_loss) / avg_win |
| Volatility-Adjusted | Scale based on stock volatility | base_size / (ATR / price) |

**Implementation Pattern**:
```python
class FollowerOrderExecutor:
    def __init__(self, broker_api, config):
        self.broker = broker_api
        self.sizing_method = config['position_sizing']
        self.max_position_pct = config.get('max_position_pct', 0.10)  # 10% max

    async def execute_follower_order(self, leader_trade):
        """
        Execute follower order based on leader trade
        """
        # 1. Calculate position size
        follower_qty = self.calculate_position_size(
            leader_trade['quantity'],
            leader_trade['symbol']
        )

        # 2. Pre-execution validation
        if not await self.validate_order(leader_trade['symbol'], follower_qty):
            logger.error(f"Order validation failed for {leader_trade['symbol']}")
            return None

        # 3. Determine order type and parameters
        order_params = self.build_order_params(leader_trade, follower_qty)

        # 4. Submit order
        try:
            order = await self.broker.submit_order(**order_params)

            # 5. Monitor order fill
            filled_order = await self.monitor_order_fill(order.id, timeout=30)

            # 6. Update position tracking
            self.update_position_tracking(filled_order)

            return filled_order

        except InsufficientBuyingPower:
            logger.error("Insufficient buying power")
            return None
        except SymbolNotTradable:
            logger.error(f"{leader_trade['symbol']} not tradable")
            return None

    def calculate_position_size(self, leader_qty, symbol):
        """
        Calculate appropriate follower position size
        """
        if self.sizing_method == 'fixed_ratio':
            # Proportional to account size
            ratio = self.follower_account_value / self.leader_account_value
            return int(leader_qty * ratio)

        elif self.sizing_method == 'fixed_dollar':
            # Fixed dollar amount
            current_price = self.get_current_price(symbol)
            return int(self.fixed_amount / current_price)

        elif self.sizing_method == 'volatility_adjusted':
            # Adjust for stock volatility
            atr = self.get_atr(symbol, period=14)
            current_price = self.get_current_price(symbol)
            volatility_ratio = atr / current_price
            base_size = int(self.base_amount / current_price)
            return int(base_size / volatility_ratio) if volatility_ratio > 0 else base_size

    async def validate_order(self, symbol, quantity):
        """
        Pre-execution validation checks
        """
        # 1. Check buying power
        account = await self.broker.get_account()
        required_capital = quantity * self.get_current_price(symbol)

        if float(account.buying_power) < required_capital:
            return False

        # 2. Check if symbol is tradable
        asset = await self.broker.get_asset(symbol)
        if not asset.tradable:
            return False

        # 3. Check position size limit
        position_value = required_capital
        account_value = float(account.equity)
        position_pct = position_value / account_value

        if position_pct > self.max_position_pct:
            logger.warning(f"Position would exceed {self.max_position_pct*100}% limit")
            return False

        return True

    def build_order_params(self, leader_trade, follower_qty):
        """
        Build order parameters for broker API
        """
        params = {
            'symbol': leader_trade['symbol'],
            'qty': follower_qty,
            'side': leader_trade['side'],
            'type': 'market',  # Start with market orders
            'time_in_force': 'day'
        }

        # Optional: Add limit order logic
        if self.use_limit_orders:
            current_price = self.get_current_price(leader_trade['symbol'])

            # Set limit price with buffer
            if leader_trade['side'] == 'buy':
                # Buy slightly above market to ensure fill
                params['type'] = 'limit'
                params['limit_price'] = round(current_price * 1.005, 2)  # 0.5% above
            else:
                # Sell slightly below market
                params['type'] = 'limit'
                params['limit_price'] = round(current_price * 0.995, 2)  # 0.5% below

        return params

    async def monitor_order_fill(self, order_id, timeout=30):
        """
        Monitor order until filled or timeout
        """
        start_time = time.time()

        while time.time() - start_time < timeout:
            order = await self.broker.get_order(order_id)

            if order.status == 'filled':
                return order
            elif order.status in ['canceled', 'rejected', 'expired']:
                raise OrderFailedError(f"Order {order_id} {order.status}")

            await asyncio.sleep(0.5)  # Check every 500ms

        # Timeout - cancel order
        await self.broker.cancel_order(order_id)
        raise OrderTimeoutError(f"Order {order_id} timed out")
```

**Order Type Selection**:

| Scenario | Order Type | Reasoning |
|----------|------------|-----------|
| Highly liquid (SPY, AAPL) | Market | Fast execution, minimal slippage |
| Moderate liquidity | Limit (0.5% buffer) | Control price, still likely to fill |
| Low liquidity | Limit (wider buffer) | Prevent excessive slippage |
| Volatile conditions | Limit with cancel/replace | Protect from adverse price movement |
| Opening trade (9:30-9:35 ET) | Limit | Avoid opening volatility |

---

## 5. Position & Risk Management

### 5.1 Solana Implementation

**Position Tracking** (`BoughtTokenInfo` struct):
```rust
pub struct BoughtTokenInfo {
    pub token_mint: String,
    pub entry_price: u64,
    pub current_price: u64,
    pub highest_price: u64,
    pub initial_amount: f64,
    pub pnl_percentage: f64,
    pub highest_pnl_percentage: f64,
    pub trailing_stop_percentage: f64,
    pub buy_timestamp: Instant,
}

// Global position registry (lock-free)
lazy_static! {
    pub static ref BOUGHT_TOKEN_LIST: Arc<DashMap<String, BoughtTokenInfo>>;
}
```

**Risk Management Service** (`src/processor/risk_management.rs`):
- Runs every 10 minutes
- Checks leader wallet balance
- Emergency sells if balance drops below threshold
- Independent from main trading loop

**Selling Strategy** (`src/processor/selling_strategy.rs`):
```rust
pub struct SellingConfig {
    pub take_profit: f64,              // 8.0 = 8% profit target
    pub stop_loss: f64,                // -2.0 = 2% loss limit
    pub max_hold_time: u64,            // 3600 seconds
    pub dynamic_trailing_stop_thresholds: Vec<(f64, f64)>,  // PnL -> Stop%
    pub copy_selling_limit: f64,       // 1.5 = sell when leader sells 1.5x position
}

impl SellingEngine {
    pub fn get_selling_action(&self, position: &BoughtTokenInfo) -> SellingAction {
        let pnl = position.pnl_percentage;
        let time_held = position.buy_timestamp.elapsed().as_secs();

        // 1. Stop loss
        if pnl <= self.config.stop_loss {
            return SellingAction::SellAll("Stop loss".to_string());
        }

        // 2. Take profit
        if pnl >= self.config.take_profit {
            return SellingAction::SellAll("Take profit".to_string());
        }

        // 3. Max hold time
        if time_held >= self.config.max_hold_time {
            return SellingAction::SellAll("Max hold time".to_string());
        }

        // 4. Dynamic trailing stop
        let trailing_stop = self.get_dynamic_trailing_stop(pnl);
        let drop_from_high = ((position.highest_price - position.current_price) as f64
                             / position.highest_price as f64) * 100.0;

        if drop_from_high >= trailing_stop {
            return SellingAction::SellAll("Trailing stop".to_string());
        }

        SellingAction::Hold
    }

    fn get_dynamic_trailing_stop(&self, current_pnl: f64) -> f64 {
        // Progressive trailing stops: higher profit = tighter stop
        // Example: 20% PnL -> 5% trail, 100% PnL -> 30% trail
        for (pnl_threshold, stop_pct) in &self.config.dynamic_trailing_stop_thresholds {
            if current_pnl >= *pnl_threshold {
                return *stop_pct;
            }
        }
        1.0  // Default 1% trailing stop
    }
}
```

### 5.2 Equities Adaptation

**Position Tracking**:
```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class EquityPosition:
    symbol: str
    entry_price: float
    quantity: int
    entry_time: datetime
    current_price: float
    highest_price: float
    lowest_price: float
    pnl_dollars: float
    pnl_percentage: float
    highest_pnl_percentage: float
    trailing_stop_price: Optional[float]
    order_id: str
    leader_position_size: int  # Track leader's position for copy-selling

    @property
    def position_value(self):
        return self.quantity * self.current_price

    @property
    def cost_basis(self):
        return self.quantity * self.entry_price

    def update_price(self, new_price: float):
        """Update position with new market price"""
        self.current_price = new_price

        # Update high/low
        if new_price > self.highest_price:
            self.highest_price = new_price
        if new_price < self.lowest_price:
            self.lowest_price = new_price

        # Update P&L
        self.pnl_dollars = (new_price - self.entry_price) * self.quantity
        self.pnl_percentage = ((new_price - self.entry_price) / self.entry_price) * 100

        # Update highest P&L reached
        if self.pnl_percentage > self.highest_pnl_percentage:
            self.highest_pnl_percentage = self.pnl_percentage

class PositionTracker:
    def __init__(self):
        self.positions = {}  # symbol -> EquityPosition
        self.lock = asyncio.Lock()

    async def add_position(self, filled_order):
        """Add new position from filled order"""
        async with self.lock:
            position = EquityPosition(
                symbol=filled_order.symbol,
                entry_price=float(filled_order.filled_avg_price),
                quantity=int(filled_order.filled_qty),
                entry_time=filled_order.filled_at,
                current_price=float(filled_order.filled_avg_price),
                highest_price=float(filled_order.filled_avg_price),
                lowest_price=float(filled_order.filled_avg_price),
                pnl_dollars=0.0,
                pnl_percentage=0.0,
                highest_pnl_percentage=0.0,
                trailing_stop_price=None,
                order_id=filled_order.id,
                leader_position_size=0  # Will update from leader API
            )
            self.positions[filled_order.symbol] = position

    async def update_from_market_data(self, symbol, current_price):
        """Update position with real-time price"""
        async with self.lock:
            if symbol in self.positions:
                self.positions[symbol].update_price(current_price)
```

**Exit Strategy Engine**:
```python
class ExitStrategyEngine:
    def __init__(self, config):
        self.take_profit_pct = config.get('take_profit', 0.08)  # 8%
        self.stop_loss_pct = config.get('stop_loss', -0.02)  # -2%
        self.max_hold_time = config.get('max_hold_time', 3600)  # seconds
        self.trailing_stop_levels = config.get('trailing_stops', {
            0.20: 0.05,   # 20% profit -> 5% trailing stop
            0.50: 0.10,   # 50% profit -> 10% trailing stop
            1.00: 0.30,   # 100% profit -> 30% trailing stop
        })
        self.copy_sell_trigger = config.get('copy_sell_trigger', 0.5)  # Sell when leader sells 50%

    def should_exit_position(self, position: EquityPosition, leader_position_size: int) -> tuple[bool, str]:
        """
        Determine if position should be closed
        Returns: (should_exit, reason)
        """
        # 1. Stop loss
        if position.pnl_percentage <= self.stop_loss_pct * 100:
            return (True, f"Stop loss triggered at {position.pnl_percentage:.2f}%")

        # 2. Take profit
        if position.pnl_percentage >= self.take_profit_pct * 100:
            return (True, f"Take profit triggered at {position.pnl_percentage:.2f}%")

        # 3. Max hold time
        hold_time = (datetime.now() - position.entry_time).total_seconds()
        if hold_time >= self.max_hold_time:
            return (True, f"Max hold time reached ({hold_time:.0f}s)")

        # 4. Dynamic trailing stop
        trailing_stop_pct = self.get_trailing_stop_percentage(position.pnl_percentage)
        drop_from_high = ((position.highest_price - position.current_price) / position.highest_price) * 100

        if drop_from_high >= trailing_stop_pct:
            return (True, f"Trailing stop triggered (drop {drop_from_high:.2f}% from high)")

        # 5. Copy-selling (leader reduced position)
        if leader_position_size < position.leader_position_size * self.copy_sell_trigger:
            reduction_pct = (1 - leader_position_size / position.leader_position_size) * 100
            return (True, f"Leader reduced position by {reduction_pct:.0f}%")

        return (False, "Hold")

    def get_trailing_stop_percentage(self, current_pnl_pct: float) -> float:
        """
        Get trailing stop % based on current P&L
        Higher profits get tighter stops to lock in gains
        """
        current_pnl_decimal = current_pnl_pct / 100

        for pnl_threshold, stop_pct in sorted(self.trailing_stop_levels.items(), reverse=True):
            if current_pnl_decimal >= pnl_threshold:
                return stop_pct * 100

        return 1.0  # Default 1% trailing stop

class RiskManager:
    """
    Background service for risk monitoring
    """
    def __init__(self, position_tracker, broker_api, config):
        self.positions = position_tracker
        self.broker = broker_api
        self.max_account_drawdown = config.get('max_drawdown', 0.10)  # 10%
        self.max_position_value_pct = config.get('max_position_pct', 0.20)  # 20%
        self.min_buying_power_pct = config.get('min_buying_power', 0.20)  # 20%

    async def monitor_risk_loop(self):
        """
        Background task to monitor portfolio-level risk
        """
        while True:
            try:
                await self.check_account_drawdown()
                await self.check_position_concentration()
                await self.check_buying_power()
                await asyncio.sleep(60)  # Check every minute
            except Exception as e:
                logger.error(f"Risk monitoring error: {e}")
                await asyncio.sleep(60)

    async def check_account_drawdown(self):
        """
        Emergency close if account drawdown exceeds limit
        """
        account = await self.broker.get_account()
        peak_value = float(account.equity)  # Would track actual peak
        current_value = float(account.equity)
        drawdown = (peak_value - current_value) / peak_value

        if drawdown >= self.max_account_drawdown:
            logger.critical(f"EMERGENCY: Drawdown {drawdown*100:.2f}% exceeds limit!")
            await self.emergency_liquidate_all()

    async def check_position_concentration(self):
        """
        Warn if single position is too large
        """
        account = await self.broker.get_account()
        account_value = float(account.equity)

        for symbol, position in self.positions.positions.items():
            position_pct = position.position_value / account_value

            if position_pct > self.max_position_value_pct:
                logger.warning(
                    f"Position {symbol} is {position_pct*100:.1f}% of account "
                    f"(limit: {self.max_position_value_pct*100}%)"
                )

    async def emergency_liquidate_all(self):
        """
        Close all positions immediately (market orders)
        """
        logger.critical("EMERGENCY LIQUIDATION INITIATED")

        for symbol, position in self.positions.positions.items():
            try:
                order = await self.broker.submit_order(
                    symbol=symbol,
                    qty=position.quantity,
                    side='sell',
                    type='market',
                    time_in_force='day'
                )
                logger.info(f"Emergency sell {symbol}: order {order.id}")
            except Exception as e:
                logger.error(f"Failed to liquidate {symbol}: {e}")
```

**Risk Checks Comparison**:

| Risk Type | Solana Bot | Equities Adaptation |
|-----------|------------|---------------------|
| Position Limit | Max concurrent tokens (COUNTER_LIMIT) | Max positions + max position % |
| Stop Loss | Fixed % from entry | Fixed % or ATR-based |
| Take Profit | Fixed % target | Fixed %, scaled exits, or trailing |
| Time Limit | Max hold time in seconds | Max hold time (seconds or days) |
| Leader Monitoring | Wallet balance check | Position size monitoring |
| Emergency Exit | Sell all if leader wallet depleted | Sell all on drawdown threshold |
| Concentration | Not implemented | Warn if position > 20% of account |

---

## 6. Configuration Management

### 6.1 Solana Implementation

**Environment Variables** (`src/common/config.rs`):
```rust
pub struct Config {
    // Connection settings
    pub yellowstone_grpc_http: String,
    pub yellowstone_grpc_token: String,
    pub rpc_http: String,

    // Trading parameters
    pub token_amount: f64,
    pub slippage: u64,
    pub counter_limit: u32,

    // Targeting
    pub copy_trading_target_address: Vec<String>,
    pub excluded_addresses: Vec<String>,

    // Selling strategy
    pub take_profit: f64,
    pub stop_loss: f64,
    pub max_hold_time: u64,
    pub dynamic_trailing_stop_thresholds: Vec<(f64, f64)>,

    // Execution mode
    pub transaction_landing_mode: TransactionLandingMode,
    pub zero_slot_tip_value: f64,
}

impl Config {
    pub async fn new() -> Self {
        dotenv().ok();

        let yellowstone_grpc_http = import_env_var("YELLOWSTONE_GRPC_HTTP");
        let slippage = import_env_var("SLIPPAGE").parse::<u64>().unwrap_or(5000);
        let counter_limit = import_env_var("COUNTER_LIMIT").parse::<u32>().unwrap_or(10);

        // ... load all config
    }
}
```

### 6.2 Equities Adaptation

**Configuration Structure**:
```python
from pydantic import BaseSettings, Field
from typing import List, Dict

class LeaderConfig(BaseSettings):
    """Configuration for leader account monitoring"""
    api_key: str = Field(..., env='LEADER_API_KEY')
    api_secret: str = Field(..., env='LEADER_API_SECRET')
    account_id: str = Field(..., env='LEADER_ACCOUNT_ID')
    base_url: str = Field('https://api.alpaca.markets', env='LEADER_API_URL')

    # Filtering
    excluded_symbols: List[str] = Field(default_factory=list, env='EXCLUDED_SYMBOLS')
    min_trade_value: float = Field(100.0, env='MIN_TRADE_VALUE')

    class Config:
        env_file = '.env'

class FollowerConfig(BaseSettings):
    """Configuration for follower account execution"""
    api_key: str = Field(..., env='FOLLOWER_API_KEY')
    api_secret: str = Field(..., env='FOLLOWER_API_SECRET')
    account_id: str = Field(..., env='FOLLOWER_ACCOUNT_ID')
    base_url: str = Field('https://api.alpaca.markets', env='FOLLOWER_API_URL')

    # Position sizing
    sizing_method: str = Field('fixed_ratio', env='POSITION_SIZING_METHOD')
    fixed_amount: float = Field(1000.0, env='FIXED_POSITION_AMOUNT')
    max_position_pct: float = Field(0.10, env='MAX_POSITION_PCT')

    class Config:
        env_file = '.env'

class TradingConfig(BaseSettings):
    """Trading strategy configuration"""
    max_positions: int = Field(10, env='MAX_POSITIONS')

    # Exit strategy
    take_profit: float = Field(0.08, env='TAKE_PROFIT')
    stop_loss: float = Field(-0.02, env='STOP_LOSS')
    max_hold_time: int = Field(3600, env='MAX_HOLD_TIME')

    # Trailing stops (JSON string in env)
    trailing_stops: Dict[float, float] = Field(
        default_factory=lambda: {0.20: 0.05, 0.50: 0.10, 1.00: 0.30},
        env='TRAILING_STOPS'
    )

    # Copy selling
    copy_sell_trigger: float = Field(0.5, env='COPY_SELL_TRIGGER')

    # Order execution
    order_type: str = Field('market', env='ORDER_TYPE')
    use_limit_with_buffer: bool = Field(False, env='USE_LIMIT_ORDERS')
    limit_price_buffer: float = Field(0.005, env='LIMIT_PRICE_BUFFER')

    class Config:
        env_file = '.env'

class RiskConfig(BaseSettings):
    """Risk management configuration"""
    max_account_drawdown: float = Field(0.10, env='MAX_DRAWDOWN')
    min_buying_power_pct: float = Field(0.20, env='MIN_BUYING_POWER')
    max_position_value_pct: float = Field(0.20, env='MAX_POSITION_VALUE_PCT')

    class Config:
        env_file = '.env'

class CopyTradingConfig:
    """Main configuration aggregator"""
    def __init__(self):
        self.leader = LeaderConfig()
        self.follower = FollowerConfig()
        self.trading = TradingConfig()
        self.risk = RiskConfig()

    @classmethod
    def from_env(cls):
        return cls()
```

**Example `.env` file**:
```env
# Leader Account (Read-Only)
LEADER_API_KEY=PK...
LEADER_API_SECRET=...
LEADER_ACCOUNT_ID=...
LEADER_API_URL=https://api.alpaca.markets

# Follower Account (Execution)
FOLLOWER_API_KEY=PK...
FOLLOWER_API_SECRET=...
FOLLOWER_ACCOUNT_ID=...
FOLLOWER_API_URL=https://paper-api.alpaca.markets  # Use paper for testing!

# Filtering
EXCLUDED_SYMBOLS=SPY,QQQ,TQQQ  # Don't copy leveraged ETFs
MIN_TRADE_VALUE=500.0

# Position Sizing
POSITION_SIZING_METHOD=fixed_ratio  # fixed_ratio | fixed_dollar | volatility_adjusted
FIXED_POSITION_AMOUNT=1000.0
MAX_POSITION_PCT=0.10

# Trading Limits
MAX_POSITIONS=10

# Exit Strategy
TAKE_PROFIT=0.08
STOP_LOSS=-0.02
MAX_HOLD_TIME=3600

# Trailing Stops (JSON format)
TRAILING_STOPS={"0.20": 0.05, "0.50": 0.10, "1.00": 0.30}

# Copy Selling
COPY_SELL_TRIGGER=0.5  # Sell when leader sells 50% of position

# Order Execution
ORDER_TYPE=market
USE_LIMIT_ORDERS=false
LIMIT_PRICE_BUFFER=0.005

# Risk Management
MAX_DRAWDOWN=0.10
MIN_BUYING_POWER=0.20
MAX_POSITION_VALUE_PCT=0.20
```

---

## 7. Data Models & State Tracking

### 7.1 Solana Implementation

**Global State (Lock-Free DashMaps)**:
```rust
lazy_static! {
    // Position tracking
    pub static ref BOUGHT_TOKEN_LIST: Arc<DashMap<String, BoughtTokenInfo>>;

    // Price monitoring cancellation
    static ref PRICE_MONITORING_TASKS: Arc<DashMap<String, CancellationToken>>;

    // Sold tokens blacklist (prevent re-entry)
    static ref BOUGHT_TOKENS_BLACKLIST: Arc<DashMap<String, u64>>;

    // Token account cache (LRU)
    pub static ref TOKEN_ACCOUNT_CACHE: LruCache<Pubkey, bool>;

    // Timeseries for price/volume
    pub static ref TOKEN_TIMESERIES: Arc<DashMap<String, TokenTimeseries>>;
}
```

### 7.2 Equities Adaptation

**State Management**:
```python
import asyncio
from collections import defaultdict
from datetime import datetime
from typing import Dict, List

class ApplicationState:
    """
    Central state management for copy trading system
    """
    def __init__(self):
        # Position tracking
        self.positions: Dict[str, EquityPosition] = {}
        self.closed_positions: List[EquityPosition] = []

        # Order tracking
        self.pending_orders: Dict[str, Order] = {}
        self.filled_orders: Dict[str, Order] = {}

        # Leader position tracking (for copy-selling)
        self.leader_positions: Dict[str, int] = {}  # symbol -> quantity

        # Symbol blacklist (sold positions, don't re-enter)
        self.symbol_blacklist: Dict[str, datetime] = {}

        # Price cache (real-time quotes)
        self.price_cache: Dict[str, float] = {}
        self.price_cache_timestamp: Dict[str, datetime] = {}

        # Statistics
        self.total_trades = 0
        self.winning_trades = 0
        self.losing_trades = 0
        self.total_pnl = 0.0

        # Thread safety
        self.lock = asyncio.Lock()

    async def update_position_price(self, symbol: str, price: float):
        """Update position with latest price"""
        async with self.lock:
            if symbol in self.positions:
                self.positions[symbol].update_price(price)

            # Update cache
            self.price_cache[symbol] = price
            self.price_cache_timestamp[symbol] = datetime.now()

    async def close_position(self, symbol: str, exit_price: float, reason: str):
        """Close a position and record statistics"""
        async with self.lock:
            if symbol not in self.positions:
                return

            position = self.positions[symbol]
            position.update_price(exit_price)

            # Record statistics
            self.total_trades += 1
            if position.pnl_dollars > 0:
                self.winning_trades += 1
            else:
                self.losing_trades += 1

            self.total_pnl += position.pnl_dollars

            # Move to closed positions
            self.closed_positions.append(position)
            del self.positions[symbol]

            # Add to blacklist (prevent immediate re-entry)
            self.symbol_blacklist[symbol] = datetime.now()

            logger.info(
                f"Closed {symbol}: "
                f"P&L ${position.pnl_dollars:.2f} ({position.pnl_percentage:.2f}%) "
                f"Reason: {reason}"
            )

    async def should_trade_symbol(self, symbol: str) -> bool:
        """Check if symbol is eligible for trading"""
        async with self.lock:
            # Check if already have position
            if symbol in self.positions:
                return False

            # Check blacklist (recently closed)
            if symbol in self.symbol_blacklist:
                blacklist_time = self.symbol_blacklist[symbol]
                if (datetime.now() - blacklist_time).total_seconds() < 3600:  # 1 hour
                    return False
                else:
                    # Remove from blacklist after cooldown
                    del self.symbol_blacklist[symbol]

            return True

    def get_statistics(self) -> Dict:
        """Get trading statistics"""
        win_rate = self.winning_trades / self.total_trades if self.total_trades > 0 else 0
        avg_win = sum(p.pnl_dollars for p in self.closed_positions if p.pnl_dollars > 0) / self.winning_trades if self.winning_trades > 0 else 0
        avg_loss = sum(p.pnl_dollars for p in self.closed_positions if p.pnl_dollars < 0) / self.losing_trades if self.losing_trades > 0 else 0

        return {
            'total_trades': self.total_trades,
            'winning_trades': self.winning_trades,
            'losing_trades': self.losing_trades,
            'win_rate': win_rate,
            'total_pnl': self.total_pnl,
            'avg_win': avg_win,
            'avg_loss': avg_loss,
            'current_positions': len(self.positions),
            'total_position_value': sum(p.position_value for p in self.positions.values())
        }
```

---

## 8. Error Handling & Retry Logic

### 8.1 Solana Implementation

**Transaction Retry** (`src/processor/transaction_retry.rs`):
```rust
pub async fn retry_transaction_until_confirmed(
    signature: &Signature,
    rpc_client: &RpcClient,
    max_retries: u32,
) -> Result<bool> {
    let mut retries = 0;

    while retries < max_retries {
        match rpc_client.get_signature_status(signature) {
            Ok(Some(status)) => {
                if let Some(confirmation_status) = status {
                    if confirmation_status == TransactionConfirmationStatus::Finalized {
                        return Ok(true);
                    }
                }
            }
            Ok(None) => {
                // Transaction not found, might need to resend
                logger.warning("Transaction not found, resending...");
            }
            Err(e) => {
                logger.error(&format!("Error checking transaction: {:?}", e));
            }
        }

        tokio::time::sleep(Duration::from_millis(500)).await;
        retries += 1;
    }

    Ok(false)
}
```

### 8.2 Equities Adaptation

**Retry Strategy**:
```python
import asyncio
from functools import wraps
from typing import Callable, Type

class RetryableError(Exception):
    """Base class for errors that should trigger retry"""
    pass

class RateLimitError(RetryableError):
    """API rate limit exceeded"""
    pass

class NetworkError(RetryableError):
    """Network connectivity issue"""
    pass

class TemporaryAPIError(RetryableError):
    """Temporary API issue (500, 503)"""
    pass

def async_retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple = (RetryableError,)
):
    """
    Decorator for automatic retry with exponential backoff
    """
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            attempt = 0
            current_delay = delay

            while attempt < max_attempts:
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    attempt += 1
                    if attempt >= max_attempts:
                        logger.error(f"{func.__name__} failed after {max_attempts} attempts")
                        raise

                    logger.warning(
                        f"{func.__name__} attempt {attempt} failed: {e}. "
                        f"Retrying in {current_delay}s..."
                    )
                    await asyncio.sleep(current_delay)
                    current_delay *= backoff

        return wrapper
    return decorator

class OrderExecutor:
    @async_retry(max_attempts=3, delay=1.0, exceptions=(RateLimitError, NetworkError))
    async def submit_order_with_retry(self, order_params):
        """
        Submit order with automatic retry on transient errors
        """
        try:
            order = await self.broker.submit_order(**order_params)
            return order
        except RateLimitError as e:
            logger.warning("Rate limit hit, will retry")
            raise
        except NetworkError as e:
            logger.warning("Network error, will retry")
            raise
        except InsufficientBuyingPower:
            # Don't retry - this is a permanent error
            logger.error("Insufficient buying power")
            raise

    async def monitor_order_with_timeout(self, order_id: str, timeout: int = 30):
        """
        Monitor order status with timeout and retry logic
        """
        start_time = asyncio.get_event_loop().time()
        check_interval = 0.5
        max_check_attempts = 3

        while asyncio.get_event_loop().time() - start_time < timeout:
            attempt = 0

            while attempt < max_check_attempts:
                try:
                    order = await self.broker.get_order(order_id)

                    if order.status == 'filled':
                        return order
                    elif order.status in ['canceled', 'rejected', 'expired']:
                        raise OrderFailedError(f"Order {order_id} {order.status}")

                    break  # Successful check, continue monitoring

                except (NetworkError, TemporaryAPIError):
                    attempt += 1
                    if attempt >= max_check_attempts:
                        raise
                    await asyncio.sleep(0.5)

            await asyncio.sleep(check_interval)

        # Timeout reached
        try:
            await self.broker.cancel_order(order_id)
        except:
            pass  # Best effort cancel

        raise OrderTimeoutError(f"Order {order_id} timed out after {timeout}s")

class APIHealthMonitor:
    """
    Monitor API health and switch to fallback if needed
    """
    def __init__(self, primary_api, fallback_apis):
        self.primary = primary_api
        self.fallbacks = fallback_apis
        self.current_api = primary_api
        self.failure_count = 0
        self.max_failures = 3

    async def execute_with_fallback(self, operation: Callable):
        """
        Execute operation with automatic fallback
        """
        apis_to_try = [self.current_api] + self.fallbacks

        for api in apis_to_try:
            try:
                result = await operation(api)

                # Reset failure count on success
                if api == self.primary:
                    self.failure_count = 0

                return result

            except (NetworkError, TemporaryAPIError) as e:
                logger.warning(f"API {api} failed: {e}")

                if api == self.primary:
                    self.failure_count += 1
                    if self.failure_count >= self.max_failures:
                        logger.critical("Switching to fallback API")
                        self.current_api = self.fallbacks[0] if self.fallbacks else self.primary

                continue

        raise AllAPIsFailedError("All API endpoints failed")
```

**Error Handling Best Practices**:

| Error Type | Strategy | Example |
|------------|----------|---------|
| Rate Limit | Exponential backoff retry | Wait 1s, 2s, 4s, 8s |
| Network Timeout | Retry with increasing timeout | 5s -> 10s -> 15s |
| Insufficient Buying Power | Don't retry, log and skip | Permanent error |
| Symbol Not Tradable | Don't retry, add to blacklist | Permanent error |
| API 500 Error | Retry up to 3 times | Transient error |
| API 400 Error | Don't retry, log details | Client error (bad request) |
| Order Timeout | Cancel and retry with limit order | Use limit + buffer |
| Connection Lost | Reconnect with exponential backoff | Maintain state |

---

## 9. Performance Optimizations

### 9.1 Solana Optimizations

1. **Blockhash Caching**: Updates every 300ms in background
2. **Token Account Cache**: LRU cache (1000 entries)
3. **Lock-Free Data Structures**: DashMap for concurrent access
4. **Parallel Monitoring**: Two independent gRPC streams
5. **Connection Pooling**: Reuse RPC connections
6. **MEV Protection**: ZeroSlot for guaranteed execution slot

### 9.2 Equities Optimizations

```python
class PerformanceOptimizations:
    """
    Performance enhancements for equities copy trading
    """

    # 1. Price Cache with TTL
    class PriceCache:
        def __init__(self, ttl_seconds=1.0):
            self.cache = {}
            self.timestamps = {}
            self.ttl = ttl_seconds

        def get(self, symbol: str) -> float:
            if symbol in self.cache:
                if time.time() - self.timestamps[symbol] < self.ttl:
                    return self.cache[symbol]
            return None

        def set(self, symbol: str, price: float):
            self.cache[symbol] = price
            self.timestamps[symbol] = time.time()

    # 2. Batch Order Submission
    async def submit_orders_batch(self, orders: List[dict]):
        """
        Submit multiple orders in parallel
        """
        tasks = [
            self.broker.submit_order(**order_params)
            for order_params in orders
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)

    # 3. WebSocket Connection Pooling
    class WebSocketPool:
        def __init__(self, max_connections=5):
            self.connections = []
            self.max_connections = max_connections
            self.semaphore = asyncio.Semaphore(max_connections)

        async def get_connection(self):
            async with self.semaphore:
                if self.connections:
                    return self.connections.pop()
                return await self.create_new_connection()

        async def release_connection(self, conn):
            if len(self.connections) < self.max_connections:
                self.connections.append(conn)
            else:
                await conn.close()

    # 4. Asynchronous Position Updates
    async def batch_update_positions(self, symbols: List[str]):
        """
        Update all positions concurrently
        """
        async def update_single(symbol):
            try:
                price = await self.market_data_api.get_latest_price(symbol)
                await self.state.update_position_price(symbol, price)
            except Exception as e:
                logger.error(f"Failed to update {symbol}: {e}")

        await asyncio.gather(*[update_single(s) for s in symbols])

    # 5. Smart Polling (Adaptive Intervals)
    class AdaptivePoller:
        def __init__(self):
            self.interval = 5.0  # Start with 5s
            self.min_interval = 1.0
            self.max_interval = 30.0

        async def poll_with_adaptation(self, fetch_func):
            while True:
                start = time.time()

                data = await fetch_func()

                # Adapt interval based on data freshness
                if data['has_new_data']:
                    # Reduce interval if activity detected
                    self.interval = max(self.min_interval, self.interval * 0.8)
                else:
                    # Increase interval if no activity
                    self.interval = min(self.max_interval, self.interval * 1.2)

                elapsed = time.time() - start
                sleep_time = max(0, self.interval - elapsed)
                await asyncio.sleep(sleep_time)
```

**Performance Comparison**:

| Aspect | Solana (gRPC) | Equities (REST/WS) | Optimization |
|--------|---------------|---------------------|--------------|
| Latency | ~50-100ms | ~100-500ms | Use WebSocket for real-time |
| Throughput | ~1000 tx/s | ~50 orders/s | Batch operations where possible |
| Connection | Persistent gRPC | REST polling or WS | Maintain persistent WebSocket |
| State Updates | Lock-free DashMap | Async with locks | Use asyncio.Lock sparingly |
| Price Updates | Real-time stream | Poll or WS stream | Cache with 1s TTL |

---

## 10. Equities-Specific Adaptations

### 10.1 Market Hours Handling

```python
from datetime import datetime, time
import pytz

class MarketHoursManager:
    def __init__(self):
        self.timezone = pytz.timezone('America/New_York')
        self.market_open = time(9, 30)  # 9:30 AM ET
        self.market_close = time(16, 0)  # 4:00 PM ET

    def is_market_open(self) -> bool:
        """Check if market is currently open"""
        now = datetime.now(self.timezone)

        # Check if weekend
        if now.weekday() >= 5:  # Saturday=5, Sunday=6
            return False

        # Check time
        current_time = now.time()
        return self.market_open <= current_time <= self.market_close

    def time_until_market_open(self) -> int:
        """Seconds until market opens"""
        now = datetime.now(self.timezone)

        # Calculate next market open
        next_open = now.replace(hour=9, minute=30, second=0, microsecond=0)

        # If after close, move to next day
        if now.time() > self.market_close:
            next_open += timedelta(days=1)

        # Skip weekends
        while next_open.weekday() >= 5:
            next_open += timedelta(days=1)

        return int((next_open - now).total_seconds())

    async def wait_for_market_open(self):
        """Block until market opens"""
        while not self.is_market_open():
            wait_time = self.time_until_market_open()
            logger.info(f"Market closed. Opening in {wait_time}s")
            await asyncio.sleep(min(wait_time, 60))  # Check every minute
```

### 10.2 Regulatory Compliance

```python
class ComplianceChecker:
    """
    Ensure trading complies with regulations
    """
    def __init__(self, account_type='cash'):
        self.account_type = account_type  # 'cash' or 'margin'
        self.daily_trades = []
        self.pdt_threshold = 3  # Pattern Day Trader threshold

    def check_pattern_day_trader(self) -> bool:
        """
        Check if account would trigger PDT rule (US)
        PDT = 4+ day trades in 5 business days
        """
        if self.account_type == 'margin':
            # Count day trades in last 5 business days
            recent_day_trades = [
                t for t in self.daily_trades
                if (datetime.now() - t['date']).days <= 5
            ]

            if len(recent_day_trades) >= self.pdt_threshold:
                logger.warning("Approaching Pattern Day Trader threshold")
                return True

        return False

    def can_day_trade(self, symbol: str) -> bool:
        """Check if day trade is allowed"""
        if self.account_type == 'cash':
            # Cash accounts: must wait T+2 for funds
            # Check if we have unsettled funds
            return self.has_settled_funds()

        elif self.account_type == 'margin':
            # Margin accounts: PDT rule applies
            return not self.check_pattern_day_trader()

    def has_settled_funds(self) -> bool:
        """Check if cash account has settled funds"""
        # Implement based on broker API
        # Alpaca provides 'cash' vs 'buying_power'
        pass
```

### 10.3 Order Routing

```python
class SmartOrderRouter:
    """
    Intelligent order routing based on market conditions
    """
    def __init__(self):
        self.market_hours_mgr = MarketHoursManager()

    def get_optimal_order_params(self, symbol: str, side: str, quantity: int) -> dict:
        """
        Determine optimal order parameters based on conditions
        """
        current_time = datetime.now(pytz.timezone('America/New_York')).time()

        # Get current market data
        quote = self.get_current_quote(symbol)
        spread_pct = (quote['ask'] - quote['bid']) / quote['mid'] * 100

        # Determine order type
        if current_time < time(9, 35):
            # Opening 5 minutes: use limit orders
            order_type = 'limit'
            if side == 'buy':
                limit_price = quote['ask'] * 1.002  # 0.2% above ask
            else:
                limit_price = quote['bid'] * 0.998  # 0.2% below bid

        elif current_time > time(15, 45):
            # Closing 15 minutes: use market orders for quick fill
            order_type = 'market'
            limit_price = None

        elif spread_pct > 1.0:
            # Wide spread: use limit order
            order_type = 'limit'
            if side == 'buy':
                limit_price = quote['bid'] + (quote['ask'] - quote['bid']) * 0.3
            else:
                limit_price = quote['ask'] - (quote['ask'] - quote['bid']) * 0.3

        else:
            # Normal conditions: market order
            order_type = 'market'
            limit_price = None

        params = {
            'symbol': symbol,
            'qty': quantity,
            'side': side,
            'type': order_type,
            'time_in_force': 'day'
        }

        if order_type == 'limit':
            params['limit_price'] = round(limit_price, 2)

        return params
```

### 10.4 Extended Hours Trading

```python
class ExtendedHoursTrader:
    """
    Handle pre-market and after-hours trading
    """
    def __init__(self, enable_extended_hours=False):
        self.enable_extended_hours = enable_extended_hours
        self.premarket_start = time(4, 0)   # 4:00 AM ET
        self.premarket_end = time(9, 30)    # 9:30 AM ET
        self.afterhours_start = time(16, 0) # 4:00 PM ET
        self.afterhours_end = time(20, 0)   # 8:00 PM ET

    def is_extended_hours(self) -> tuple[bool, str]:
        """
        Check if currently in extended hours
        Returns: (is_extended, session_type)
        """
        now = datetime.now(pytz.timezone('America/New_York')).time()

        if self.premarket_start <= now < self.premarket_end:
            return (True, 'pre-market')
        elif self.afterhours_start <= now < self.afterhours_end:
            return (True, 'after-hours')
        else:
            return (False, 'regular')

    def should_copy_extended_hours_trade(self, leader_trade) -> bool:
        """
        Decide whether to copy a trade during extended hours
        """
        if not self.enable_extended_hours:
            return False

        is_extended, session = self.is_extended_hours()

        if not is_extended:
            return True  # Regular hours, always copy

        # In extended hours, apply stricter filters
        # Only copy if:
        # 1. Sufficient liquidity
        # 2. Not too volatile
        # 3. Symbol is active in extended hours

        return self.is_liquid_in_extended_hours(leader_trade['symbol'])

    def modify_order_for_extended_hours(self, order_params):
        """
        Adjust order parameters for extended hours
        """
        is_extended, session = self.is_extended_hours()

        if is_extended:
            # Use extended_hours flag
            order_params['extended_hours'] = True

            # Wider limits due to lower liquidity
            if order_params['type'] == 'limit':
                order_params['limit_price'] *= 1.01 if order_params['side'] == 'buy' else 0.99

        return order_params
```

---

## Summary: Key Takeaways for Equities Copy Trading

### Architecture
✅ **Dual API Connection Pattern**: Separate read (leader) and write (follower) API clients
✅ **Asynchronous Design**: Use async/await for concurrent operations
✅ **Background Services**: Health checks, position updates, risk monitoring
✅ **Lock-Free State**: Minimize locking with proper async patterns

### Monitoring
✅ **Real-Time Subscriptions**: WebSocket preferred over polling
✅ **Trade Filtering**: Exclude symbols, minimum trade value, position limits
✅ **Dual Monitoring**: Leader trades + market data for positions
✅ **Heartbeat Mechanism**: Keep connections alive

### Execution
✅ **Position Sizing**: Fixed ratio, fixed dollar, or volatility-adjusted
✅ **Pre-Execution Validation**: Buying power, tradability, position limits
✅ **Smart Order Types**: Market for liquid, limit for wide spreads
✅ **Order Monitoring**: Track until filled with timeout

### Risk Management
✅ **Stop Loss**: Fixed percentage or ATR-based
✅ **Take Profit**: Fixed target or scaled exits
✅ **Dynamic Trailing Stops**: Tighter stops as profit increases
✅ **Copy-Selling**: Exit when leader reduces position
✅ **Portfolio Limits**: Max positions, max position %, max drawdown

### Configuration
✅ **Environment-Based**: Use .env for credentials and parameters
✅ **Pydantic Models**: Type-safe configuration loading
✅ **Secrets Management**: Never commit API keys

### Error Handling
✅ **Automatic Retry**: Exponential backoff for transient errors
✅ **Fallback APIs**: Switch to backup endpoints on failure
✅ **Graceful Degradation**: Continue with reduced functionality

### Equities-Specific
✅ **Market Hours**: Check before executing trades
✅ **Compliance**: PDT rule, settled funds (cash accounts)
✅ **Extended Hours**: Optional pre-market/after-hours
✅ **Order Routing**: Adapt based on time of day and spread

---

## Next Steps

1. **Choose Broker APIs**: Alpaca, Interactive Brokers, or TD Ameritrade for both leader & follower
2. **Implement Core**: Start with basic monitoring + execution
3. **Add Risk Management**: Position limits, stop loss, take profit
4. **Test with Paper Trading**: Use paper accounts before live
5. **Monitor & Iterate**: Log everything, analyze performance, adjust parameters

---

*This analysis extracts production-grade patterns from a sophisticated Solana copy trading bot and adapts them for equities markets. The core architectural principles remain the same: real-time monitoring, intelligent execution, robust risk management, and resilient error handling.*
