# Indicator-Based Strategy Guide

Build strategies triggered by technical indicators (RSI, MACD, Bollinger Bands, EMA crossover, etc.) using the **3-layer pattern**.

> **Prerequisite**: Read [STRATEGY_API.md](../STRATEGY_API.md) first for the base Strategy trait and wiring.

---

## Architecture: 3 Layers

Every indicator strategy separates concerns into 3 layers:

```
Quote ticks → [BarBuilder] → OHLCV bar → [Indicator] → signal → [Strategy] → orders
```

| Layer          | File           | Responsibility                                        | Reusable?                                 |
| -------------- | -------------- | ----------------------------------------------------- | ----------------------------------------- |
| **BarBuilder** | `bar.rs`       | Aggregates streaming ticks into time-based OHLCV bars | ✅ Copy as-is into any indicator strategy |
| **Indicator**  | `indicator.rs` | Pure math — takes `f64`, returns `Option<f64>`        | ✅ Swap for any indicator                 |
| **Strategy**   | `strategy.rs`  | Wires indicator + thresholds + order lifecycle        | Adapt per strategy                        |

---

## Layer 1: BarBuilder (reusable)

Converts the tick stream into OHLCV bars of configurable interval. Copy this into any indicator strategy.

```rust
pub struct Bar {
    pub open: f64,
    pub high: f64,
    pub low: f64,
    pub close: f64,
}

pub struct BarBuilder {
    interval_ms: i64,
    current_bar: Option<PartialBar>,
}

impl BarBuilder {
    pub fn new(interval_secs: u64) -> Self { ... }

    /// Feed a tick. Returns Some(Bar) when the interval elapses.
    pub fn update(&mut self, price: f64, timestamp_ms: i64) -> Option<Bar> {
        // Opens a new bar on first tick, updates H/L/C on subsequent ticks,
        // emits completed Bar when timestamp crosses the interval boundary.
    }
}
```

**Key behavior**: The bar closes on the first tick _after_ the interval expires, so it captures real market data timing.

**Config parameter**: `bar_interval_secs` — controls bar duration (5s for testing, 60s+ for production).

---

## Layer 2: Indicator (swap this)

The indicator is pure math with no bot dependencies. Interface contract:

```rust
pub struct MyIndicator {
    // Internal state (period, averages, buffers, etc.)
}

impl MyIndicator {
    /// Create with configurable parameters
    pub fn new(period: usize) -> Self { ... }

    /// Feed one completed bar's close price.
    /// Returns None during warmup, Some(value) once converged.
    pub fn update(&mut self, close: f64) -> Option<f64> { ... }
}
```

### Built-in: Wilder's RSI

The `strategy-rsi` crate includes a production-ready RSI indicator:

```rust
use crate::indicator::Rsi;

let mut rsi = Rsi::new(14); // 14-period RSI
// Returns None for first 14 bars (warmup), then Some(0.0..100.0)
let value: Option<f64> = rsi.update(close_price);
```

### Indicator examples to implement

| Indicator       | `update()` input   | Output                   | Key state                      |
| --------------- | ------------------ | ------------------------ | ------------------------------ |
| RSI             | close              | 0–100                    | avg_gain, avg_loss             |
| EMA             | close              | smoothed price           | prev_ema                       |
| MACD            | close              | (macd_line, signal_line) | fast_ema, slow_ema, signal_ema |
| Bollinger Bands | close              | (upper, mid, lower)      | rolling window, std_dev        |
| Stochastic      | (high, low, close) | (%K, %D)                 | rolling window                 |
| ATR             | (high, low, close) | volatility value         | prev_atr                       |
| Volume-weighted | (close, volume)    | vwap                     | cumulative vol/price           |

> **Design rule**: Indicators have no `bot_core` dependency. They're pure `f64` → `Option<f64>`. This makes them trivially unit-testable.

---

## Layer 3: Strategy (Phase Machine)

Every indicator strategy uses the same state machine for order lifecycle:

```rust
pub enum Phase {
    WarmingUp,   // Collecting bars for indicator convergence
    Watching,    // Indicator ready, waiting for signal
    Opening,     // Entry order placed, awaiting fill
    InPosition,  // Position held, watching for exit signal
    Closing,     // Exit order placed, awaiting fill
}
```

### Event flow in `on_event`:

```rust
Event::Quote(q) => {
    let mid = (q.bid.0 + q.ask.0) / 2;  // f64

    // Layer 1: Tick → Bar
    if let Some(bar) = self.bar_builder.update(mid, q.ts) {

        // Layer 2: Bar → Signal
        if let Some(value) = self.indicator.update(bar.close) {

            // Phase transitions
            if self.phase == Phase::WarmingUp {
                self.phase = Phase::Watching;
            }

            // Layer 3: Signal → Order
            match self.phase {
                Phase::Watching => {
                    if self.should_enter(value) {
                        self.place_market_crossing_order(ctx, ...);
                        self.phase = Phase::Opening;
                    }
                }
                Phase::InPosition => {
                    if self.should_exit(value) {
                        self.place_market_crossing_order(ctx, ...);
                        self.phase = Phase::Closing;
                    }
                }
                _ => {}
            }
        }
    }
}

Event::OrderFilled(_) => {
    match self.phase {
        Phase::Opening => self.phase = Phase::InPosition,
        Phase::Closing => self.phase = Phase::Watching,
        _ => {}
    }
}

Event::OrderRejected(_) | Event::OrderCanceled(_) => {
    // Revert to previous valid phase
    match self.phase {
        Phase::Opening => self.phase = Phase::Watching,
        Phase::Closing => self.phase = Phase::InPosition,
        _ => {}
    }
}
```

---

## Config Pattern

Indicator strategies use **strategy-only params** in the config JSON. The `market` and `environment` come from the top-level `BotConfig` (no duplication):

```json
{
  "strategy_type": "rsi",
  "environment": "mainnet",
  "markets": [
    { "exchange": "hyperliquid", "type": "perp", "base": "ETH", "index": 0 }
  ],

  "rsi": {
    "strategy_id": "my-rsi-bot",
    "rsi_period": 14,
    "bar_interval_secs": 60,
    "oversold": 30.0,
    "overbought": 70.0,
    "order_size": "0.01",
    "side": "long",
    "leverage": "1"
  }
}
```

The constructor receives market from `build_strategy()`:

```rust
pub fn new(config: MyConfig, market: Market, environment: Environment) -> Self { ... }
```

And in `build_strategy()`:

```rust
} else if strategy_type == "myindicator" {
    let my_config: MyConfig = config.custom_config("myindicator")?;
    let market = config.primary_market().clone();
    let environment = config.parse_environment();
    Ok(Box::new(MyStrategy::new(my_config, market, environment)))
}
```

---

## Indicators as Addons to Existing Strategies

Indicators aren't only for standalone strategies. They can be **composed into any existing strategy** as entry/exit guards. The indicator + bar builder are self-contained modules — drop them into Grid, DCA, or any strategy to gate when the strategy activates.

### Use Cases

| Existing Strategy | Indicator Addon  | Effect                                                                           |
| ----------------- | ---------------- | -------------------------------------------------------------------------------- |
| **Grid**          | RSI guard        | Only build/activate the grid when RSI < 30 (oversold). Stop grid when RSI > 80.  |
| **Grid**          | Bollinger guard  | Start grid when price touches lower band. Close all when upper band is hit.      |
| **DCA**           | MACD crossover   | Only trigger DCA entries when MACD crosses above signal (momentum confirmation). |
| **DCA**           | ATR filter       | Skip DCA entries when ATR is too high (volatile market, wait for calm).          |
| **Any**           | EMA trend filter | Only allow long entries when fast EMA > slow EMA (uptrend).                      |

### Integration Pattern

Add `BarBuilder` + `Indicator` as fields on the existing strategy struct. Feed them in `handle_quote()` / `on_event()`. Use the indicator output to gate the strategy's core logic.

```rust
// Inside an existing strategy like GridStrategy:
pub struct GridStrategy {
    config: GridConfig,
    state: GridState,
    instrument_meta: Option<InstrumentMeta>,

    // ── Indicator addon fields ──
    bar_builder: Option<BarBuilder>,       // None = no indicator guard
    rsi: Option<Rsi>,                      // Or any indicator
    indicator_ready: bool,                 // Has the indicator converged?
}
```

### Gating logic inside `handle_quote`:

```rust
fn handle_quote(&mut self, ctx: &mut dyn StrategyContext, bid: Price, ask: Price) {
    let mid = Price((bid.0 + ask.0) / TWO);

    // ── Feed indicator (if configured) ──
    if let (Some(bar_builder), Some(rsi)) = (&mut self.bar_builder, &mut self.rsi) {
        let mid_f64 = mid.0.to_f64().unwrap_or(0.0);
        if let Some(bar) = bar_builder.update(mid_f64, ctx.now_ms()) {
            if let Some(rsi_value) = rsi.update(bar.close) {
                self.indicator_ready = true;

                // ── Entry guard: don't build grid until indicator says go ──
                if !self.state.is_initialized && rsi_value > self.config.rsi_entry_threshold {
                    ctx.log_debug(&format!("RSI {:.1} > {} — waiting for entry signal",
                        rsi_value, self.config.rsi_entry_threshold));
                    return; // Skip grid initialization
                }

                // ── Exit guard: stop grid if indicator says exit ──
                if self.state.is_initialized && rsi_value > self.config.rsi_exit_threshold {
                    self.stop_strategy(ctx, &format!(
                        "RSI exit signal: {:.1} > {}", rsi_value, self.config.rsi_exit_threshold));
                    return;
                }
            }
        }
    }

    // ... existing grid logic continues here (build_grid, sync_orders, etc.)
}
```

### Config extension

Add optional indicator params to the existing strategy config:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct GridConfig {
    // ... existing grid fields ...

    /// Optional: RSI period for entry/exit guard (0 = disabled)
    #[serde(default)]
    pub rsi_period: u32,

    /// RSI below this → allow grid to start (default: 30)
    #[serde(default = "default_rsi_entry")]
    pub rsi_entry_threshold: f64,

    /// RSI above this → stop grid (default: 80)
    #[serde(default = "default_rsi_exit")]
    pub rsi_exit_threshold: f64,

    /// Bar interval in seconds for indicator calculation
    #[serde(default = "default_bar_interval")]
    pub indicator_bar_secs: u64,
}
```

```json
{
  "strategy_type": "grid",
  "grid": {
    "grid_levels": 20,
    "start_price": "88000",
    "end_price": "92000",

    "rsi_period": 14,
    "rsi_entry_threshold": 30.0,
    "rsi_exit_threshold": 80.0,
    "indicator_bar_secs": 60
  }
}
```

> **Key principle**: The indicator is **optional**. When `rsi_period = 0` (default), the strategy behaves exactly as before — no indicator overhead. When configured, it adds an intelligent entry/exit filter on top of the existing logic.

### Why this works

The indicator module (`Rsi`, `Ema`, etc.) has **zero `bot_core` dependency** — it's just `f64` → `Option<f64>`. So you can drop it into any strategy without coupling. The `BarBuilder` is also self-contained. Together they're ~300 lines of pure Rust that compose cleanly with any existing strategy's `on_event` loop.

---

## Crate Structure (5-file pattern)

Indicator strategies extend the base 3-file pattern with `bar.rs` and `indicator.rs`:

```
strategy-myindicator/
├── Cargo.toml
└── src/
    ├── lib.rs          # pub mod + re-exports
    ├── config.rs       # Strategy params (period, thresholds, size)
    ├── state.rs        # Phase enum + runtime counters
    ├── bar.rs          # BarBuilder (copy from strategy-rsi)
    ├── indicator.rs    # Your indicator math (pure f64)
    └── strategy.rs     # Strategy trait impl (phase machine)
```

---

## Quick-Start Checklist

1. **Copy `bar.rs`** from `strategy-rsi` (reusable as-is)
2. **Write `indicator.rs`** — implement `new(period)` + `update(f64) -> Option<f64>`
3. **Write unit tests** for the indicator against known reference values
4. **Define `config.rs`** — indicator params + thresholds + order size
5. **Implement `strategy.rs`** — wire BarBuilder → Indicator → Phase machine
6. **Wire into engine** — 3 files to touch (see [STRATEGY_API.md](../STRATEGY_API.md#e2e-build--run))
7. **Test**: `cargo test -p strategy-myindicator` → `supurr dev build` → `supurr dev run`

---

## Reference Implementation

The RSI strategy is the reference for this pattern:

| File           | Lines | Source                                 |
| -------------- | ----- | -------------------------------------- |
| `bar.rs`       | 117   | `crates/strategy-rsi/src/bar.rs`       |
| `indicator.rs` | 179   | `crates/strategy-rsi/src/indicator.rs` |
| `config.rs`    | 104   | `crates/strategy-rsi/src/config.rs`    |
| `state.rs`     | 45    | `crates/strategy-rsi/src/state.rs`     |
| `strategy.rs`  | 310   | `crates/strategy-rsi/src/strategy.rs`  |
