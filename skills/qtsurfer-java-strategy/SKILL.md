---
name: qtsurfer-java-strategy
description: Write, review, and debug QTSurfer Java trading strategies. Covers AbstractTickerStrategy, the indicator builder API, window listeners, state management, signal emission, and submission via the MCP server or SDK.
license: Apache-2.0
---

# QTSurfer Java Strategy

A QTSurfer strategy is a plain Java class (no framework annotations required) that extends `AbstractTickerStrategy`. It receives real-time tickers, configures technical indicators, and emits buy/sell signals. The engine compiles strategies server-side — no local toolchain needed.

## Minimal template

```java
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.strategy.AbstractTickerStrategy;

public class MyStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        // configure indicators here — called once per instrument on first tick
    }
}
```

`acceptInstrument` and `getExecutionMode` have sensible defaults (accept all instruments, LONG mode). Override only when needed:

```java
import com.wualabs.qtsurfer.engine.core.Instrument;
import com.wualabs.qtsurfer.engine.strategy.execution.ExecutionMode;

@Override
public boolean acceptInstrument(Instrument instrument) {
    return instrument.base().equals("BTC"); // filter instruments here if needed
}

@Override
public ExecutionMode getExecutionMode(Instrument instrument) {
    return ExecutionMode.LONG; // LONG, SHORT, or LONG_SHORT
}
```

> Note: the **default** `acceptInstrument` is *not* unconditional — it gates on the strategy's
> output currency / `acceptCurrency`. To accept **every** instrument unconditionally, override it
> explicitly with `return true`.

## Indicator setup

All indicators are defined in `setupIndicators` using the fluent builder on `InstrumentGroupRTIndicator`. Methods return `this` for chaining.

```java
@Override
protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
    indicators
        .addPrice()                     // source: close price
        .ema("emaFast", 9)             // 9-period EMA named "emaFast"
        .ema("emaSlow", 21)            // 21-period EMA named "emaSlow"
        .rsi(14)                        // 14-period RSI named "rsi14"
        .bollinger("bb", 20, 2.0)      // Bollinger Bands → "bb", "bbUpper", "bbLower"
        .window("emaFast", WindowTime.s1, new MyListener(this, indicators));
}
```

See [references/indicators.md](references/indicators.md) for the full indicator catalogue.

### WindowTime values

`WindowTime.s1`, `s5`, `s10`, `s30`, `m1`, `m3`, `m5`  
Custom: `Duration.ofSeconds(n)` or `Duration.ofMinutes(n)`

### Reading indicator values outside a listener

```java
import com.wualabs.qtsurfer.engine.core.Instrument;
import com.wualabs.qtsurfer.engine.core.Ticker;

@Override
public void update(Ticker ticker) {
    Instrument instrument = ticker.instrument();
    updateInstrument(instrument, ticker.timestamp());
    var ind = updateIndicators(instrument, ticker);

    if (!ind.getExisting("emaSlow").isReady()) return; // wait for warmup

    double fast = ind.getValue("emaFast");
    double slow = ind.getValue("emaSlow");

    if (fast > slow) emitBuy(ticker.last());
    else             emitSell(ticker.last());
}
```

`Ticker` is an engine record — use accessor methods (`ticker.last()`, `ticker.bid()`, `ticker.ask()`, `ticker.instrument()`, `ticker.timestamp()`) rather than JavaBean getters.

## Window listener pattern (recommended)

Listeners fire once per time window rather than on every tick. Prefer this over `update()` for strategies that react to bar closes.

```java
import com.wualabs.qtsurfer.engine.strategy.AbstractOnChangeListener;
import com.wualabs.qtsurfer.engine.core.state.StateStoreSupport;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;

public class MyStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators
            .addPrice()
            .rsi(14)
            .window("rsi14", WindowTime.m1, new SignalListener(this, indicators));
    }

    private class SignalListener extends AbstractOnChangeListener {

        public SignalListener(AbstractTickerStrategy strategy,
                              InstrumentGroupRTIndicator indicators) {
            super(strategy, indicators);
        }

        @Override
        public void onChange(StateStoreSupport store, double prev, double actual) {
            initStore(store);                  // lazy-init this.store
            long count = this.store.inc("bars");

            if (actual < 30) emitBuy(indicators.getValue("price"));
            if (actual > 70) emitSell(indicators.getValue("price"));
        }
    }
}
```

`AbstractOnChangeListener` gives you:
- `emitBuy(price)` / `emitSell(price)` / `emitSignal(signal)`
- `detectCrossAbove(state, idx, left, right)` / `detectCrossBelow(...)`
- `initStore(storeSupport)` → initialises `this.store` (call at top of `onChange`)
- `this.instrument` — current instrument
- `this.indicators` — indicator group

## State management

`StateStore` is per-instrument, lazily initialised. Access via `initStore(storeSupport)` inside a listener, or `getStateStore(instrument)` inside `update()`.

```java
this.store.inc("count")          // int counter, returns new value
this.store.dec("count")
this.store.set("inPosition")     // boolean flag → true
this.store.unset("inPosition")   // → false
this.store.is("inPosition")      // read boolean
this.store.add("pnl", delta)     // double accumulator, returns new value
this.store.setState("key", obj)  // arbitrary object
this.store.getState("key", def)  // with default
```

## Configurable properties

```java
@StrategyProperty(name = "rsi.period", description = "RSI period", defaultValue = "14")
private int rsiPeriod = 14;

@StrategyProperty(name = "ema.fast", description = "Fast EMA period", defaultValue = "9")
private int fastPeriod = 9;
```

Properties are injected before `setupIndicators` is called.

## Signal emission

| Method | When to use |
|--------|-------------|
| `emitBuy(price)` | Enter long position |
| `emitSell(price)` | Enter short / close long |
| `emitSignal(signal)` | Custom signal (`BuySignal`, `SellSignal`, `InfoStrategySignal`) |

### Data / analytics signals — `InfoStrategySignal`

For non-trading strategies that emit **computed fields** (analytics, metrics) rather than
buy/sell, build an `InfoStrategySignal` and attach arbitrary key/values, then `emitSignal`:

```java
InfoStrategySignal signal = createInfoStrategySignal(instrument);  // from AbstractTickerStrategy
signal.set("interval", "1m");
signal.set("zscore", z);
signal.set("vwap", vwap);
emitSignal(signal);
```

Subscribers read the fields with `signal.get("key")` / `signal.has("key")` and
`signal.getInstrument()`. Prefix a field's name with `_` to keep it out of reporting metadata.

## Crossover detection helper

```java
private Boolean[] crossState = new Boolean[1]; // one slot per crossover

// In onChange or update:
if (detectCrossAbove(crossState, 0, fast, slow)) emitBuy(price);
if (detectCrossBelow(crossState, 0, fast, slow)) emitSell(price);
```

## Building a Ticker in tests

`engine.core.Ticker` is a **14-field record** — construct it directly (there is no builder):

```java
import com.wualabs.qtsurfer.engine.core.Ticker;
import com.wualabs.qtsurfer.engine.core.Instrument;

new Ticker(
    new Instrument("BTC", "USDT"),       // instrument (base, quote[, settle])
    bid, bidSize, ask, askSize,          // BigDecimal (nullable)
    open, high, low, last, vwap,         // BigDecimal (nullable)
    volume, quoteVolume, percentageChange, // BigDecimal (nullable)
    epochMillis);                        // long — epoch MILLISECONDS
```

Field order: `instrument, bid, bidSize, ask, askSize, open, high, low, last, vwap, volume,
quoteVolume, percentageChange, timestamp`. `Instrument` is also a record:
`new Instrument(base, quote)` (spot) or `new Instrument(base, quote, settle)` (derivative).

## Compile & submit via MCP

Download the MCP server from [QTSurfer/mcp-java releases](https://github.com/QTSurfer/mcp-java/releases/latest) (native binary or fat JAR) and configure it in your agent. Once connected:

1. Use `list_exchanges` → `list_instruments` to pick a valid exchange and instrument.
2. Call `submit_backtest` with `strategyCode` = the full Java source of your strategy class.
3. Poll `get_job_status` until `COMPLETED`, then read the results.

The engine compiles the strategy server-side — only the `.java` source is sent.

## Current scope and roadmap

**`AbstractTickerStrategy`** — processes real-time tickers (bid/ask/last, volume). This is the current primary strategy type.

**`AbstractMultiStrategy`** _(coming soon)_ — will support multiple data sources in a single strategy: Tickers + KLines + FundingRates. Do not attempt to implement multi-source strategies with the current API.

## Cross-instrument (market-wide) strategies

A strategy instance sees **every** accepted instrument, each with its own indicator group. To
compute something *across* instruments (a market-wide percentile, a relative-strength rank, a
basket signal), override `update(Ticker)`, call `super.update(ticker)` first so the engine
advances the firing instrument's indicators, then read any instrument's indicators:

```java
@Override
public void update(Ticker ticker) {
    super.update(ticker);                       // engine updates THIS instrument's group
    Instrument ins = ticker.instrument();

    double z = getRTIndicator(ins, "closeZScore")     // this instrument's own indicator
        .map(RTIndicator::getValue).orElse(Double.NaN);

    List<Double> prices = new ArrayList<>();          // read across all tracked instruments
    for (Instrument other : getInstruments()) {
        getRTIndicator(other, "price")
            .filter(RTIndicator::isReady)
            .ifPresent(ind -> prices.add(ind.getValue()));
    }
    // ... compute a market-wide stat from `prices`, then emitSignal(...)
}
```

Helpers on `AbstractTickerStrategy`:
- `getInstruments()` — the set of instruments the strategy is tracking.
- `getRTIndicator(instrument, name)` → `Optional<RTIndicator>` — any instrument's named indicator (read-only, no re-update).
- Always call `super.update(ticker)` **first** — skip it and the indicators never advance.

## Common mistakes

- **Forgetting `isReady()` check** — indicators need warmup periods. Always check before reading values.
- **Mutating indicators in `update()`** — use `getReadOnlyExisting()` instead of `getExisting()` to prevent accidental state changes.
- **One `setupIndicators` per strategy class** — it is called once per instrument, not per tick.
- **Inner class vs lambda for listeners** — `AbstractOnChangeListener` gives access to helpers; prefer inner class over raw lambda.
- **Hidden indicators** — prefix with `_` (e.g. `_gain`) to exclude from signal reporting metadata.
- **Using JavaBean getters on Ticker** — `Ticker` is a record; use `ticker.last()` not `ticker.getLast()`, `ticker.instrument()` not `ticker.getInstrument()`, `ticker.timestamp()` not `ticker.getTimestamp().getTime()`.

