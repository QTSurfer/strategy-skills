# QTScript examples

Complete `.qtscript` files — each one is the whole strategy, nothing omitted. QTScript is in beta;
see [SKILL.md](../SKILL.md) for the language and the
**`qtsurfer-java-strategy`** skill for the API the braced bodies are written against.

## 1. RSI reversion (ticker)

The smallest useful strategy: one indicator, one window, two signals.

```
strategy "RSI reversion"

param rsiLow  = 30  "Oversold"
param rsiHigh = 70  "Overbought"

instruments */usdt

setup:
  rsi(14) window m1 {
    if (actual < rsiLow)  emitBuy(price);
    if (actual > rsiHigh) emitSell(price);
  }
```

## 2. EMA cross with a position flag (ticker)

Two indicators read by name, and the shared per-instrument store used to emit on the crossing only
once.

```
strategy "EMA cross"

param fast = 9  "Fast EMA"
param slow = 21 "Slow EMA"

instruments */usdt

setup:
  ema(fast)
  ema(slow)
  window price s5 {
    double f = value("ema" + fast);
    double s = value("ema" + slow);
    if (f > s && !store.is("long")) {
      store.set("long");
      emitBuy(price);
    }
    if (f < s && store.is("long")) {
      store.unset("long");
      emitSell(price);
    }
  }
```

`value("name")` reads any registered indicator; `store` is the same store every window of the
instrument shares.

## 3. Named sections and a shared counter (ticker)

Two windows on different indicators and periods, bodies written at column 0 rather than inline.

```
strategy "Two windows"

param rsiLow = 30

setup:
  rsi(14) window Oversold
  ema(20)
  window ema20 Trend

window Oversold m1 {
  if (actual < rsiLow && store.inc("oversoldBars") >= 3) {
    emitBuy(price);
  }
}

Trend m5 {
  store.setState("trendUp", actual > prev);
}
```

Both bodies write to one store, so `Trend` can leave a value `Oversold` reads on its next close.

## 4. Crossing a level with `prev` (ticker)

A window hands you the value the indicator had when the previous window closed as `prev`, so a crossing needs
nothing in the store. This enters when the RSI climbs back through a level, once, on the window where it
happens.

```
strategy "RSI re-entry"

param level = 30 "Level to climb back through"

setup:
  rsi(14) window m1 {
    if (prev < level && actual >= level) emitBuy(price);
  }
```

The first time a window fires there is no earlier window: `prev` is the value the indicator had when that
first window opened, and an indicator with a warm-up period has not settled yet. Let the first firings pass
when the rule needs two real closes.

## 5. True range from the previous close (kline)

On `kline`, a window on `close` gives the previous bar's close as `prev`, which is what the true range needs
next to the bar's own `high` and `low`.

```
strategy "True range" kline

setup:
  window close m1 {
    double trueRange = Math.max(high - low, Math.max(Math.abs(high - prev), Math.abs(low - prev)));
    emitInfo("trueRange", trueRange);
  }
```

## 6. Candle range breakout (kline)

On `kline` the bar's fields are in scope as plain variables. The bar width is not in the file: it
is the `cadence` you prepare the data at (`1m`, `1h`, `1d`, …), so the same strategy runs at any of
them.

```
strategy "Range breakout" kline

param minRange = 0.5 "Minimum range, percent"

setup:
  window close m1 {
    double range = (high - low) / low * 100.0;
    if (range < minRange) return;
    if (close > open) emitBuy(close);
    else              emitSell(close);
  }
```

## 7. Funding rate watch (funding)

A strategy that computes rather than trades: `emitInfo` attaches fields to an informational signal.
On funding there is no price — the value is `rate`. The platform accepts this file and can prepare
funding data, but a backtest or sweep over funding data is rejected for now.

```
strategy "Funding watch" funding

instruments ~"^[A-Z]+/USDT:USDT$"

Main m5 {
  emitInfo("funding", rate, "instrument", instrument.symbol());
}
```

The regular expression keeps this to perpetuals quoted and settled in USDT, which is what a funding
strategy sees.

## 8. An initialiser and the native instrument filter

`init` for engine setters, and the braced form of `instruments` when a pattern is not enough.

```
strategy "Configured"

param gain = 0.5 "Minimum percent gain"

init {
  setPercentGain(gain);
  setMultiEntryEnabled(true);
}

instruments {
  if (instrument.quote().equalsIgnoreCase("usdt")) {
    return !instrument.base().equals("USDC");
  }
  return false;
}

setup:
  ema(12) window m1 {
    if (actual > prev) emitBuy(price);
  }
```

> `init` cannot assign a `param` — `setPercentGain(gain)` *reads* one, which is fine. Assigning
> `gain` there would be overwritten by the declared default; the compiler warns.

## 9. Pro indicators: bands and a cloud (kline)

`uses pro.*` brings in the thirteen pro indicators, which read whole bars and so belong in a `kline` strategy. This one
uses bands and a cloud: a window on `superTrend` or `ichimoku` attaches to the main line, and the body reads the others with
`value($indicator + "…")`. Running it needs the Pro or Elite plan; on another plan the answer is a `402`.

```
strategy "Cloud and bands" kline
uses pro.*

setup:
  superTrend(10, 3) window m5 {
    if (close > value($indicator + "Upper")) emitSell(close);
    if (close < value($indicator + "Lower")) emitBuy(close);
  }
  ichimoku() window m5 {
    if (close > value($indicator + "SenkouA") && close > value($indicator + "SenkouB")) emitBuy(close);
  }
  vwap()
  yangZhang(20)
```
