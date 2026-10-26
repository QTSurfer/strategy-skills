---
name: qtsurfer-qtscript-strategy
description: Write, review, and debug QTSurfer strategies in QTScript (.qtscript) — a compact strategy language, currently in beta, whose braced bodies are plain Java. Use when writing a strategy as sections (strategy, param, init, instruments, uses, setup:, windows, onCommand) instead of a Java class, or when reading QTScript compile and run-time errors. For the full Java strategy API — the indicator catalogue, listeners, state and signals used inside every body — see the qtsurfer-java-strategy skill.
license: Apache-2.0
metadata:
  version: 1.1.0
---

# QTSurfer QTScript Strategy

> **Beta.** QTScript is one of two ways to write a QTSurfer strategy, and the newer one. The other —
> a Java class extending a strategy base class — is the established route, has the full power of the
> engine API, and is documented in the **`qtsurfer-java-strategy`** skill. Nothing here replaces it:
> anything QTScript cannot express is a reason to write Java directly. Expect this language to gain
> syntax; what is documented here is what the platform accepts today.

QTScript removes the ceremony around a strategy — package, imports, class declaration, base class,
`@StrategyProperty` blocks, listener boilerplate — and keeps the part that is actually yours.
**Every `{ }` body is plain Java**, copied through as written, so the whole strategy API from the
`qtsurfer-java-strategy` skill applies inside a body unchanged: the same indicators, the same
`StateStore`, the same signal emission.

A `.qtscript` source becomes exactly one Java class and is compiled by the same server-side chain a
Java strategy goes through.

## A whole strategy

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

That is the complete file: no imports, no class, no listener class. It registers a 14-period RSI,
watches it on a one-minute window and emits on the crossings.

## Submitting one

Nothing new to configure. The platform decides the language from the **source text** — a strategy
whose first token is `strategy` is QTScript, and whitespace or comments (`//`, `/* */`) before it are
ignored, so a description can sit on top of the file — so a `.qtscript` source goes wherever a Java source
goes: the API, the SDK, or `submit_backtest` through the
[MCP server](https://github.com/QTSurfer/mcp-java). The file extension is a convention for you and
your editor; only the text is sent.

## `strategy` — the header

```
strategy "RSI reversion"          // ticker (default)
strategy Reversion                // a bare identifier works too
strategy Bars kline               // candles
strategy Rates funding            // funding rates
```

The first line of every file. A quoted name is kept verbatim as the strategy's title; the Java class
name is derived from it (`"RSI reversion"` → `RSIreversion`). The optional keyword picks the data
source, and maps to the base class you would otherwise have extended (ticker, kline or funding-rate
— see *Strategy base classes* in the `qtsurfer-java-strategy` skill).

`kline` never takes an interval: the bar width is not part of the strategy. You choose it when you
prepare the data for a run — the `cadence` of the prepare request: `1s` (the default) up to `1q`, the same labels as for
tickers (`5s`, `3m`, `8h` and `1w` among them) — and the same file can be run at several cadences. The
multi-source base is not offered, matching what the public backtest path runs.

**What can be run today.** `ticker` and `kline` strategies can be executed, swept and
walk-forward validated. A `funding` strategy registers and its data can be prepared, but a run or a
sweep over funding data is rejected with a `400` for now (`funding data can be prepared but not
executed yet`), so a funding file is accepted and cannot yet be backtested.

## `param` — configurable values

```
param fastEma = 9     "Fast EMA period"
param rsiLow  = 30
param useVol  = true
```

One line per parameter. Each becomes a configurable property you read as a plain variable in any
body. The type comes from the literal: `9` → `int`, `0.5` → `double`, `true` → `boolean`,
`"text"` → `String`. The description is optional.

**The parameter name is the key** — the same key a run's `params`, a sweep axis and the strategy's
declared properties use. Names the engine already uses for its own properties are rejected when you
compile, naming the line, rather than being silently ignored at run time.

## `init { }` — constructor, optional

```
init {
  setPercentGain(0.5);
  setExecutionMode(ExecutionMode.LONG_MULTI);
}
```

Plain Java, run once when the strategy is built: engine setters and your own initialisation.

**Do not assign a `param` here.** Declared defaults are applied after construction, and run-time
values later still, so an assignment in `init` is overwritten without a word. Anything that depends
on a parameter belongs in `setup:` or in a window body; a compile-time warning points at it.

## `onCommand { }` — handling a command, optional

```
onCommand {
  if ("flatten".equals($command)) {
    // close the position, cancel pending orders, whatever "flatten" means for this strategy
  }
}
```

A live run's owner can tell it a command from outside — `POST /live/{runId}/commands` with
`{"command": "<text>"}` — while it keeps running, without restarting it (see "Receiving commands" in
the `qtsurfer-java-strategy` skill for the full contract). `onCommand { }` is how a QTScript strategy
acts on one: at most one per file, plain Java, run once per command rather than once per market event.
It has no period and no indicator — it is not a window, the same way `init { }` is not one.

Inside its body, `$command` is the command's text, as a `String`. Nothing a window body has —
`actual`, `$indicator`, `value(...)`, the ambient `store` — is in scope, because a command is not tied
to a market tick or, unlike a window, to one instrument already chosen for you. Every `param` is
readable and settable.

A command may carry a `properties` object of your own choosing, alongside `command` in the request
body. Read a value from it with `$command.<key>` — a `String`, `null` when the command carried no such
key. `$command.<key>` fires only when `<key>` is not itself a call, so `$command.equals(...)`,
`$command.startsWith(...)` and the rest still read as ordinary `String` methods on `$command` itself.

A window body gets its instrument's `store` handed to it; `onCommand` does not, since a command names
no instrument on its own — but `getStateStore("<symbol>")` takes one directly, so a command whose own
properties name an instrument can still reach that instrument's store:

```
strategy "Manual flatten"

onCommand {
  if ("flatten".equals($command)) {
    getStateStore($command.instrument).set("flattened");
  }
}
```

A strategy with no `onCommand { }` does not implement `CommandRequestHandler`, so
`POST /live/{runId}/commands` answers every command sent to one of its runs with a `409`. A command is
always a plain string and is transient, exactly as in Java — see "Receiving commands" in the
`qtsurfer-java-strategy` skill for what that means across a restart. Setting a `param` or writing to a
`StateStore` from inside `onCommand` both take effect immediately, but neither survives a restart: a
`StateStore` is memory, gone on a restart the same as a field. Only `PUT /live/{runId}/params` writes
something a restarted replica actually starts from. `getStateStore(...)` here is not about durability —
it is how `onCommand` reaches the per-instrument state a window body already reads, since a command
carries no instrument of its own.

## `instruments` — which markets, optional

Two forms; a file uses one or the other.

```
instruments */usdt                                  // any base quoted in USDT
instruments btc/usdt, eth/*, ~"^SOL/(USDT|USDC)$"   // a list: any entry accepts
```

- A **pair** is `BASE/QUOTE`, each side a currency code or `*`, compared ignoring case. It matches on
  base and quote whatever the kind of instrument, so `*/usdt` accepts the spot `BTC/USDT` and the
  perpetual `BTC/USDT:USDT`.
- A **regular expression**, `~"..."`, must match the whole symbol (`BTC/USDT`, `BTC/USDT:USDT`,
  `BTC/USDT:USDT-260327`). It is how you tell spot from a perpetual.
- Entries are OR-ed. `instruments *` is the same as writing no section at all.

The declarative form **narrows** the engine's own instrument filter rather than replacing it, so the
output-currency match and the whitelist/blacklist still apply. Mistakes — a malformed pair, a
currency that is not a code, a stray comma, a duplicate, a regular expression that does not compile —
are reported when you compile, with the line and column, because nothing downstream would catch them.

The native form replaces the default, exactly like overriding the method in Java:

```
instruments {
  return instrument.quote().equalsIgnoreCase("usdt");
}
```

It is the body of `acceptInstrument(Instrument instrument)`: every path must return, and
`super.acceptInstrument(instrument)` is available if you want to keep the default and add to it.

Not in this version: exclusions, partial wildcards (`BT*`), and a settlement currency inside a pair —
use a regular expression for the last one.

## `setup:` — indicators, one call per line

```
setup:
  ema(12)
  ema(26)
  distance("distEma12_26", "ema12", "ema26")
  bollinger(20, 2)
```

Each line is one call on the indicator builder, without the semicolon — the same builder a Java
strategy uses (`references/indicators.md` in the `qtsurfer-java-strategy` skill describes its
catalogue). One call per line; a call may not span
lines.

**The names that work** are the builder's methods that take plain values:

`add` `bollinger` `bollingerBandwidth` `cci` `clamp` `diff` `distance` `distanceMa` `ema` `equal`
`gain` `greatOrEqual` `greatThan` `lessOrEqual` `lessThan` `loss` `macd` `max` `min` `mul` `notEqual`
`percentChange` `rateChange` `round` `rsi` `sma` `stochRsi` `sum`

These are the forms of each name that a QTScript line can write: a `String` is the name of an indicator, either one it
reads or the one it registers, and a `Number` is a whole number or a decimal. What each argument means is in the Javadoc of
the builder, `InstrumentGroupRTIndicator`.

| Indicator | The forms QTScript can write |
|---|---|
| `add` | `add(String, String)` · `add(String, String, String)` |
| `bollinger` | `bollinger()` · `bollinger(int)` · `bollinger(int, Number)` · `bollinger(String, int, Number)` |
| `bollingerBandwidth` | `bollingerBandwidth(String)` |
| `cci` | `cci(int)` · `cci(String, int)` |
| `clamp` | `clamp(String, Number, Number)` · `clamp(String, String, Number, Number)` |
| `diff` | `diff(String, String)` · `diff(String, String, String)` |
| `distance` | `distance(String, String)` · `distance(String, String, String)` |
| `distanceMa` | `distanceMa(String)` · `distanceMa(String, String)` |
| `ema` | `ema(int)` · `ema(String, int)` · `ema(String, String, int)` |
| `equal` | `equal(String, String, Number)` · `equal(String, String, Number, String)` · `equal(String, String, Number, String, String)` |
| `gain` | `gain(String)` · `gain(String, String)` · `gain(String, String, boolean)` |
| `greatOrEqual` | `greatOrEqual(String, String, Number)` · `greatOrEqual(String, String, Number, String)` · `greatOrEqual(String, String, Number, String, String)` |
| `greatThan` | `greatThan(String, String, Number)` · `greatThan(String, String, Number, String)` · `greatThan(String, String, Number, String, String)` |
| `lessOrEqual` | `lessOrEqual(String, String, Number)` · `lessOrEqual(String, String, Number, String)` · `lessOrEqual(String, String, Number, String, String)` |
| `lessThan` | `lessThan(String, String, Number)` · `lessThan(String, String, Number, String)` · `lessThan(String, String, Number, String, String)` |
| `loss` | `loss(String)` · `loss(String, String)` · `loss(String, String, boolean)` |
| `macd` | `macd()` · `macd(String)` · `macd(String, int, int)` · `macd(String, String, int, int)` |
| `max` | `max(String)` · `max(String, String)` |
| `min` | `min(String)` · `min(String, String)` |
| `mul` | `mul(String, Number)` · `mul(String, String)` · `mul(String, String, Number)` · `mul(String, String, String)` |
| `notEqual` | `notEqual(String, String, Number)` · `notEqual(String, String, Number, String)` · `notEqual(String, String, Number, String, String)` |
| `percentChange` | `percentChange(String)` · `percentChange(String, String)` |
| `rateChange` | `rateChange(String)` · `rateChange(String, String, boolean)` |
| `round` | `round(String, int)` · `round(String, String, int)` |
| `rsi` | `rsi(int)` · `rsi(int, boolean)` · `rsi(String, int, boolean)` |
| `sma` | `sma(int)` · `sma(String, int)` · `sma(String, String, int)` · `sma(String, int, boolean)` |
| `stochRsi` | `stochRsi(int)` · `stochRsi(String, int)` |
| `sum` | `sum(String)` · `sum(String, String)` |

A call whose arguments match none of them is rejected with the forms that do exist: `ema(1, 2, 3)` answers that its forms are `ema(int)`,
`ema(String, int)` and `ema(String, String, int)`. The methods that take a function (`predicate`, `fun`, `decorate`,
`periodCount`) and `conditional`, which takes a function and indicators as objects, can only be written in Java. To
compare an indicator with a number, use `equal`, `notEqual`, `greatThan`, `greatOrEqual`, `lessThan` or `lessOrEqual`.

**A name that is not on the list, and that no `uses` line brings in (next section), is rejected as `unknown indicator
'<name>'`.** `wma` and `zscore` are among the names that answer this way, and so are `atr` and `adx` unless the file
brings them in with `uses`. So is `value("name")`, which reads an indicator inside a `{ }` body and has no meaning on a
`setup:` line.

**`setup:` is the one place indentation matters.** Its body is the indented lines that follow it,
and the first line back at column 0 ends the section.

The value indicators are registered for you, before your lines: `price` on ticker; `price` (the
close) plus `open`, `high`, `low`, `close` and `volume` on kline; `rate` on funding — where there is
no price at all, only the rate.

## The pro indicators: `uses`

The pro indicators are not in the list above. A file brings them in with a `uses` line, and then calls them in `setup:` by
their bare name, like any other indicator.

- `uses pro.*` brings in all thirteen. `uses pro.adx, pro.ichimoku` brings in only those. Several `uses` lines add up.
  Write them with `param` and `instruments`, before `setup:`.
- Only the names that a `uses` line brings in are pro. With `uses pro.atr`, a call to `adx(14)` is still `unknown
  indicator 'adx'`, reported on the line of the call. The builder's own names (`ema`, `rsi` and the rest of the list
  above) are never pro.
- `pro` is the only set there is. `uses` is a reserved word: a file cannot have a `param` or a section called `uses`.
- Pro indicators read whole bars, so use them in a `kline` strategy. On a `ticker` strategy the values are not
  the bar's and mean nothing, and validating or running the strategy returns an engine notice that says so.
- A pro line is an ordinary `setup:` line: one call per line, with a window after it as usual.
- Backtests, sweeps (walk-forward included), validation (`POST /strategy/{id}/validate`) and live runs run the pro
  indicators.

| Indicator | Forms | Registers, with the default arguments | Also registers |
|---|---|---|---|
| `atr` | `atr()` · `atr(int)` · `atr(String, int)` | `atr14` | |
| `adx` | `adx()` · `adx(int)` · `adx(String, int)` | `adx14` | `adx14PlusDI` · `adx14MinusDI` |
| `superTrend` | `superTrend()` · `superTrend(int, double)` · `superTrend(String)` · `superTrend(String, int, double)` | `st10_3` | `st10_3Upper` · `st10_3Lower` |
| `ichimoku` | `ichimoku()` · `ichimoku(int, int, int)` · `ichimoku(String)` · `ichimoku(String, int, int, int)` | `ichi9_26_52`, the Tenkan line | `ichi9_26_52Kijun` · `ichi9_26_52SenkouA` · `ichi9_26_52SenkouB` · `ichi9_26_52Chikou` |
| `obv` | `obv()` · `obv(String)` | `obv` | |
| `vwap` | `vwap()` · `vwap(String)` | `vwap` | |
| `mfi` | `mfi()` · `mfi(int)` · `mfi(String, int)` | `mfi14` | |
| `cmf` | `cmf()` · `cmf(int)` · `cmf(String, int)` | `cmf20` | |
| `kurtosis` | `kurtosis(int)` · `kurtosis(String, int)` | `kurt20` (the period is required; it reads the close) | |
| `parkinson` | `parkinson(int)` · `parkinson(String, int)` · `parkinson(String, int, double)` | `park20` | |
| `garmanKlass` | `garmanKlass(int)` · `garmanKlass(String, int)` · `garmanKlass(String, int, double)` | `gk20` | |
| `rogersSatchell` | `rogersSatchell(int)` · `rogersSatchell(String, int)` · `rogersSatchell(String, int, double)` | `rs20` | |
| `yangZhang` | `yangZhang(int)` · `yangZhang(String, int)` · `yangZhang(String, int, double)` | `yz20` | |

The `double` of the four volatility estimators is the annualization factor. With other arguments the name follows the same
pattern: `atr(20)` registers `atr20`, `kurtosis(30)` registers `kurt30`, `ichimoku(7, 22, 44)` registers `ichi7_22_44`, and
`superTrend(10, 2.5)` registers `st10_2.5` (a whole multiplier loses its decimal: `superTrend(7, 3)` registers `st7_3`).
With a name as the first argument the indicator registers under that name, and the other indicators it registers take it
as their prefix: `adx("trend", 14)` registers `trend`, `trendPlusDI` and `trendMinusDI`.

Some of them register more than one indicator under one name, as `bollinger` does (see "A window on a call that publishes
several names" under Windows). A window written on such a line attaches to the main
one, so `actual` is its value, and the body reads the others with `value($indicator + "<suffix>")`, using the suffixes of
the last column. `$indicator` is the name the window is attached to, so it also works when you gave the indicator a name
of your own. The main one of `adx` is the ADX, that of `superTrend` is the SuperTrend line, and that of `ichimoku` is the Tenkan line.

This reads the ADX with the two directional lines beside it:

```
strategy "ADX trend" kline
uses pro.adx, pro.atr

param adxLevel = 25 "ADX level"

setup:
  ema(20)
  atr(14)
  adx(14) window m1 {
    if (actual > adxLevel && value("adx14PlusDI") > value("adx14MinusDI")) emitBuy(close);
    if (actual > adxLevel && value("adx14PlusDI") < value("adx14MinusDI")) emitSell(close);
  }
```

`references/examples.md` has one that brings in all of them and uses bands and a cloud.

A file that brings in only `atr` and then calls `adx(14)` does not compile. The line of the call is rejected with
`unknown indicator 'adx'`:

```
uses pro.atr

setup:
  adx(14)
```

A `uses` line that is not well formed is rejected, against your file like the other errors of QTScript, with one of
these messages:

- `uses needs the indicators to bring in: uses pro.* or uses pro.adx, pro.ichimoku`, when nothing follows `uses`.
- `expected pro.* or pro.<indicator>, found 'pro'`, for `uses pro`.
- `unknown set 'ta': the only set is pro`, for a set that does not exist.
- `unknown pro indicator 'zscore': the pro indicators are adx, atr, cmf, garmanKlass, ichimoku, kurtosis, mfi, obv, parkinson,
  rogersSatchell, superTrend, vwap, yangZhang`, for a member that does not exist.
- `expected pro.* or pro.<indicator> here`, for a comma with nothing after it, as in `uses pro.adx,`.
- `expected pro.* or pro.<indicator>, found 'pro.adx pro.atr'`, for two members with no comma between them.

### The pro indicators and your plan

The pro indicators come with the Pro and Elite plans (your plan is the `tier` that `GET /account` returns).
Registering a strategy that uses them is accepted on every plan; what a plan without them cannot do is run it.

- **`POST /strategy` says what the strategy needs.** The answer carries `requires`, a list: `["indicators.pro"]` when the
  strategy uses the pro indicators and `[]` when it does not. `GET /strategy/{strategyId}` does not show it.
- **A plan that does not include them is refused with `402 Payment Required`**, without a `Retry-After` header, since waiting
  does not change the plan. That is the answer to validating the strategy, to starting a backtest with it, to starting a
  sweep with it (walk-forward included) and to starting a live run with it. The message names the first pro indicator the
  strategy asks for, the line of your file and your plan (on the Basic plan it ends `your plan: basic`). The body is the usual
  error, with `retryable` set to `false`, so a client that retries on `retryable` does not retry this one. When the platform
  has a page where you change plan, the body also carries its address as `upgradeUrl`; no other error does:

```json
{"code": 402, "message": "strategy uses pro indicators (pro.adx, QTScript line 7); your plan: free", "retryable": false, "upgradeUrl": "https://example.com/upgrade"}
```

## Windows — where the logic goes

A window fires when its period closes, not on every tick, and wraps one source indicator. Five ways
to write one:

```
setup:
  rsi(14) window m1 { … }        // inline, attached to the indicator this line registers
  rsi(33) window Oversold        // same, but the body is a named section below
  window price m5 { … }          // inline, attached to an indicator by name
  window ema12 Trend             // by name, body in a named section

window Oversold m1 { … }         // a named section, at column 0
Trend s5 { … }                   // the `window` keyword is optional here
```

**Period** is a `WindowTime` member — `s1 s5 s10 s30 m1 m3 m5` — or a bare integer meaning seconds
(`window 900 { … }`). No quotes and no units: write `s1`, not `1s`. Omitted, it is `s1`. A reference
to a named section takes that section's period.

A section named `Main` that nothing references is attached to the primary value (`price`, or `rate`
on funding), which makes the shortest useful file:

```
strategy Simple

Main m1 {
  if (actual > prev) emitBuy(price);
}
```

**The window gives you the previous value.** A window fires when its period closes, so `prev` is the
value the indicator had when the **previous** window closed and `actual` is its value now — the `close[1]`
of other tools. A crossing, or anything else that compares one close with the one before, needs nothing
kept in the `store`: the previous value is already in scope. Entering when the RSI climbs back through a
level, once, on the window where it happens:

```
strategy "RSI re-entry"

param level = 30 "Level to climb back through"

setup:
  rsi(14) window m1 {
    if (prev < level && actual >= level) emitBuy(price);
  }
```

On a `kline` strategy a window on `close` hands you the previous bar's close as `prev`, which is what the
true range needs next to the bar's own `high` and `low`:

```
strategy "True range" kline

setup:
  window close m1 {
    double trueRange = Math.max(high - low, Math.max(Math.abs(high - prev), Math.abs(low - prev)));
    emitInfo("trueRange", trueRange);
  }
```

The first time a window fires there is no earlier window to compare with: `prev` is the value the
indicator had when that first window opened. A rule that needs two real closes should let the first firing
pass, and an indicator with a warm-up period, such as an RSI or an EMA, has not settled in its first
windows either. Keep the `store` for what a window cannot hand you: a flag, a counter, a value from
several windows ago.

**A window on a call that publishes several names.** `bollinger(20, 2)` registers three indicators
under one name: the middle band, `blgr20_2`, and the outer bands, that name followed by `Upper` and
`Lower`. The inline postfix form attaches to the middle band, so `bollinger(20, 2) window m1 { … }`
works and `actual` is the middle band's value. The by-name form, `window blgr20_2 m1 { … }`, is the
same window.

**`$indicator` is the name of the indicator the window is attached to**, as a `String` (`"ema12"` on
`ema(12) window m1 { … }`, `"price"` on a `Main`, `"rate"` on funding). It is how a body reaches the
outer bands that its own line registered, and what tells a named section apart when it is attached to
several indicators (each time it fires it sees the one that fired):

```
setup:
  bollinger(20, 2) window m1 {
    if (price > value($indicator + "Upper")) emitSell(price);
    if (price < value($indicator + "Lower")) emitBuy(price);
  }
```

The `$` marks a name QTScript provides, so the names you declare cannot start with one.

**A window on a name that is not registered** (`window nosuch m1 { … }`) is not rejected when you
register the source: whether the name exists can depend on your `param`s and on indicators registered
from Java. It is found when the strategy is validated (see "When something is wrong").

### Inside a body

Your Java, plus what is already in scope:

| In scope | What it is |
|---|---|
| `actual`, `prev` | the window's new value and the one it had when the previous window closed |
| `$indicator` | the name of the indicator this window is attached to, as a `String` |
| `store` | the per-instrument `StateStore`, shared by every window of that instrument |
| `price` (ticker), `price open high low close volume` (kline), `rate` (funding) | the current values, as plain variables |
| `value("name")` | any other indicator's current value |
| `emitBuy(price)`, `emitSell(price)`, `emitInfo(key, values…)`, `emitSignal(signal)` | signal emission |
| `indicators`, `instrument`, `strategy` | the listener's own fields |
| `getPrevInstant()`, `getCurrInstant()`, `getEngineVersion()` | the closing window's boundaries, the running engine |

Every `param` is readable by name, and nothing needs importing: the packages a strategy is allowed
to use are already in scope.

## Sizing

A strategy says when and which way, not how much. `emitBuy(price)` and `emitSell(price)` carry a price and
nothing else, and no call puts a quantity or a notional on a signal. The size of each entry is a setting of
the run, `percentAmountToLock`: the percentage of the account's free balance that an entry locks, above 0 and
up to 100.

Where it is given depends on the run: `baseConfig` for a backtest or a sweep, the `paper` block for a paper
live run. Left out, a backtest puts the whole free balance into each entry (all-in) and a live run's paper
account locks 10%. To see how a strategy behaves at another size, run it again with another
`percentAmountToLock`; the file does not change.

## What QTScript does not do

By design — and each one is a reason to write the strategy in Java instead, with the
`qtsurfer-java-strategy` skill:

- **No `update()`.** Windows are the model; a strategy that has to see every tick belongs in Java.
- **No cross-instrument logic.** Reading other instruments' indicators (the cross-instrument pattern
  in that skill) needs the full class.
- **No custom indicator classes**, no extra fields or methods beyond what the sections declare, and
  no multi-source strategies.
- **One class.** Anything that wants helper types is a Java strategy.

Switching is never a dead end: a `.qtscript` file is a Java class with the ceremony left out, so the
Java version of the same strategy is the template in the other skill with your bodies pasted in.

## When something is wrong

Compile errors are reported against **your** line and column, never the generated Java:

```
Line 5, Column 13: incompatible types: java.lang.String cannot be converted to int
```

That covers both kinds of mistake: QTScript's own (an unknown section, a bad period, a duplicate
parameter, a malformed instrument pattern) and Java's, from inside a body. A failure while the
strategy is running is reported the same way, on the line the body came from:

```
QTScript line 6: Index 2 out of bounds for length 1
```

**A `200` from registering means the source parsed and compiled, not that it will run.** Registering
does not set the strategy up, so what only shows when it does (a window on an indicator that is not
registered, for one) is found by **validating** it (`POST /strategy/{id}/validate`), which reports it
against your file the same way: `QTScript line 4: unknown indicator 'nosuch'`. Validate a strategy you
have just written before you run it.

Two more things to know:

- **Reserved names.** The keywords (`strategy`, `param`, `instruments`, `init`, `setup`, `window`,
  `kline`, `funding`, `onCommand`), the in-scope names above (`price`, `actual`, `store`, `$command`, …)
  and the engine's own property keys cannot be used as parameter names. You are told which, and why,
  at compile time.
- **Size.** The source is capped at 32 KiB: a larger body is refused with `413` and the API's JSON
  error, which names the cap. A strategy is also compiled and stored as one class; a source large
  enough to exceed the platform's stored-class budget is refused when you register it, with the size
  in the message, rather than failing later.

## More

- [references/examples.md](references/examples.md) — worked strategies, one per data source.
- The **`qtsurfer-java-strategy`** skill — the strategy API every body is written against: indicator
  catalogue, window listeners, `StateStore`, signal emission, and the Java route itself.
