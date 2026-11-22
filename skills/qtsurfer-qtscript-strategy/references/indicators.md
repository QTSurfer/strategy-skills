# QTScript indicators

Every indicator a `.qtscript` file can call, grouped by kind, with what each one computes. A line under `setup:` writes one
call by its bare name, like `ema(20)` or `atr(14)`.

- **The arguments each name accepts** are the forms in the table under "What goes under `setup:`" in [SKILL.md](../SKILL.md).
  This file does not repeat them.
- The names that are not pro work on every plan. There are 39.
- **Pro** names come with the Pro and Elite plans and need a `uses` line (see "The pro indicators: `uses`" in
  [SKILL.md](../SKILL.md), which has their forms and the names they register). There are 12.
- Called without a name of its own, an indicator registers a name made from its own and its parameters: `rsi(14)` registers
  `rsi14`, `ema(9)` registers `ema9`, `wma(5)` registers `wma5`. Where the name is not that obvious, the notes give it.
- A name that is on neither list, and a pro name whose `uses` line is missing, is rejected as `unknown indicator '<name>'`.

| Group | Names |
|---|---:|
| Averages | 7 |
| Oscillators | 4 |
| Bands | 3 |
| Volatility | 5 |
| Distance and change | 4 |
| Counting and extremes | 5 |
| Arithmetic | 3 |
| Transformations | 2 |
| Comparisons | 6 |

## Averages

| Name | What it is | Notes |
|---|---|---|
| `sma` | Simple moving average | Discrete by default; `sma(String, int, boolean)` with `false` is the continuous mode |
| `ema` | Exponential moving average | |
| `wma` | Weighted moving average | |
| `tema` | Triple exponential moving average | |
| `hma` | Hull moving average | |
| `kama` | Kaufman's adaptive moving average | Takes the efficiency-ratio period and the fast and slow periods; `kama()` registers `kama10_2_30` |
| `mma` | Moving average with Wilder's smoothing | |

## Oscillators

| Name | What it is | Notes |
|---|---|---|
| `rsi` | Relative strength index | Cutler's by default; `rsi(int, boolean)` with `false` is Wilder's smoothing |
| `stochRsi` | Stochastic RSI | Between 0 and 100; `stochRsi(14)` registers `stochRsi14` |
| `macd` | Moving average convergence divergence | 12 and 26 by default; `macd()` registers `macd`; the last form runs it on another indicator |
| `cci` | Commodity channel index | `cci(20)` registers `cci20` |

## Bands

| Name | What it is | Notes |
|---|---|---|
| `bollinger` | Bollinger bands | Registers the middle line and, beside it, `<name>Upper` and `<name>Lower` |
| `bollingerBandwidth` | Width of a Bollinger band, in percent | Takes the name of the band it measures |
| `bollingerPercentB` | Bollinger %B, where the price sits within the bands | Carries bands of its own; `bollingerPercentB()` registers `pctB20_2` |

## Volatility

| Name | What it is | Notes |
|---|---|---|
| `atr` | Average true range, with Wilder's smoothing | **Reads whole bars: use it in a `kline` strategy.** On a `ticker` strategy its values are not the bar's and mean nothing. `atr()` registers `atr14`. There is no `pro.atr` |
| `stdDev` | Standard deviation | |
| `variance` | Variance | |
| `volatility` | Average of how often the price changes | `volatility(5)` registers `vol5` |
| `percentVolatility` | The same as a percentage | `percentVolatility(5)` registers `pvol5` |

## Distance and change

| Name | What it is | Notes |
|---|---|---|
| `distance` | Distance between two indicators in percent, `100 * (a - b) / b` | |
| `distanceMa` | Distance of the price from a moving average in percent, `100 * (price - ma) / ma` | Takes an SMA, an EMA or an MMA; `distanceMa("ema9")` registers `distema9` |
| `rateChange` | Change from the update before, as a fraction: 0.1 is 10% | One update back, no period; the `boolean` form chooses percent. Give it a name: by default it and `percentChange` both register `chgprice` for the price, and the second replaces the first |
| `percentChange` | Change from the update before, in percent: 10 is 10% | `rateChange` times 100. Give it a name, other than `chg`, which a ticker strategy registers |

## Counting and extremes

| Name | What it is | Notes |
|---|---|---|
| `gain` | The rise since the update before, or 0 | An amount, not a count: the count of consecutive rises is `getPeriodCount()` in Java. Give it a name: `gain("price")` registers `_gainprice`, which is hidden |
| `loss` | The fall since the update before, as a positive number, or 0 | An amount, not a count, like `gain`. `loss("price")` registers `_lossprice`, which is hidden |
| `max` | The highest value since the start | No window; `max("price")` registers `maxprice`. Over another indicator it includes its warm-up |
| `min` | The lowest value since the start | No window; `min("price")` registers `minprice`. Over another indicator it includes its warm-up: the minimum of an RSI is 0 |
| `sum` | The total since the start | No window; `sum("price")` registers `sumprice`. Over another indicator it includes its warm-up |

## Arithmetic, transformations and comparisons

| Name | What it is | Notes |
|---|---|---|
| `add` | Sum of two indicators | |
| `diff` | First indicator minus the second | |
| `mul` | An indicator scaled by a number, or two indicators multiplied | |
| `clamp` | An indicator held to a range | |
| `round` | An indicator rounded to a number of decimals | |
| `equal`, `notEqual`, `greatThan`, `greatOrEqual`, `lessThan`, `lessOrEqual` | An indicator compared with a number, as a boolean indicator | The five-argument forms choose between two indicators: `equal("selected", "signal", 1, "emaFast", "emaSlow")` gives `emaFast` while `signal` equals 1 and `emaSlow` otherwise |

## The pro indicators

A `uses` line brings them in: `uses pro.adx, pro.obv` for those two, `uses pro.*` for all twelve, and several `uses` lines
add up. A plan without them can register the strategy but not run it (`402`).

Eleven of the twelve read whole bars, not a single value, so they belong in a `kline` strategy: on a `ticker` strategy their
values are not the bar's and mean nothing. `kurtosis` is the exception, since it reads one value, the price of the group (the
close on a kline), so it means something on a `ticker` strategy and raises no notice.

### Trend

| Name | What it is | Notes |
|---|---|---|
| `adx` | Average directional index | The two directional lines register beside it |
| `superTrend` | SuperTrend line | An upper and a lower band register beside it |
| `ichimoku` | Ichimoku cloud | The Tenkan line is the main one, with the Kijun, Senkou A, Senkou B and Chikou lines beside it |

### Volume

| Name | What it is | Notes |
|---|---|---|
| `obv` | On-balance volume | |
| `vwap` | Volume-weighted average price | |
| `mfi` | Money flow index | |
| `cmf` | Chaikin money flow | |

### Statistics

| Name | What it is | Notes |
|---|---|---|
| `kurtosis` | Excess kurtosis of the price over a period | Reads one value, not a bar. The period is required |

### Volatility estimators

Each takes an annualization factor as its last, optional argument.

| Name | What it is | Notes |
|---|---|---|
| `parkinson` | Parkinson volatility | From the high and the low |
| `garmanKlass` | Garman-Klass volatility | From the open, high, low and close |
| `rogersSatchell` | Rogers-Satchell volatility | From the open, high, low and close |
| `yangZhang` | Yang-Zhang volatility | From the open, high, low and close |

## What QTScript cannot call

The builder methods that take a function (`predicate`, `fun`, `decorate`, `periodCount`) and `conditional` can only be written
in a Java strategy. So can the indicator classes that are not builder names: the Java engine has many more, pro ones included (a
z-score, Aroon, the Sharpe ratio and others), and the catalogue of the `qtsurfer-java-strategy` skill lists them with the
package each lives in.
