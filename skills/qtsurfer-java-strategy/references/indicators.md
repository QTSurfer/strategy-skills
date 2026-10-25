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

## The pro indicators

The pro indicators come with the Pro and Elite plans. In Java they are the methods of
`Pro`, in `com.wualabs.qtsurfer.engine.indicators.pro`: `Pro.on(indicators)` takes the indicator group that `setupIndicators`
receives, and each call registers one pro indicator with the same names, forms and ids as the
table under "The pro indicators: `uses`" in the `qtsurfer-qtscript-strategy` skill lists (`atr`, `adx`, `cmf`,
`garmanKlass`, `ichimoku`, `kurtosis`, `mfi`, `obv`, `parkinson`, `rogersSatchell`, `superTrend`, `vwap`, `yangZhang`). `Pro.on(indicators).adx(14)` registers `adx14` and, beside it,
`adx14PlusDI` and `adx14MinusDI`. A window goes on the main one (`adx14`), and the listener reads the others by name.

They read whole bars, so use them in a kline strategy. This one buys when the ADX says there is a trend and the plus line is
above the minus line, and sells when it is below:

```java
import com.wualabs.qtsurfer.engine.core.KlineInterval;
import com.wualabs.qtsurfer.engine.core.state.StateStore;
import com.wualabs.qtsurfer.engine.indicators.core.KlineValueSource;
import com.wualabs.qtsurfer.engine.indicators.helpers.WindowTimeRTIndicator.WindowTime;
import com.wualabs.qtsurfer.engine.indicators.helpers.group.InstrumentGroupRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.pro.Pro;
import com.wualabs.qtsurfer.engine.strategy.AbstractKlineStrategy;
import com.wualabs.qtsurfer.engine.strategy.AbstractWindowListener;

public class AdxTrend extends AbstractKlineStrategy {

  @Override
  public KlineInterval getInterval() {
    return KlineInterval.s1;
  }

  @Override
  protected void setupIndicators(InstrumentGroupRTIndicator indicators) {
    indicators.addPrice();
    indicators.add("open", KlineValueSource.Open);
    indicators.add("high", KlineValueSource.High);
    indicators.add("low", KlineValueSource.Low);
    indicators.add("close", KlineValueSource.Close);
    indicators.add("volume", KlineValueSource.Volume);
    Pro.on(indicators).adx(14);
    indicators.window("adx14", WindowTime.m1, new TrendListener(this, indicators));
  }

  private class TrendListener extends AbstractWindowListener {

    TrendListener(AbstractKlineStrategy strategy, InstrumentGroupRTIndicator indicators) {
      super(strategy, indicators);
    }

    @Override
    public void onChange(StateStore store, double prev, double actual) {
      double close = indicators.getValue("close");
      double plus = indicators.getValue("adx14PlusDI");
      double minus = indicators.getValue("adx14MinusDI");
      if (actual > 20 && plus > minus) emitBuy(close);
      if (actual > 20 && plus < minus) emitSell(close);
    }
  }
}
```

On a `ticker` strategy a pro indicator is not refused, but its values are not the bar's and mean nothing, and validating or
running the strategy returns an engine notice that says so.

Registering a strategy that uses them is accepted on every plan, and the answer of `POST /strategy` carries
`requires: ["indicators.pro"]`. Running it on a plan that does not include them is refused with `402`, when validating it,
starting a backtest or a sweep with it and starting a live run with it. The message names the first call the strategy makes and
the line of your source, here `strategy uses pro indicators (Pro.adx, line 25); your plan: free`. The check reads the compiled
strategy, so any other mention of a class of the pro package counts the same, and the message then names the class. The status,
the body and the `upgradeUrl` are the ones "The pro indicators and your plan" in the `qtsurfer-qtscript-strategy` skill
describes.

## Advanced indicator catalogue (statistics & pro)

Beyond the fluent builder methods above, the engine ships ~150 indicator classes across two tiers,
one sub-package per category: `com.wualabs.qtsurfer.engine.indicators.<category>` for the **free**
tier (`averages`, `momentum`, `distance`, `bollinger`, `statistics`, …) and
`com.wualabs.qtsurfer.engine.indicators.<category>.pro` for the **pro** tier
(`averages.pro`, `trend.pro`, `momentum.pro`, `volatility.pro`, `volume.pro`, `statistics.pro`, …) —
a `pro` package segment is always the pro-tier marker. Pro classes come with the Pro and Elite plans, and are
not shipped in the OSS/TypeScript port of the engine. They are plain `RTIndicator` instances — add any of them by
class with `.add("name", new XxxRTIndicator(...))`, then read with `indicators.getValue("name")`.

**A submitted strategy can use a `pro` class.** `import com.wualabs.qtsurfer.engine.indicators.statistics.pro.ZScoreRTIndicator;`
compiles, and the strategy is registered on every plan; what a plan without the pro indicators cannot do is run it (see
"The pro indicators" above). The check reads the compiled strategy, so any mention of a `pro` class counts: an import
that the code uses, a cast, a field, a signature. The example below shows the shape of the call; its last line is pro, the
others are free.

```java
import com.wualabs.qtsurfer.engine.indicators.statistics.StandardDeviationRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.statistics.pro.ZScoreRTIndicator;   // pro: needs the Pro or Elite plan to run

indicators
    .addPrice()                                                      // "price"
    .sma("mean", 20)
    .add("std",    new StandardDeviationRTIndicator(20))             // ctor (int periods)
    .add("stdOf",  new StandardDeviationRTIndicator(                 // ctor (RTIndicator, int periods)
            indicators.getReadOnlyExisting("mean"), 20))
    .add("zscore", new ZScoreRTIndicator(/* see class for ctor */));   // pro: needs the Pro or Elite plan to run
```

Constructors vary per class — most take `(int periods)` and/or `(RTIndicator source, int periods)`;
some (lombok-built) differ, so check the class. Useful classes by category — Trend, Volume, and
performance Ratios are **pro-only today**, the free tier has no indicator in those categories yet:

| Category | Tier | `*RTIndicator` classes |
|---|---|---|
| Moving averages | Free | `Sma`, `Ema`, `Wma`, `Hma`, `Kama`, `Tema`, `Mma` |
| Moving averages | Pro | `Alma`, `Dema`, `Frama`, `LeastSquaresMovingAverage`, `McGinleyDynamic`, `Smma`, `Wma` (O(1) twin), `Envelopes` |
| Statistics | Free | `StandardDeviation`, `Variance` |
| Statistics | Pro | `StandardDeviation`, `Variance` (O(1) twins), `ZScore`, `Skewness`, `Kurtosis`, `RollingPercentile`, `Correlation`, `Covariance`, `Beta`, `LinearRegressionSlope`, `SimpleLinearRegression` |
| Trend | Pro only | `Adx`, `Aroon`, `SuperTrend`, `ParabolicSar`, `Ichimoku`, `DonchianChannel`, `EfficiencyRatio` |
| Volatility | Free | `VolatilityRTIndicator`, `PercentVolatilityRTIndicator` |
| Volatility | Pro | `Atr`, `Natr`, `RealizedVolatility`, `Parkinson`, `GarmanKlass`, `EwmaVolatility` |
| Volume | Pro only | `Vwap`, `Obv`, `Mfi`, `Cmf`, `Adl`, `ElderForceIndex` |
| Oscillators | Free | `Macd`, `StochasticRsi`, `Cci` |
| Oscillators | Pro | `StochasticOscillator`, `Roc`, `Momentum`, `WilliamsR`, `UltimateOscillator` |
| Ratios (performance) | Pro only | `SharpeRatio`, `SortinoRatio`, `CalmarRatio`, `MaxDrawdown`, `OmegaRatio`, `UlcerIndex` |

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

### Rich indicators (full snapshot access)

Scalar indicators receive a single `double` (one field, extracted via `TickerValueSource`).
When an indicator needs **more than one field** — OHLC for ATR, volume+price for VWAP/OBV,
bid/ask for microstructure — implement `RichRTIndicator<T extends MarketSnapshot>` instead and
read the whole snapshot:

```java
import com.wualabs.qtsurfer.engine.indicators.core.RichRTIndicator;
import com.wualabs.qtsurfer.engine.core.MarketSnapshot;

public class MyOhlcIndicator implements RichRTIndicator<MarketSnapshot> {
    private double value;
    @Override public double updateFrom(MarketSnapshot snap) {  // full snapshot: O/H/L/C/V, bid/ask
        this.value = /* combine several fields */ 0;
        return value;
    }
    @Override public double getValue() { return value; }
    @Override public double update(double v) { return value; }  // scalar path unused
    @Override public void reset() { value = 0; }
}
```

The engine builds the snapshot once per tick and dispatches it to every registered
`RichRTIndicator`, while scalar indicators keep receiving their extracted field. Register it the
same way: `indicators.add("myOhlc", new MyOhlcIndicator())`.

## Hidden indicators

Prefix with `_` to exclude from signal reporting metadata:

```java
.gain("_rawGain", "price")   // internal use, not reported
```

This is sugar over the `VISIBILITY` metadata entry — see below.

## Indicator metadata

Every indicator carries small descriptive key/value metadata about itself, kept separate from its
registered lookup name, readable from any `RTIndicator` instance (e.g. via
`indicators.getExisting("name")` / `getReadOnlyExisting("name")`):

```java
RTIndicator ind = indicators.getExisting("gap");
ind.getId();                     // canonical type id, e.g. "distance", "bollinger", "rsi"
ind.getDisplayHint();            // DisplayHint: ABSOLUTE (default), PERCENT, or VOLUME
ind.isHidden();                  // true if internal-only (the "_" prefix above sets this)
ind.getMeta().get("periods");    // any other descriptive key, or null if unset
```

Useful for introspection without parsing the name string — e.g. checking
`getDisplayHint() == DisplayHint.PERCENT` before formatting a value for display, or `getId()` to
branch generically over whatever indicator is registered under a name. `distance()` /
`percentChange()` / `distanceMa()` set `PERCENT` automatically; most other indicators carry no
metadata — `getMeta()` returns the shared `IndicatorMeta.EMPTY`, never `null`.

To attach metadata on a **custom** indicator (see
[Writing a custom RTIndicator](#writing-a-custom-rtindicator)), extend `AbstractRTIndicator` and
use its fluent setters at registration:

```java
import com.wualabs.qtsurfer.engine.indicators.core.AbstractRTIndicator;
import com.wualabs.qtsurfer.engine.indicators.core.IndicatorMeta;
import com.wualabs.qtsurfer.engine.indicators.core.DisplayHint;

indicators.add("gap",
    new MyDistanceIndicator(a, b)
        .withMeta(IndicatorMeta.ID, "distance")
        .withMeta("periods", 20)
        .withDisplayHint(DisplayHint.PERCENT));
```

Metadata is a **write-only descriptor** — an indicator must never read its own metadata back to
drive its computation, that would make it a second, undeclared configuration channel. Set it once
at registration; read it only from the outside.
