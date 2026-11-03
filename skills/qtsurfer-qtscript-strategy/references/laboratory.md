# QTScript in the Laboratory

The QTSurfer Laboratory runs a `.qtscript` file in the browser, on market data the person has loaded, and shows
its signals, trades and metrics as the file changes. It does not send the file to the platform: it translates
the bodies itself, and that translation covers a subset of Java. Everything in [SKILL.md](../SKILL.md) about
the language holds here; this page says what is narrower, and what does not apply.

## What runs

- **`ticker` and `kline` strategies.** A `funding` strategy is refused when it compiles:
  `funding strategies are not supported offline yet: they run on the server`. Do not write one for the
  Laboratory.
- **Every section**: `strategy`, `param`, `init`, `instruments`, `uses`, `setup:`, windows and named sections.
  `onCommand` compiles, but nothing sends a command in the Laboratory.
- **Every indicator** of [indicators.md](indicators.md), the pro ones included: in the Laboratory `uses pro.*`
  runs for everyone. The plan only limits what the platform runs.

## Inside a body

| Works | Refused here, for example |
|---|---|
| `double`, `int`, `long`, `boolean` and `String` locals | arrays, and `for (x : xs)` |
| `if`/`else`, `for` and `while` with an index, `switch`, `? :` | `List`, `ArrayList`, `Map`, `HashMap`, streams, lambdas |
| `Math.*`, `Double.MAX_VALUE`, `Double.valueOf`, casts | `String.format`, `BigDecimal`, `try`/`catch` |
| what SKILL.md lists in scope: `actual`, `prev`, the bar's fields, `value(...)`, `emitBuy`, `emitSell`, `emitInfo`, `getPrevInstant()` | `System.currentTimeMillis()` and the rest of `System` but `out` |
| the `store` methods of "Keeping state between windows" in SKILL.md | `store.getCounter`, `store.getAccumulator`, `store.getOrCreateState` |

What the Laboratory cannot translate is refused when the file compiles, on its line, with
`… is not supported offline yet: it runs on the server`. Keep past values in the store as several keys, as
example 11 of [examples.md](examples.md) does, rather than in an array.

Do not write `System` at all. `System.out.println` runs here and prints to the Laboratory's output, but the
platform refuses any file that names `System` before it compiles it:
`Strategy source names a forbidden class/package: System`.

## Runs here, fails on the platform

The Laboratory drops the casts to `Double`, `Integer` and the other boxes, and does not unbox, so it is more
lenient than the platform with the store. Numeric casts, such as `(int) d`, it runs as Java does. Write the reads as SKILL.md says, because these run here and fail there:

- a read with no default used as a number: `double x = (Double) store.getState("x")` before anything was
  written is a `NullPointerException`;
- a value read as another type than it was written: `store.setState("p", 0)` and then
  `(Double) store.getState("p", 0.0)` is a `ClassCastException`;
- a counter read as a value: `(Integer) store.getState("bars", 0)` after `store.inc("bars")` is a
  `ClassCastException`.

## What does not apply here

The Laboratory has no API in front of it: nothing is registered, validated or sized from the file. Where
SKILL.md talks about submitting a strategy, `POST /strategy`, validating it, the plan and `402`,
`percentAmountToLock`, or sending a command to a live run, that is the platform, not the Laboratory. The
settings of a run are in the Laboratory, not in the file, and an error shows in the editor, on its line, as
soon as the file compiles.
