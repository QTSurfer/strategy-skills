---
name: qtsurfer-qtscript-strategy
description: Write, review, and debug QTSurfer strategies in QTScript (.qtscript) — a compact strategy language, currently in beta, whose braced bodies are plain Java. Use when writing a strategy as sections (strategy, param, init, instruments, setup:, windows) instead of a Java class, or when reading QTScript compile and run-time errors. For the full Java strategy API — the indicator catalogue, listeners, state and signals used inside every body — see the qtsurfer-java-strategy skill.
license: Apache-2.0
metadata:
  version: 1.0.0
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
whose first token is `strategy` is QTScript — so a `.qtscript` source goes wherever a Java source
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

`kline` never takes an interval: candles are one-second, and there is no syntax for anything else.
The multi-source base is not offered, matching what the public backtest path runs.

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

Each line is one call on the indicator builder, without the semicolon — the same builder and the
same indicator catalogue a Java strategy uses (`references/indicators.md` in the
`qtsurfer-java-strategy` skill). One call per line; a call may not span lines.

**`setup:` is the one place indentation matters.** Its body is the indented lines that follow it,
and the first line back at column 0 ends the section.

The value indicators are registered for you, before your lines: `price` on ticker; `price` (the
close) plus `open`, `high`, `low`, `close` and `volume` on kline; `rate` on funding — where there is
no price at all, only the rate.

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

**Attaching by position has one limit.** The inline postfix form deduces which indicator the line
registered, so it cannot be used on a call that publishes several names — `bollinger(20, 2)`
publishes three. The compiler says so, names the candidates and points at the by-name form; use
`window blgr20_2 m1 { … }`.

### Inside a body

Your Java, plus what is already in scope:

| In scope | What it is |
|---|---|
| `actual`, `prev` | the window's new and previous value |
| `store` | the per-instrument `StateStore`, shared by every window of that instrument |
| `price` (ticker), `price open high low close volume` (kline), `rate` (funding) | the current values, as plain variables |
| `value("name")` | any other indicator's current value |
| `emitBuy(price)`, `emitSell(price)`, `emitInfo(key, values…)`, `emitSignal(signal)` | signal emission |
| `indicators`, `instrument`, `strategy` | the listener's own fields |
| `getPrevInstant()`, `getCurrInstant()`, `getEngineVersion()` | the closing window's boundaries, the running engine |

Every `param` is readable by name, and nothing needs importing: the packages a strategy is allowed
to use are already in scope.

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
QTS line 6: Index 2 out of bounds for length 1
```

Two more things to know:

- **Reserved names.** The keywords (`strategy`, `param`, `instruments`, `init`, `setup`, `window`,
  `kline`, `funding`), the in-scope names above (`price`, `actual`, `store`, …) and the engine's own
  property keys cannot be used as parameter names. You are told which, and why, at compile time.
- **Size.** A strategy is compiled and stored as one class; a source large enough to exceed the
  platform's stored-class budget is refused when you register it, with the size in the message,
  rather than failing later.

## More

- [references/examples.md](references/examples.md) — worked strategies, one per data source.
- The **`qtsurfer-java-strategy`** skill — the strategy API every body is written against: indicator
  catalogue, window listeners, `StateStore`, signal emission, and the Java route itself.
