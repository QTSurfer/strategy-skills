# Advanced Strategy Patterns

Proven patterns extracted from production and backtested legacy strategies.

## Noise filtering chain

The most sophisticated signal processing pattern in the codebase. Converts a raw indicator into a clean, smoothed signal:

```
raw signal
  → clamp(±threshold → 0)        suppress micro-noise
    → percentChange               convert to rate of change
      → conditional(≠0, EMA(n))  only feed non-zero changes to EMA
```

```java
indicators
    .add("raw", TickerValueSource.Close)
    .distance("distemas", "ema60", "ema500")
    .clamp("distemas", v -> Math.abs(v) <= 0.1, 0.0)
    .percentChange("chgDistemas")
    .conditional("smoothDistemas", "chgDistemas", v -> v != 0,
        indicators.getReadOnlyExisting("ema500"), zeroIndicator)
    .window("smoothDistemas", WindowTime.s1, new DetectorListener(this, indicators));
```

> Names carry no `%` — `distance()`/`percentChange()` already tag the indicator's metadata with
> `DisplayHint.PERCENT` (goal indicator-integrity), so a percent-y name doesn't need to say so.

**Why:** Raw EMA-distance signals have too much noise for reliable decisions. The conditional prevents zero-padding from diluting the EMA when there's no meaningful change.

---

## EMA distance analysis

Use the **distance between EMAs** as the primary signal instead of raw EMA values. Distance is a derivative — it measures momentum of the trend, not the trend itself.

```java
indicators
    .ema("ema60", 60)
    .ema("ema500", 500)
    .ema("ema2500", 2500)
    .distance("shortDistemas", "ema60", "ema500")   // short-term momentum
    .distance("longDistemas",  "ema500", "ema2500") // medium-term trend
    .gain("longStreakCount", "ema7500");              // macro uptrend
```

**Entry gate:** `longDistemas >= 0.35` AND `shortDistemas` rising AND macro streak >= 300.

---

## Trailing exit variants

### Midpoint trailing (production proven, ~7% avg)

```java
// In onChange:
double percentGain = (price - buyPrice) / buyPrice * 100;
if (percentGain > maxPcnGain) maxPcnGain = percentGain;
double trigger = maxPcnGain - (maxPcnGain - minPercentGain) / 2;
if (percentGain > minPercentGain && percentGain <= trigger) {
    emitSell(price);
}
```

Sells when gain retraces to the midpoint between the minimum threshold and the peak gain. Self-calibrates to the magnitude of the move.

### Peak trailing

```java
// In onChange:
if (price > maxPrice) maxPrice = price;
double fallFromPeak = (maxPrice - price) / maxPrice * 100;
if (fallFromPeak >= 1.0) emitSell(price); // 1% drop from peak
```

Simpler. Good for faster-moving scalps.

### EMA gain reset

```java
// In onChange:
if (exitEmaGain.getPeriodCount() == 0) emitSell(price);
```

Sell when the exit EMA stops rising (momentum exhausted). Requires a `GainRTIndicator` wrapping the exit EMA.

---

## Conditional stop-loss

A macro-aware stop-loss that avoids stopping out during healthy dips:

```java
// In onChange:
double percentGain = (price - buyPrice) / buyPrice * 100;
if (percentGain < 0
        && Math.abs(percentGain) >= 0.5           // loss > 0.5%
        && longDistemas < 0.01) {                  // macro trend weakening
    emitSell(price);
    store.set("fail");
}
```

Only triggers when the macro context is also weak. Reduces false stops in volatile but ultimately trending markets.

---

## Re-entry protection ("fail" state)

Block new entries after a losing trade until the macro context fully resets:

```java
// Entry listener:
if (store.is("fail")) return;  // block until macro reset
// ...entry conditions...

// Separate macro reset check (in another window or update):
if (store.is("fail") && longEmaValue < vlongEmaValue) {
    store.unset("fail");
}
```

Prevents revenge trading after a stop-loss. The reset condition (`longEma < vlongEma`) ensures a full macro cycle completes before re-entry is allowed.

---

## OPS-based instrument activity

Track operations/second per instrument to identify dormant coins waking up:

```java
// In update():
long now = System.currentTimeMillis();
long currentOps = opsCounter.incrementAndGet();
if (now - lastOpsWindow > 1000) {
    double ops = (double) currentOps / ((now - lastOpsWindow) / 1000.0);
    lastOps = ops;
    lastOpsWindow = now;
    opsCounter.set(0);
}
// Low ops + sudden spike = pump candidate
```

**Pattern:** Sort all instruments by ops ascending. Coins with < 0.1 ops/sec that suddenly spike are pump candidates.

---

## Multi-stage entry filter 

All production-tested strategies use 2–3 independent confirmation stages before entry:

| Strategy | Stage 1 | Stage 2 | Stage 3 |
|----------|---------|---------|---------|
| S1 | BB bandwidth in [0.5, 0.6] | Price above EMA500 | — |
| S2 | EMA gain streak >= 3 | Price above EMA200 | Volatility >= 50% |
| S3 | VlongEMA streak >= 300 | Distance rising for 10+ ticks | Distance >= 0.35% |

**Rule of thumb:** At least one momentum condition + one macro trend condition + one noise/false-positive guard.

---

## Shared StateStore between windows

When two windows on the same instrument need to coordinate (e.g., entry window sets a flag, exit window reads it):

```java
// In setupIndicators, share the instrument's strategy-level store
StateStore sharedStore = getStateStore(instrument).orElse(null);

indicators
    .globalStateStore()  // or pass the shared store explicitly
    .window("ema10", WindowTime.s1, new EntryListener(this, indicators))
    .window("price", Duration.ofMillis(100), new ExitListener(this, indicators));

// Both listeners call initStore(storeSupport) — they see the same store
```

---

## Volatility gate

Prevent entries during low-activity periods:

```java
indicators
    .addPrice()
    .add("vlts", new VolatilityRTIndicator(smaPeriods).clampUpdates(warmupPeriods))
    // ...other indicators...

// In entry listener:
double volatility = indicators.getValue("vlts");
if (volatility < 50.0) return; // market not active enough
```

`clampUpdates(n)` suppresses the first N updates (returns 0) to allow the underlying SMA to warm up before volatility values are meaningful.

---

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
