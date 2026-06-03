# Indicator Catalogue

All methods below are on `InstrumentGroupRTIndicator` and return `this` for chaining.
Default indicator name (when `name` is omitted) is the method name + parameters, e.g. `rsi14`.

## Price sources

```java
.addPrice()                            // close price → "price"
.add("bid",  TickerValueSource.Bid)
.add("ask",  TickerValueSource.Ask)
.add("vol",  TickerValueSource.Volume)
// TickerValueSource: Bid, BidSize, Ask, AskSize, Open, High, Low, Close,
//                   Vwap, Volume, VolumeQuote, PercentChange, AutoAskClose
```

## Moving averages

```java
.sma(20)                               // 20-period SMA → "sma20"
.sma("s20", 20)                        // custom name
.sma("s20", 20, false)                 // continuous mode (default: discrete)
.sma("s20", "rsi14", 20)              // SMA of another indicator
.ema(9)                                // 9-period EMA → "ema9"
.ema("fast", 9)
.ema("fast", "vol", 9)                // EMA of volume
```

## Oscillators & momentum

```java
.rsi(14)                               // Cutler's RSI → "rsi14"
.rsi(14, false)                        // Wilder's smoothing
.rsi("myRsi", 14, true)

.bollinger("bb", 20, 2.0)             // → "bb", "bbUpper", "bbLower"
.bollingerBandwidth("bb")             // % width of a Bollinger band
```

## Rate of change & distance

```java
.percentChange("price")                // % change tick-over-tick
.rateChange("price")                   // absolute rate of change
.rateChange("rc", "price", true)      // percent=true
.distanceMa("ema9")                   // % distance from MA
.distance("gap", "ema9", "ema21")     // % distance between two indicators
```

## Gain / loss / extremes

```java
.gain("price")                         // consecutive gain periods
.loss("price")                         // consecutive loss periods
.gain("g", "price", false)            // resetPeriodsOnSustain=false
.max("price")                          // running max
.min("price")                          // running min
.sum("vol")                            // running sum
```

## Arithmetic

```java
.add("spread", "ask", "bid")          // spread = ask + bid
.diff("spread", "ask", "bid")         // diff = ask - bid
.mul("price", 0.01)                   // scale by coefficient
.mul("ratio", "vol", "price")         // vol * price
.fun("custom", "a", "b", (a, b) -> a / b)  // arbitrary BiFunction
```

## Predicates & conditionals

```java
.lessThan("oversold", "rsi14", 30)    // boolean: rsi14 < 30
.greatThan("overbought", "rsi14", 70)
.greatOrEqual("ge", "price", 50000)
.lessOrEqual("le", "price", 50000)
.equal("eq", "price", 100)
.notEqual("ne", "price", 100)
.predicate("custom", "price", v -> v > 0 && v < 100)
.periodCount("cnt", "oversold", v -> v > 0)  // count consecutive true periods
```

## Conditional selection

```java
// If indicator == coef → thenIndicator else elseIndicator
.equal("selected", "signal", 1, "emaFast", "emaSlow")
.conditional("out", "flag", ind -> ind.getValue() > 0, thenInd, elseInd)
```

## Transformations

```java
.clamp("price", 0.0, 100.0)          // clamp to [min, max]
.clamp("price", v -> v < 0, 0.0)     // clamp when predicate true
.round("price", 2)                    // round to N decimals
.decorate("price", "price", ind -> new MyWrapper(ind))
```

## Window listeners

```java
.window("ema9", WindowTime.s1, listener)     // fire every 1 s
.window("ema9", Duration.ofSeconds(15), l)   // custom duration
.window()                                     // builder pattern
    .windowTime(WindowTime.m5)
    .indicator("rsi14")
    .listener(myListener)
    .build()
```

## Composing indicators (read-only access)

When building a custom indicator that references another, use a read-only view to avoid mutating shared state. Two equivalent approaches:

```java
// Option A — .ro() on any RTIndicator instance (default method on RTIndicator)
RTIndicator src = indicators.getExisting("ema9").ro();
indicators.add("custom", new MyIndicator(src));

// Option B — getReadOnlyExisting() on the indicator group
RTIndicator src = indicators.getReadOnlyExisting("ema9");
indicators.add("custom", new MyIndicator(src));

// Option C — getReadOnly() returns Optional (safe if indicator may not exist)
indicators.getReadOnly("ema9").ifPresent(src ->
    indicators.add("custom", new MyIndicator(src)));
```

`.ro()` is a default method on `RTIndicator` itself — available on every indicator instance without going through the group. Use it when you already hold a reference to the indicator object.

## Advanced indicator catalogue (statistics & pro)

Beyond the fluent builder methods above, the engine ships ~180 indicator classes under
`com.wualabs.qtsurfer.engine.indicators.{averages, bollinger, statistics, statistics.pro, pro}`.
They are plain `RTIndicator` instances — add any of them by class with
`.add("name", new XxxRTIndicator(...))`, then read with `indicators.getValue("name")`:

```java
import com.wualabs.qtsurfer.engine.indicators.statistics.StandardDeviationRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.statistics.pro.ZScoreRTIndicator;

indicators
    .addPrice()                                                      // "price"
    .sma("mean", 20)
    .add("std",    new StandardDeviationRTIndicator(20))             // ctor (int periods)
    .add("stdOf",  new StandardDeviationRTIndicator(                 // ctor (RTIndicator, int periods)
            indicators.getReadOnlyExisting("mean"), 20))
    .add("zscore", new ZScoreRTIndicator(/* see class for ctor */));
```

Constructors vary per class — most take `(int periods)` and/or `(RTIndicator source, int periods)`;
some (lombok-built) differ, so check the class. Useful classes by category:

| Category | `*RTIndicator` classes |
|---|---|
| Moving averages | `Sma`, `Ema`, `Wma`, `Hma`, `Kama`, `Tema`, `Mma` |
| Statistics | `StandardDeviation`, `Variance`, `ZScore`, `Skewness`, `Kurtosis`, `RollingPercentile`, `Correlation`, `Covariance`, `Beta` |
| Regression / trend | `LinearRegressionSlope`, `SimpleLinearRegression`, `Adx`, `Aroon`, `SuperTrend`, `ParabolicSar`, `Ichimoku` |
| Volatility | `Atr`, `Natr`, `PercentVolatility`, `RealizedVolatility`, `Parkinson`, `GarmanKlass`, `EwmaVolatility` |
| Volume | `Vwap`, `Obv`, `Mfi`, `Cmf`, `Adl`, `ElderForceIndex` |
| Oscillators | `Macd`, `StochasticOscillator`, `StochasticRsi`, `Cci`, `Roc`, `Momentum`, `WilliamsR`, `UltimateOscillator` |
| Ratios (performance) | `SharpeRatio`, `SortinoRatio`, `CalmarRatio`, `MaxDrawdown`, `OmegaRatio`, `UlcerIndex` |

Compose them by feeding one indicator's read-only view into another's `(RTIndicator, …)` constructor
(e.g. a `ZScore` of an `Sma`). This is how to do rolling stats / aggregation **without
reinventing the wheel** in `update()`.

## Writing a custom RTIndicator

When no built-in fits, implement the `RTIndicator` interface
(`com.wualabs.qtsurfer.engine.indicators.core.RTIndicator`) — or extend `AbstractRTIndicator`
for the common scaffolding:

```java
import com.wualabs.qtsurfer.engine.indicators.core.RTIndicator;

public class MyIndicator implements RTIndicator {
    private double value;
    private boolean ready;

    @Override public double getValue() { return value; }

    @Override public double update(double newValue) {     // called once per tick with the source value
        this.value = /* compute incrementally from newValue */ newValue;
        this.ready = true;
        return value;
    }

    @Override public boolean isReady() { return ready; }  // gate warmup (default true)

    @Override public void reset() { value = 0; ready = false; }  // from Resettable
}
```

Register it like any built-in: `indicators.add("myInd", new MyIndicator())`. The interface is
small: `getValue()` (current output), `update(double)` (incremental, per tick), `isReady()`
(warmup gate, default `true`), `reset()`. `update(Number)` / `update(RTIndicator)` and `ro()`
(read-only view) come as default methods for free.

## Hidden indicators

Prefix with `_` to exclude from signal reporting metadata:

```java
.gain("_rawGain", "price")   // internal use, not reported
```
