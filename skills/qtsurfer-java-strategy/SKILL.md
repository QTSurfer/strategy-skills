---
name: qtsurfer-java-strategy
description: Write, review, and debug QTSurfer Java trading strategies — classes extending a strategy base class (AbstractTickerStrategy, AbstractKlineStrategy, or AbstractFundingRateStrategy). Use when configuring indicators, window listeners, or per-instrument state, or submitting a strategy to backtest via the MCP server.
license: Apache-2.0
metadata:
  version: 1.3.0
---

# QTSurfer Java Strategy

A QTSurfer strategy is a plain Java class (no framework annotations required) that extends a strategy base class — most commonly `AbstractTickerStrategy` (see [Strategy base classes](#strategy-base-classes) for the kline, funding-rate, and multi-source siblings). It receives real-time market data, configures technical indicators, and emits buy/sell signals. The engine compiles strategies server-side — no local toolchain needed.

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
import com.wualabs.qtsurfer.engine.core.instrument.Instrument;
import com.wualabs.qtsurfer.engine.strategy.execution.ExecutionMode;

@Override
public boolean acceptInstrument(Instrument instrument) {
    return instrument.base().equals("BTC"); // filter instruments here if needed
}

@Override
public ExecutionMode getExecutionMode(Instrument instrument) {
    return ExecutionMode.LONG; // LONG, SHORT, or LONG_MULTI
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
import com.wualabs.qtsurfer.engine.core.instrument.Instrument;
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

`Ticker` is an engine record — read fields with accessor methods: `ticker.last()`, `ticker.bid()`, `ticker.ask()`, `ticker.instrument()`, `ticker.timestamp()`.

## Window listener pattern (recommended)

Listeners fire once per time window rather than on every tick. Prefer this over `update()` for strategies that react to bar closes.

```java
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;
import com.wualabs.qtsurfer.engine.core.state.StateStore;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;

public class MyStrategy extends AbstractTickerStrategy {

    @Override
    protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
        indicators
            .addPrice()
            .rsi(14)
            .window("rsi14", WindowTime.m1, new SignalListener(this, indicators));
    }

    private class SignalListener extends AbstractWindowListener {

        public SignalListener(AbstractTickerStrategy strategy,
                              InstrumentGroupRTIndicator indicators) {
            super(strategy, indicators);
        }

        @Override
        public void onChange(StateStore store, double prev, double actual) {
            long count = store.inc("bars");

            if (actual < 30) emitBuy(indicators.getValue("price"));
            if (actual > 70) emitSell(indicators.getValue("price"));
        }
    }
}
```

`AbstractWindowListener` gives you:
- `emitBuy(price)` / `emitSell(price)` / `emitSignal(signal)`
- `getPrevInstant()` / `getCurrInstant()` — when the window that just fired opened / closed
- `getEngineVersion()` / `getEngineVersionMajor()` / `getEngineVersionMinor()` — the running engine version (see [Engine version](#engine-version))
- `this.instrument` — current instrument
- `this.indicators` — indicator group

Crossover detection is a standalone helper, not a method on the listener — see
[Crossover detection helper](#crossover-detection-helper) below.

`store` arrives as `onChange`'s first parameter — already resolved, nothing to initialise. It's
the same store every listener on this instrument shares (see [State management](#state-management)
below); `getPrevInstant()`/`getCurrInstant()` only resolve when the listener is registered on a
window (via `.window(...)`, as above) — calling them on a listener attached to a plain indicator
throws.

## State management

`StateStore` is per-instrument and shared by every listener on that instrument's indicator group
(one store, not one per window). It's created lazily — a window nobody listens to never touches
it. Inside a window listener it arrives as `onChange`'s first parameter; outside a listener (e.g.
in `update()`) reach it via `getStateStore(instrument)`, which returns `Optional<StateStore>`:

```java
@Override
public void update(Ticker ticker) {
    Instrument instrument = ticker.instrument();
    updateInstrument(instrument, ticker.timestamp());
    var ind = updateIndicators(instrument, ticker);

    StateStore store = getStateStore(instrument).orElseThrow();
    long ticks = store.inc("ticks");
    // ...
}
```

`getStateStore(instrument)` is inherited from the strategy base class — always present (never
`Optional.empty()`) for the documented base classes, so `.orElseThrow()` is safe; it's `Optional`
because the underlying `Strategy` contract allows an implementation to not support per-instrument
state at all. Calling it, unlike a window's own store access, resolves the store immediately —
it doesn't wait for a listener.

```java
store.inc("count")          // int counter, returns new value
store.dec("count")
store.set("inPosition")     // boolean flag → true
store.unset("inPosition")   // → false
store.is("inPosition")      // read boolean
store.add("pnl", delta)     // double accumulator, returns new value
store.setState("key", obj)  // arbitrary object
store.getState("key", def)  // with default
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
import com.wualabs.qtsurfer.engine.strategy.CrossDetector;

private final CrossDetector fastSlowCross = new CrossDetector(); // one instance per pair watched

// In onChange or update:
CrossDetector.Cross cross = fastSlowCross.check(fast, slow);
if (cross.above()) emitBuy(price);
if (cross.below()) emitSell(price);
```

One `check(left, right)` call reports both directions together, so they always describe the same tick.

## Compile & submit via MCP

Download the MCP server from [QTSurfer/mcp-java releases](https://github.com/QTSurfer/mcp-java/releases/latest) (native binary or fat JAR) and configure it in your agent. Once connected:

1. Use `list_exchanges` → `list_instruments` to pick a valid exchange and instrument.
2. Call `submit_backtest` with `strategyCode` = the full Java source of your strategy class.
3. Poll `get_job_status` until `COMPLETED`, then read the results.

The engine compiles the strategy server-side — only the `.java` source is sent.

## Strategy base classes

Every strategy extends one engine base class, chosen by the data source it consumes. The three
single-source bases all extend `AbstractSubscriptionStrategy<T>` and share the **same model** this
skill documents (indicator builder, window listeners, `StateStore`, signal emission) — only the
`update(...)` payload differs. The examples here use `Ticker`, the most common source. Types live
in `com.wualabs.qtsurfer.engine.core`.

| Base class | Source | Handler | Via `submit_backtest` |
|---|---|---|---|
| `AbstractTickerStrategy` | `Ticker` (record) | `update(Ticker)` | ✅ primary, fully documented |
| `AbstractKlineStrategy` | `Kline` (class) | `update(Kline)` | ✅ |
| `AbstractFundingRateStrategy` | `FundingRate` (record) | `update(FundingRate)` | ✅ |
| `AbstractMultiSourceStrategy` | Ticker + Kline + FundingRate | `onTicker` / `onKline` / `onFundingRate` | ⚠️ engine-only — not yet public |

- **`AbstractKlineStrategy`** subscribes to candles for `getInterval()` (a `KlineInterval`). **OHLCV only** — order-book sizes, vwap, and percentage-change fields are absent on this path. `Kline` is a plain class, so use getters (`kline.getInstrument()`, `kline.getCloseTime()`), unlike the `Ticker` record.
- **`AbstractFundingRateStrategy`** receives `update(FundingRate)` on each funding-rate update.
- **`AbstractMultiSourceStrategy`** declares `getRequiredSources()` → `Set<MarketDataSource>` (`Ticker`, `KLine`, `FundingRate`) and dispatches each to `onTicker` / `onKline` / `onFundingRate`; when `KLine` is required, `getKlineInterval()` must be non-null. It compiles and registers in the engine but **is not yet runnable via the public `submit_backtest`** — don't ship multi-source strategies for backtesting until it is exposed.

## Cross-instrument (market-wide) strategies

A strategy instance sees **every** accepted instrument, each with its own indicator group. To
compute something *across* instruments (a market-wide percentile, a relative-strength rank, a
basket signal), override `update(Ticker)` and read other instruments' indicators via
`getInstruments()` / `getRTIndicator(...)`. See
[references/patterns.md](references/patterns.md) → "Cross-instrument (market-wide) strategies".

## Engine version

Three accessors are available with no import, both on the strategy and inside a window listener:

```java
@Override
public void update(Ticker ticker) {
    log.info("running on engine {}", getEngineVersion());  // e.g. "1.0.81"

    if (getEngineVersionMajor() >= 1) { /* ... */ }        // also getEngineVersionMinor()
}
```

The value is read from the loaded engine jar's own metadata, so it reports the engine actually
running rather than one baked in when the strategy was compiled. Nothing here throws — when the
version cannot be determined `getEngineVersion()` returns `EngineVersion.UNKNOWN` (`"unknown"`) and
the numeric accessors return `EngineVersion.UNKNOWN_COMPONENT` (`-1`), so they are safe to call
unguarded and a version gate fails closed instead of matching by accident. Major and minor resolve
together: check one and you can trust the other.

For the patch component, import the engine class — it is deliberately not mirrored onto the sugar:

```java
import com.wualabs.qtsurfer.engine.EngineVersion;

int patch = EngineVersion.getPatch();  // 81
```

Worth emitting (in an `InfoStrategySignal`, or logged on first tick) for strategies that are stored
and re-run later: engine APIs do change between versions, and a stored strategy that suddenly
misbehaves is far easier to diagnose when the engine it ran on is recorded alongside the result.

## Common mistakes

- **Forgetting `isReady()` check** — indicators need warmup periods. Always check before reading values.
- **Mutating indicators in `update()`** — use `getReadOnlyExisting()` instead of `getExisting()` to prevent accidental state changes.
- **One `setupIndicators` per strategy class** — it is called once per instrument, not per tick.
- **Inner class vs lambda for listeners** — `AbstractWindowListener` gives access to helpers; prefer inner class over raw lambda.
- **Using JavaBean getters on Ticker** — `Ticker` is a record; use `ticker.last()` not `ticker.getLast()`, `ticker.instrument()` not `ticker.getInstrument()`, `ticker.timestamp()` not `ticker.getTimestamp().getTime()`.

