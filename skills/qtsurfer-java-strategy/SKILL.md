---
name: qtsurfer-java-strategy
description: >-
  Write, review, and debug QTSurfer Java trading strategies using
  AbstractTickerStrategy, its indicator-builder API, window listeners,
  state management, and signal emission. The trigger branches are:
  - write a QTSurfer strategy (template, indicators, listeners)
  - review or debug a strategy (common mistakes, cross-instrument patterns)
  - configure properties or submit via MCP
license: Apache-2.0
metadata:
  version: 1.0.2
---

# QTSurfer Java Strategy

A QTSurfer strategy is a plain Java class extending `AbstractTickerStrategy`. It receives
real-time tickers, configures technical indicators via a **fluent builder**, and emits
buy/sell signals. The engine compiles strategies server-side — no local toolchain needed.

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

Override `acceptInstrument` and `getExecutionMode` only when the defaults don't fit:

```java
@Override
public boolean acceptInstrument(Instrument instrument) {
    return instrument.base().equals("BTC");
}

@Override
public ExecutionMode getExecutionMode(Instrument instrument) {
    return ExecutionMode.LONG; // LONG, SHORT, or LONG_SHORT
}
```

> **Gotcha:** the default `acceptInstrument` gates on the strategy's output currency. To accept
> **every** instrument unconditionally, override it with `return true`.

## Indicator setup

All indicators are defined in `setupIndicators` using the fluent builder.
Methods return `this` for chaining. See [references/indicators.md](references/indicators.md)
for the full catalogue.

```java
@Override
protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
    indicators
        .addPrice()
        .ema("emaFast", 9)
        .ema("emaSlow", 21)
        .rsi(14)
        .bollinger("bb", 20, 2.0)
        .window("emaFast", WindowTime.s1, new MyListener(this, indicators));
}
```

**WindowTime values:** `s1`, `s5`, `s10`, `s30`, `m1`, `m3`, `m5`
or `Duration.ofSeconds(n)` / `Duration.ofMinutes(n)`.

### Reading indicators outside a listener

```java
@Override
public void update(Ticker ticker) {
    updateInstrument(ticker.instrument(), ticker.timestamp());
    var ind = updateIndicators(ticker.instrument(), ticker);

    if (!ind.getExisting("emaSlow").isReady()) return;

    double fast = ind.getValue("emaFast");
    double slow = ind.getValue("emaSlow");
    if (fast > slow) emitBuy(ticker.last());
    else             emitSell(ticker.last());
}
```

> `Ticker` is a record — use accessor methods (`ticker.last()`, `ticker.bid()`,
> `ticker.instrument()`) rather than JavaBean getters.

## Window listener pattern (recommended)

Listeners fire once per time window rather than on every tick. Prefer over raw `update()`.

```java
private class SignalListener extends AbstractOnChangeListener {
    public SignalListener(AbstractTickerStrategy strategy,
                          InstrumentGroupRTIndicator indicators) {
        super(strategy, indicators);
    }

    @Override
    public void onChange(StateStoreSupport store, double prev, double actual) {
        initStore(store);
        if (actual < 30) emitBuy(indicators.getValue("price"));
        if (actual > 70) emitSell(indicators.getValue("price"));
    }
}
```

Helpers on `AbstractOnChangeListener`:
- `emitBuy(price)` / `emitSell(price)` / `emitSignal(signal)`
- `detectCrossAbove(state, idx, left, right)` / `detectCrossBelow(...)`
- `initStore(storeSupport)` — call at top of `onChange` to initialise `this.store`

## State management

`StateStore` is per-instrument, lazily initialised.

```java
store.inc("count")            // int counter, returns new value
store.dec("count")
store.set("inPosition")       // boolean flag → true
store.unset("inPosition")     // → false
store.is("inPosition")        // read boolean
store.add("pnl", delta)       // double accumulator
store.setState("key", obj)    // arbitrary object
store.getState("key", def)    // with default
```

## Configurable properties

```java
@StrategyProperty(name = "rsi.period", description = "RSI period", defaultValue = "14")
private int rsiPeriod = 14;
```

Properties are injected before `setupIndicators` is called.

## Signal emission

| Method | Use |
|--------|-----|
| `emitBuy(price)` | Enter long |
| `emitSell(price)` | Enter short / close long |
| `emitSignal(signal)` | Custom signal |

For non-trading strategies (analytics, metrics), build an `InfoStrategySignal`:

```java
InfoStrategySignal signal = createInfoStrategySignal(instrument);
signal.set("zscore", z);
emitSignal(signal);
```

Subscribers read fields with `signal.get("key")`. Prefix a field with `_`
to exclude it from reporting metadata.

## Crossover detection

```java
private Boolean[] crossState = new Boolean[1];

if (detectCrossAbove(crossState, 0, fast, slow)) emitBuy(price);
if (detectCrossBelow(crossState, 0, fast, slow)) emitSell(price);
```

## Building a Ticker in tests

`Ticker` is a 14-field record (no builder):

```java
new Ticker(
    new Instrument("BTC", "USDT"),
    bid, bidSize, ask, askSize,
    open, high, low, last, vwap,
    volume, quoteVolume, percentageChange,
    epochMillis);
```

`Instrument` is also a record: `new Instrument(base, quote)` or
`new Instrument(base, quote, settle)` for derivatives.

## Compile & submit via MCP

Download the **MCP server** from [QTSurfer/mcp-java releases](https://github.com/QTSurfer/mcp-java/releases/latest)
(native binary or fat JAR) and configure it in your agent.

1. `list_exchanges` → `list_instruments` to pick a valid pair.
2. `submit_backtest` with `strategyCode` = the full Java source.
3. Poll `get_job_status` until `COMPLETED`.

The engine compiles server-side — only the `.java` source is sent.

## Cross-instrument strategies

To compute across instruments (market-wide percentile, relative-strength rank), override
`update(Ticker)`, call `super.update(ticker)` first, then read any instrument's indicators:

```java
@Override
public void update(Ticker ticker) {
    super.update(ticker);
    for (Instrument other : getInstruments()) {
        getRTIndicator(other, "price")
            .filter(RTIndicator::isReady)
            .ifPresent(ind -> prices.add(ind.getValue()));
    }
}
```

Helpers: `getInstruments()` (tracked set), `getRTIndicator(instrument, name)`
(any instrument's indicator, read-only). Always call `super.update(ticker)` first.

## Common mistakes

- **isReady check** — indicators need warmup. Always check before reading.
- **Mutating in update()** — use `getReadOnlyExisting()` to prevent accidental state changes.
- **One `setupIndicators` per class** — called once per instrument, not per tick.
- **Inner class over lambda** — listeners need helpers from `AbstractOnChangeListener`.
- **Hidden indicators** — prefix with `_` (e.g. `_gain`) to exclude from reporting.
- **Ticker is a record** — `ticker.last()`, not `ticker.getLast()`.

## Reference files

| File | Content |
|------|---------|
| [references/indicators.md](references/indicators.md) | Full indicator catalogue (fluent builder + advanced classes) |
| [references/examples.md](references/examples.md) | Complete strategy examples (EMA crossover, RSI, Bollinger, configurable) |
| [references/patterns.md](references/patterns.md) | Production patterns (noise filtering, trailing exits, re-entry protection) |